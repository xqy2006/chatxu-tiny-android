<template>
  <div class="flex flex-col h-screen">
    <div class="flex flex-nowrap fixed w-full items-baseline top-0 px-6 py-4 bg-gray-100">
      <div class="text-lg font-bold">ChatXu-tiny</div>
      <div class="ml-4 text-sm text-gray-500">
        您的智能AI助手&nbsp;&nbsp;&nbsp;
      </div>
      <button class="btn" style="float: right;" :disabled="isTalking" @click="rem()">
          撤回
        </button>
    </div>

    <div class="flex-1 mx-2 mt-20 mb-2" ref="chatListDom">
      <div class="group flex flex-col px-4 py-3 hover:bg-slate-100 rounded-lg"
        v-for="item of messageList.filter((v) => v.role !== 'system')">
        <div class="flex justify-between items-center mb-2">
          <div class="font-bold">{{ roleAlias[item.role] }}：</div>
          <Copy class="invisible group-hover:visible" :content="item.content" />
        </div>
        <div>
          <div class="prose text-sm text-slate-600 leading-relaxed" v-if="item.content"
            v-html="md.render(item.content)"></div>
          <Loding v-else />
        </div>
      </div>
    </div>

    <div class="sticky bottom-0 w-full p-6 pb-8 bg-gray-100">
      <div class="flex">
        <input class="input" type="text" :placeholder="isTalking ? '请稍候' : '请输入'"
          v-model="messageContent" @keydown.enter="isTalking || sendOrSave()" />
        <button class="btn" :disabled="isTalking" @click="sendOrSave()">
          {{ isTalking ? "加载中" : "发送" }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ChatMessage } from "@/types";
import { ref, watch, nextTick, onMounted } from "vue";
import { chat } from "@/libs/gpt";
import cryptoJS from "crypto-js";
import Loding from "@/components/Loding.vue";
import Copy from "@/components/Copy.vue";
import { md } from "@/libs/markdown";
import { Wllama } from "@wllama/wllama";
let apiKey = "";
let isConfig = ref(false);
let isTalking = ref(false);
let messageContent = ref("");
const chatListDom = ref<HTMLDivElement>();
const decoder = new TextDecoder("utf-8");
const roleAlias = { user: "用户", assistant: "ChatXu", system: "System" };
const messageList = ref<ChatMessage[]>([
]);
var wllama = undefined
var bin = undefined
isTalking.value = true
new Promise(async () => {
  wllama = await new Wllama({
    'single-thread/wllama.js': './esm/single-thread/wllama.js',
    'single-thread/wllama.wasm': './esm/single-thread/wllama.wasm',
    'multi-thread/wllama.js': './esm/multi-thread/wllama.js',
    'multi-thread/wllama.wasm': './esm/multi-thread/wllama.wasm',
    'multi-thread/wllama.worker.mjs': './esm/multi-thread/wllama.worker.mjs',
  })
  bin = await fetchFile("model.gguf")
  await wllama.loadModel([new Blob([new Uint8Array(bin)], { type: 'application/octet-binary' })])
  isTalking.value = false
});
function fetchFile(url) {
  return new Promise((resolve, reject) => {
    var xhr = new XMLHttpRequest();
    xhr.open('GET', url, true);
    xhr.responseType = 'arraybuffer';

    xhr.onload = function () {
      if (xhr.status === 200) {
        resolve(xhr.response);
      } else {
        reject(new Error('Failed to fetch ' + url + ': ' + xhr.status));
      }
    };

    xhr.onerror = function () {
      reject(new Error('Network error occurred while fetching ' + url));
    };

    xhr.send();
  });
}

onMounted(async () => {
  if (getAPIKey()) {
    switchConfigStatus();

  }

});

