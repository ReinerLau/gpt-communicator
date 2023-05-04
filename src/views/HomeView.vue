<template>
  <div class="px-5 select-none bg-[#eeeeee] h-screen">
    <div
      v-show="recordStatus"
      class="flex flex-col justify-center items-center fixed top-1/2 left-1/2 translate-x-[-50%] bg-slate-500 text-white p-5 rounded opacity-50"
    >
      <img class="h-8" src="@/assets/microphone.svg" />
      <span>语音识别中...</span>
    </div>
    <div
      @click="handlePlay"
      @touchstart="handleRecordStart"
      @touchend="handleRecordEnd"
      class="fixed bottom-5 right-5 bg-green-500 p-2 rounded-full"
    >
      <img class="h-8 pointer-events-none" src="@/assets/microphone.svg" />
    </div>
    <div class="py-3 text-center text-green-500 text-xl font-bold">智能移动机器人中山研究院</div>
    <div>
      <select @change="handleVoiceChange" ref="voiceOptions" class="w-full" id="voiceOptions">
        <option v-for="voice in voices" :key="voice.name" :value="voice.name">
          {{ voice.name }}
        </option>
      </select>
    </div>
    <div class="flex justify-end my-5">
      <div class="max-w-xs p-2 rounded bg-green-500 text-white break-all shadow-md">
        {{ question }}
      </div>
    </div>
    <div class="flex justify-start my-5">
      <div @touchstart="handlePlay" class="max-w-xs p-2 rounded bg-white break-all shadow-md">
        <Markdown :source="answer"></Markdown>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ChatCompletionRequestMessage } from 'openai'
import { Configuration, OpenAIApi } from 'openai'
import { ref } from 'vue'
import Markdown from 'vue3-markdown-it'

const synth = window.speechSynthesis
const SpeechRecognition =
  (window as any).SpeechRecognition || (window as any).webkitSpeechRecognition
const recognition = new SpeechRecognition()
let voice
const question = ref('')
const answer = ref('')
const recordStatus = ref(false)
const voices = ref<any>([])

function handleVoiceChange(e) {
  voice = voices.value[e.target.selectedIndex]
}

function initSpeech() {
  voices.value = synth.getVoices().filter((item) => item.lang.includes('zh'))
  voice = voices.value[0]
  synth.onvoiceschanged = () => {
    voices.value = synth.getVoices().filter((item) => item.lang.includes('zh'))
    voice = voices.value[0]
  }
}

function handlePlay() {
  if (synth.speaking) {
    synth.cancel()
    return
  }
  const utter = new SpeechSynthesisUtterance(answer.value)
  utter.voice = voice
  synth.speak(utter)
}

function initRecognition() {
  recognition.continuous = true
  recognition.lang = 'zh-CN'
  recognition.interimResults = true
  recognition.maxAlternatives = 1
  recognition.onresult = (e) => {
    const text = e.results[0][0].transcript
    question.value = text
  }

  recognition.onerror = (e) => {
    console.log(e.error)
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
  if (!question.value) return
  if (messages.length > 1 && question.value === messages[messages.length - 2].content) return
  messages.push({
    role: 'user',
    content: question.value
  })
  getAnswer()
}

const messages: ChatCompletionRequestMessage[] = [
  // { role: 'system', content: 'You are a helpful assistant.' }
]

async function getAnswer() {
  answer.value = '思考中......'
  const configuration = new Configuration({
    apiKey: import.meta.env.VITE_APP_OPEN_AI_KEY,
    // basePath: 'https://closeai.deno.dev/v1'
    basePath: 'https://api.openai-proxy.com/v1'
  })
  const openai = new OpenAIApi(configuration)
  try {
    const completion = await openai.createChatCompletion({
      model: 'gpt-3.5-turbo',
      messages
    })
    answer.value = completion.data.choices[0].message?.content as string
    messages.push({
      role: 'assistant',
      content: answer.value
    })
    handlePlay()
  } catch (error: any) {
    answer.value = error.message
  }
}

initSpeech()
initRecognition()
</script>
