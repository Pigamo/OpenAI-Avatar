<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
const systemMessage = ref('This is a system message to set the context for the conversation.')
const chatBody = ref<HTMLDivElement | null>(null)
const messages = ref<Array<{ type: 'system' | 'user' | 'assistant'; text: string }>>([

])
const userInput = ref('')

onMounted(() => {
  if (chatBody.value)
    chatBody.value.scrollTop = chatBody.value.scrollHeight
})

watch(messages, () => {
  if (chatBody.value)
    chatBody.value.scrollTop = chatBody.value.scrollHeight
})

async function onSubmit() {
  if (userInput.value.trim() === '')
    return

  messages.value.push({ type: 'user', text: userInput.value.trim() })

  const response = await fetch('http://localhost:3001/api/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      messages: [
        { role: 'system', content: systemMessage.value },
        ...messages.value.map(message => ({
          role: message.type,
          content: message.text,
        })),
        { role: 'user', content: userInput.value.trim() },
      ],
    }),
  })

  const data = await response.json()
  const aiResponse = data.choices && data.choices[0] && data.choices[0].message.content.trim()

  if (aiResponse)
    messages.value.push({ type: 'assistant', text: aiResponse })
  userInput.value = ''
}
const formattedMessages = computed(() => {
  return messages.value.map((message) => {
    const formattedText = message.text.replace(/\n/g, '<br>')
    return { ...message, text: formattedText }
  })
})
</script>

<template>
  <div
    w-full flex items-center justify-center lg:p10
    bg-gray-100 dark:bg-gray-700
  >
    <div
      w-full flex flex-wrap items-stretch bg-gray-200
      dark:bg-gray-700 rounded-2xl overflow-hidden shadow
      max-w-300
    >
      <div
        bg-gray-200 dark:bg-gray-700 class="lg:w-1/3"
        w-full order-last lg:order-first
      >
        <Face h-100 w-100 :messages="messages" />
        <div px4 py2>
          <label class="block mt2  text-left">System
            Message:</label>
          <textarea
            v-model="systemMessage" dark:bg-gray-800
            w-full mt2 min-h-23 shadow-inner
            type="textarea"
            placeholder="Type system message..." flex-grow
            border-0 focus:ring-0 focus:border-0 p-2
            rounded-lg
          />
        </div>
      </div>
      <div
        class="lg:w-2/3" flex flex-col flex-grow
        self-stretch content-between items-stretch
      >
        <div bg-gray-200 dark:bg-gray-700 p-4 text-2xl>
          <h2>Chat with AI</h2>
        </div>
        <div
          ref="chatBody"
          class="flex-grow self-stretch overflow-y-auto p-4 space-y-4 text-left"
          bg-light-50 dark:bg-gray-900 shadow-inner mx4
          min-h-40
        >
          <div
            v-for="(message, index) in formattedMessages"
            :key="index" class="flex"
            :class="message.type === 'user' ? 'justify-end' : ''"
          >
            <div
              :class="`rounded-lg p-4 ${message.type === 'user' ? 'bg-blue-500 text-white ' : 'bg-gray-300 text-black'}`"
              v-html="message.text"
            />
          </div>
        </div>
        <div bg-gray-200 dark:bg-gray-700 p-4 w-full>
          <form flex @submit.prevent="onSubmit">
            <input
              v-model="userInput" type="text"
              shadow-inner placeholder="Type your message..."
              flex-grow border-0 focus:ring-0 focus:border-0
              p-2 rounded-lg dark:bg-gray-800
            >
            <button
              type="submit" ml-4 px-4 py-2 rounded-lg
              bg-blue-500 text-white
            >
              Send
            </button>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>
