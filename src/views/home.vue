<template>
    <div class="flex flex-col h-screen">
        <div
            class="flex flex-nowrap fixed w-full items-baseline top-0 px-6 py-4 bg-gray-100"
        >
            <div class="text-2xl font-bold">ChatGPT</div>
            <div class="ml-4 text-sm text-gray-500">Open AI Chatbot</div>
            <div
                class="ml-auto px-3 py-2 text-sm cursor-pointer hover:bg-white rounded-md"
                @click="clickConfig()"
            >
                Setup
            </div>
        </div>

        <div class="flex-1 mx-2 mt-20 mb-2" ref="chatListDom">
            <div
                class="group flex flex-col px-4 py-3 hover:bg-slate-100 rounded-lg"
                v-for="item of messageList.filter((v) => v.role !== 'system')"
            >
                <div class="flex justify-between items-center mb-2">
                    <div class="font-bold">{{ roleAlias[item.role] }}：</div>
                    <Copy
                        class="invisible group-hover:visible"
                        :content="item.content"
                    />
                </div>
                <div>
                    <div
                        class="prose text-sm text-slate-600 leading-relaxed"
                        v-if="item.content"
                        v-html="md.render(item.content)"
                    ></div>
                    <Loding v-else />
                </div>
            </div>
        </div>

        <div class="sticky bottom-0 w-full p-6 pb-8 bg-gray-100">
            <div class="-mt-2 mb-2 text-sm text-gray-500" v-if="isConfig">
                Enter your OpenAI API Key：
            </div>
            <div class="flex">
                <input
                    class="input"
                    :type="isConfig ? 'password' : 'text'"
                    :placeholder="
                        isConfig ? 'sk-xxxxxxxxxx' : 'Please enter your message'
                    "
                    v-model="messageContent"
                    @keydown.enter="isTalking || sendOrSave()"
                />
                <button class="btn" :disabled="isTalking" @click="sendOrSave()">
                    {{ isConfig ? "Config" : "Send" }}
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

let apiKey = "";
let isConfig = ref(true);
let isTalking = ref(false);
let messageContent = ref("");
const chatListDom = ref<HTMLDivElement>();
const decoder = new TextDecoder("utf-8");
const roleAlias = { user: "ME", assistant: "ChatGPT", system: "System" };
const messageList = ref<ChatMessage[]>([
    {
        role: "system",
        content:
            "you are a taciturn assistant, you are a specialist in the field of passive online income, in particular, you are well versed in creating faceless Youtube channels, marketing for such channels, how to monetize, create content for social networks, YouTube videos, use chatGPT, google spreadsheets and Canva for content design, you are very good at coming up with capacious and short quotes, headlines, always cheerful and ready to go. If the user asks something on a different topic, you do not refuse, but without further ado you answer the user's questions. Default language is Russian.",
    },
    {
        role: "assistant",
        content:
            "Приветствую! Я - эксперт в области создания пассивного дохода через интернет, специализируясь на анонимных YouTube-каналах и маркетинге для них.\n\nМогу помочь вам с созданием контента, его оформлением через Canva и Google таблицы, а также в вопросах использования ChatGPT.\n\nГотов ответить и на другие ваши вопросы. Общаюсь на русском языке.".trim(),
    },
]);

onMounted(() => {
    if (getAPIKey()) {
        switchConfigStatus();
    }
});

const sendChatMessage = async (content: string = messageContent.value) => {
    try {
        isTalking.value = true;
        if (messageList.value.length === 2) {
            messageList.value.pop();
        }
        messageList.value.push({ role: "user", content });
        clearMessageContent();
        messageList.value.push({ role: "assistant", content: "" });

        const { body, status } = await chat(messageList.value, getAPIKey());
        if (body) {
            const reader = body.getReader();
            await readStream(reader, status);
        }
    } catch (error: any) {
        appendLastMessageContent(error);
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

const clickConfig = () => {
    if (!isConfig.value) {
        messageContent.value = getAPIKey();
    } else {
        clearMessageContent();
    }
    switchConfigStatus();
};

const getSecretKey = () => "mainfish";

const saveAPIKey = (apiKey: string) => {
    if (apiKey.slice(0, 3) !== "sk-" || apiKey.length !== 51) {
        alert("API Key error, please check and re-enter!");
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
        "Nimbus Sans L", Arial, "Liberation Sans", "PingFang SC",
        "Hiragino Sans GB", "Noto Sans CJK SC", "Source Han Sans SC",
        "Source Han Sans CN", "Microsoft YaHei", "Wenquanyi Micro Hei",
        "WenQuanYi Zen Hei", "ST Heiti", SimHei, "WenQuanYi Zen Hei Sharp",
        "Arial", "Roboto", "Segoe UI", "Tahoma", "Verdana", "Tahoma", sans-serif,
        "Arial", "Roboto", "Noto Sans", "Arial Unicode MS", "Helvetica",
        "Verdana", "Tahoma", "Calibri", "Trebuchet MS", "Lucida Sans Unicode",
        "Arial Narrow", "Georgia", "Times New Roman", serif;
}
</style>