const sendChatMessage = async (content: string = messageContent.value) => {
  try {
    isTalking.value = true;
    messageList.value.push({ role: "user", content });
    var resultstr = '';
    console.log(messageList.value);
    for (var i of messageList.value) {
      if (i.role == "user") {
        resultstr += "<|im_start|>user\n" + i.content + "\n<|im_end|>\n<|im_start|>assistant\n";
      }
      if (i.role == "assistant") {
        resultstr += i.content + "\n<|im_end|>\n";
      }
    }
    console.log(resultstr);
    clearMessageContent();
    messageList.value.push({ role: "assistant", content: "" });

    await wllama.createCompletion(resultstr, {
      nPredict: parseInt(2048),
      sampling: {
        temp: 0.5,
        top_k: 40,
        top_p: 0.9,
      },
      onNewToken: (token, piece, currentText) => {
        messageList.value[messageList.value.length - 1].content = currentText;
      },
    });

  } finally {
    isTalking.value = false;
  }
};

const readStream = async (
  reader: ReadableStreamDefaultReader<Uint8Array>,
  status: number
) => {
  let partialLine = "";

  while (true) {
    // eslint-disable-next-line no-await-in-loop
    const { value, done } = await reader.read();
    if (done) break;

    const decodedText = decoder.decode(value, { stream: true });

    if (status !== 200) {
      const json = JSON.parse(decodedText); // start with "data: "
      const content = json.error.message ?? decodedText;
      appendLastMessageContent(content);
      return;
    }

    const chunk = partialLine + decodedText;
    const newLines = chunk.split(/\r?\n/);

    partialLine = newLines.pop() ?? "";

    for (const line of newLines) {
      if (line.length === 0) continue; // ignore empty message
      if (line.startsWith(":")) continue; // ignore sse comment message
      if (line === "data: [DONE]") return; //

      const json = JSON.parse(line.substring(6)); // start with "data: "
      const content =
        status === 200
          ? json.choices[0].delta.content ?? ""
          : json.error.message;
      appendLastMessageContent(content);
    }
  }
};

const appendLastMessageContent = (content: string) =>
  (messageList.value[messageList.value.length - 1].content += content);

const sendOrSave = () => {
  if (!messageContent.value.length) return;
  if (isConfig.value) {
    if (saveAPIKey(messageContent.value.trim())) {
      switchConfigStatus();
    }
    clearMessageContent();
  } else {
    sendChatMessage();
  }
};
const rem = () => { 
  messageList.value.pop();
  messageList.value.pop();

};
const clickConfig = () => {
  if (!isConfig.value) {
    messageContent.value = getAPIKey();
  } else {
    clearMessageContent();
  }
  switchConfigStatus();
};

const getSecretKey = () => "lianginx";

const saveAPIKey = (apiKey: string) => {
  if (apiKey.slice(0, 3) !== "sk-" || apiKey.length !== 51) {
    alert("API Key 错误，请检查后重新输入！");
    return false;
  }
  const aesAPIKey = cryptoJS.AES.encrypt(apiKey, getSecretKey()).toString();
  localStorage.setItem("apiKey", aesAPIKey);
  return true;
};

const getAPIKey = () => {
  if (apiKey) return apiKey;
  const aesAPIKey = localStorage.getItem("apiKey") ?? "";
  apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
    cryptoJS.enc.Utf8
  );
  return apiKey;
};

const switchConfigStatus = () => (isConfig.value = !isConfig.value);

const clearMessageContent = () => (messageContent.value = "");

const scrollToBottom = () => {
  if (!chatListDom.value) return;
  scrollTo(0, chatListDom.value.scrollHeight);
};

watch(messageList.value, () => nextTick(() => scrollToBottom()));
</script>

<style scoped>
pre {
  font-family: -apple-system, "Noto Sans", "Helvetica Neue", Helvetica,
    "Nimbus Sans L", Arial, "Liberation Sans", "PingFang SC", "Hiragino Sans GB",
    "Noto Sans CJK SC", "Source Han Sans SC", "Source Han Sans CN",
    "Microsoft YaHei", "Wenquanyi Micro Hei", "WenQuanYi Zen Hei", "ST Heiti",
    SimHei, "WenQuanYi Zen Hei Sharp", sans-serif;
}
</style>
