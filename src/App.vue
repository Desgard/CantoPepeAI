<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";
import useApiKey from "./AppState";
import ChatGPTAPI from "./ChatGPTAPI";

interface ChatModel {
  isGPT: boolean;
  message: string;
}

const { getApiKey, setApiKey } = useApiKey();
const showHelp = ref(false); // 展示帮助面板
const officialTips = ref(""); // 展示官方对话
const items = ref<ChatModel[]>([]);
const inputText = ref("");
const isLoading = ref(false);
const api = ref<ChatGPTAPI>(new ChatGPTAPI(""));

function renew() {
  const apiKey = getApiKey();
  if (apiKey != null && apiKey.length > 0) {
    api.value = new ChatGPTAPI(apiKey);
  }
}

renew();

function toggleHelp() {
  showHelp.value = !showHelp.value;
}

function newChat() {
  items.value = [];
  renew();
}

function addItem() {
  const input = inputText.value.trim();
  inputText.value = "";

  if (input.startsWith("key:")) {
    // 更新 key
    const key = input.replace("key:", "");

    setApiKey(key);
    renew();

    officialTips.value = "API Key Updated。";
    return;
  }

  if (input == "Log out") {
    officialTips.value = "API Key Clearn。";

    // 登出，清空 key
    setApiKey("");
    return;
  }

  const apiKey = getApiKey();
  if (apiKey == null || apiKey.length == 0) {
    officialTips.value =
      "Use the format「key:YOUR_API_KEY」to update API Key。";
    return;
  }

  // 有 key
  officialTips.value = "";

  if (isLoading.value) {
    // 加载中
    // 不应该出现这种情况，应该禁用
    inputText.value = input;
  } else {
    // 非加载中
    if (input) {
      isLoading.value = true;
      items.value.push({ isGPT: false, message: input });
      items.value.push({ isGPT: true, message: "💭 croak..." });

      // 请求 ChatGPT
      api.value
        .sendMessageStream(input)
        .then((reply) => {
          isLoading.value = false;
          items.value[items.value.length - 1].message = reply.trim();
        })
        .catch((error) => {
          isLoading.value = false;
          items.value[items.value.length - 1] = {
            isGPT: true,
            message: `[出错咯]${error}`,
          };
        });
    }
  }
}

function stop() {
  isLoading.value = false;
}

// 输入框占位符
const inputPlaceholder = computed(() => {
  if (isLoading.value) {
    return "💭 croak...";
  } else {
    return "Say...";
  }
});

// 键盘与输入框
const inputBottom = ref(0);

function handleKeyboardEvent(event: any) {
  if (event.type === "show") {
    // 键盘弹起时，将input组件显示在键盘上方
    inputBottom.value = event.height;
  } else if (event.type === "hide") {
    // 键盘收起时，将input组件重新放置在屏幕底部
    inputBottom.value = 0;
  }
}

onMounted(() => {
  // 监听键盘事件
  window.addEventListener("keyboardWillShow", handleKeyboardEvent);
  window.addEventListener("keyboardWillHide", handleKeyboardEvent);
});

onUnmounted(() => {
  // 取消监听键盘事件
  window.removeEventListener("keyboardWillShow", handleKeyboardEvent);
  window.removeEventListener("keyboardWillHide", handleKeyboardEvent);
});
</script>

<template>
  <div class="chatgkd">
    <!-- Header -->
    <div class="header">
      <span class="button" @click="toggleHelp">Help</span>
      <div class="title-container">
        <span class="title">Pepe AI 🐸</span>
      </div>
      <span class="button" @click="newChat">New Topic</span>
    </div>

    <!-- 帮助区域 -->
    <div class="help" v-if="showHelp">
      <div>
        Welcome use Pepe AI.
        <br />
      </div>
    </div>
    <div class="list" v-if="items.length > 0 || officialTips">
      <div class="tips" v-if="officialTips">{{ officialTips }}</div>
      <div v-for="(item, index) in items" :key="index" class="item">
        <span v-if="item.isGPT">️🐸：</span>
        <span v-else>👤：</span>
        <span v-if="item.isGPT">{{ item.message }}</span>
        <span v-else class="human-text">{{ item.message }}</span>
      </div>
    </div>

    <!-- Footer -->
    <div class="footer" :style="{ bottom: inputBottom + 'px' }">
      <input
        type="text"
        v-model="inputText"
        v-bind:placeholder="inputPlaceholder"
        v-bind:disabled="isLoading"
        @keydown.enter="addItem"
      />
      <button v-if="isLoading" class="footer-button" @click="stop">🛑</button>
      <button v-else class="footer-button" @click="addItem">Send</button>
    </div>
  </div>
</template>

<style scoped>
body {
  padding: 0px;
  margin: 0px;
}

.chatgkd {
  display: flex;
  flex-direction: column;
  height: 100%;

  padding-top: 50px;
  padding-left: 10px;
  padding-right: 10px;

  padding-bottom: 50px; /* 预留input组件的高度 */
}

.header {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  height: 50px;

  display: flex;
  align-items: center;
  padding: 10px;
  background-color: #e2e2e2;
}

.title {
  font-size: 24px;
  font-weight: 600;
}

.subtitle {
  font-size: 12px;
}

.title-container {
  flex: 1;
  display: flex;
  align-items: center;
  flex-direction: column;
}

.button {
  color: blue;
  border: 1px solid #000;
  padding: 5px;
  border-radius: 5px;
}

.footer {
  position: fixed;
  left: 0;
  right: 0;
  height: 50px;
}

.help {
  position: fixed;
  left: 0;
  right: 0;
  top: 70px;

  background-color: rgba(83, 97, 246, 0.95);
  color: #ffffff;
  padding: 10px;
  margin: 10px;
  border-radius: 10px;
}

.list {
  flex: 1;
  overflow-y: auto;

  margin-top: 30px;
  margin-bottom: 10px;
  padding: 10px 10px 0px 10px;
  border-radius: 10px;
  background-color: #efefef;
}

.tips {
  padding-bottom: 10px;
}

.item {
  padding-bottom: 10px;
}

.human-text {
  font-weight: 600;
}

input {
  width: 100%;
  height: 100%;
  border: none;
  outline: none;
  font-size: 14px;
  padding: 0 10px;

  background-color: #e2e2e2;
}

.footer-button {
  position: fixed;

  right: 0;
  bottom: 0;

  width: 80px;
  height: 50px;

  font-size: 16px;
  padding-left: 10px;
  padding-right: 10px;
}
</style>
