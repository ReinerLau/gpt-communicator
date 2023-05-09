<template>
  <div class="h-screen flex flex-col bg-[#343541]">
    <div
      v-show="recordStatus"
      class="flex flex-col justify-center items-center fixed top-1/2 left-1/2 translate-x-[-50%] bg-slate-500 text-white p-5 rounded opacity-50"
    >
      <img class="h-8" src="@/assets/microphone.svg" />
      <span>语音识别中...</span>
    </div>
    <div class="flex-1 overflow-auto p-5">
      <template v-for="item in messages" :key="item.content">
        <div v-if="item.role === 'user'" class="flex justify-end items-center mb-4">
          <div
            @click="() => handlePlay(item.content)"
            class="max-w-full p-2 rounded bg-slate-600 text-white break-all shadow-md cursor-pointer"
          >
            {{ item.content }}
          </div>
          <div class="text-2xl ml-3">🙍‍♂️</div>
        </div>
        <div v-if="item.role === 'assistant'" class="flex justify-start items-center mb-4">
          <div class="text-2xl mr-3">‍️🤖</div>
          <div
            @click="() => handlePlay(item.content)"
            class="max-w-full p-2 rounded bg-slate-600 text-white break-all shadow-md cursor-pointer"
          >
            {{ item.content }}
          </div>
        </div>
      </template>
    </div>
    <div class="relative h-24 flex justify-center">
      <div class="bg-[#40414f] shadow rounded overflow-hidden w-1/2 h-1/2 flex p-3">
        <input
          @keydown.enter="sendMessage"
          v-model="question"
          class="flex-1 bg-[#40414f] outline-none text-white"
        />
        <img @click="handleRecord" class="h-full cursor-pointer" src="@/assets/microphone.svg" />
      </div>
      <div
        class="bg-amber-200 h-1/2 rounded px-2 flex items-center ml-2 cursor-pointer select-none hover:bg-amber-100 active:bg-amber-200"
        @click="sendMessage"
      >
        SEND
      </div>
      <div
        class="bg-red-500 text-white h-1/2 rounded px-2 flex items-center ml-2 cursor-pointer select-none hover:bg-red-400 active:bg-red-500"
        @click="clearMessage"
      >
        CLEAR
      </div>
      <div
        class="text-xl text-white h-1/2 rounded px-2 flex items-center ml-2 cursor-pointer select-none absolute bottom-0 right-0"
        @click="handleSetting"
      >
        ⚙️
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ChatCompletionRequestMessageRoleEnum,
  Configuration,
  OpenAIApi,
  type ChatCompletionRequestMessage
} from 'openai'
import { ref } from 'vue'

const synth = window.speechSynthesis
const SpeechRecognition =
  (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition
const recognition = new SpeechRecognition()
let voice
const question = ref('')
const recordStatus = ref(false)
const voices = ref<any>([])

function initSpeech() {
  voices.value = synth.getVoices()
  voice = voices.value[0]
  synth.onvoiceschanged = () => {
    voices.value = synth.getVoices()
    voice = voices.value[0]
  }
}

function handlePlay(message) {
  if (synth.speaking) synth.cancel()
  const utter = new SpeechSynthesisUtterance(message)
  utter.voice = voice
  synth.speak(utter)
}

function initRecognition() {
  recognition.continuous = true
  recognition.interimResults = true
  recognition.maxAlternatives = 1
  recognition.onresult = (e) => {
    const text = e.results[0][0].transcript
    question.value = text
  }
}

function handleRecord() {
  if (recordStatus.value) {
    handleRecordEnd()
  } else {
    handleRecordStart()
  }
}

function handleRecordStart() {
  synth.cancel()
  recordStatus.value = true
  recognition.start()
}

function handleRecordEnd() {
  recordStatus.value = false
  recognition.stop()
}

const messages = ref<ChatCompletionRequestMessage[]>([])

async function getAnswer() {
  const configuration = new Configuration({
    apiKey: import.meta.env.VITE_APP_OPEN_AI_KEY,
    basePath: import.meta.env.VITE_APP_PROXY
  })
  const openai = new OpenAIApi(configuration)
  const answer = {
    role: ChatCompletionRequestMessageRoleEnum.Assistant,
    content: 'Thinking...'
  }
  messages.value.push(answer)
  try {
    const completion = await openai.createChatCompletion({
      model: 'gpt-3.5-turbo',
      messages: messages.value.filter((item) => item.content !== 'Thinking...')
    })
    messages.value.slice(-1)[0].content = completion.data.choices[0].message?.content as string
    handlePlay(messages.value.slice(-1)[0].content)
  } catch (error: any) {
    messages.value.slice(-1)[0].content = error.message
  }
}

function sendMessage() {
  if (!question.value) return
  recordStatus.value = false
  messages.value.push({
    role: ChatCompletionRequestMessageRoleEnum.User,
    content: question.value
  })
  question.value = ''
  getAnswer()
}

function clearMessage() {
  messages.value = []
}

const settingVisible = ref(false)

function handleSetting() {
  settingVisible.value = true
}

initSpeech()
initRecognition()
</script>

<style>
::-webkit-scrollbar {
  display: none;
}
</style>
