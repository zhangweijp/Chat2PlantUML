<template>
  <div id="app">
    <header>
      <h1>PlantUML AI Assistant</h1>
      <div class="tools">
        <button @click="copyCode" class="btn btn-primary">Copy Code</button>
        <button @click="exportJPEG" class="btn btn-secondary">Export JPEG</button>
      </div>
    </header>
    <main>
      <nav class="chat-nav">
        <button @click="newChat" class="btn btn-primary new-chat">New Chat</button>
        <ul class="chat-list">
          <li v-for="(chat, index) in chats" :key="index" @click="switchChat(index)" :class="{ active: currentChatIndex === index }">
            Chat {{ index + 1 }}
          </li>
        </ul>
        <button class="btn btn-icon settings-btn">
          <span class="material-icons">settings</span>
        </button>
      </nav>
      <div v-if="currentChat" class="chat-area">
        <div class="chat-messages">
          <div v-for="(message, index) in currentChat.messages" :key="index" :class="['message', message.type]">
            <div class="message-content">
              <div v-if="message.type === 'ai'" class="ai-icon">
                <span class="material-icons" :style="{ color: getAIColor(message.aiType) }">smart_toy</span>
              </div>
              <div v-if="message.type === 'ai' && message.content.includes('@startuml')" class="code-block">
                <pre><code>{{ message.content }}</code></pre>
                <button @click="copyMessageCode(message.content)" class="btn btn-small">Copy</button>
              </div>
              <div v-else>{{ message.content }}</div>
            </div>
          </div>
        </div>
        <div class="chat-input">
          <input v-model="userInput" @keyup.enter="sendMessage" placeholder="Type your message...">
          <button @click="sendMessage" class="btn btn-send">
            <span class="material-icons">send</span>
          </button>
        </div>
      </div>
      <div v-if="currentChat" class="editor-preview">
        <div class="code-preview">
          <MonacoEditor
            v-model="currentChat.plantUMLCode"
            language="plantuml"
            @change="updateUMLDiagram"
          />
        </div>
        <div class="uml-diagram">
          <div class="uml-controls">
            <button @click="zoomIn" class="btn btn-icon"><span class="material-icons">zoom_in</span></button>
            <button @click="zoomOut" class="btn btn-icon"><span class="material-icons">zoom_out</span></button>
            <button @click="resetZoom" class="btn btn-icon"><span class="material-icons">center_focus_strong</span></button>
            <button @click="toggleFullscreen" class="btn btn-icon"><span class="material-icons">fullscreen</span></button>
          </div>
          <div class="uml-image-container" ref="umlImageContainer" :class="{ 'fullscreen': isFullscreen }">
            <img :src="currentChat.umlImageUrl" alt="UML Diagram" ref="umlImage" :style="{ transform: `scale(${zoomLevel})` }" @wheel="handleWheel" />
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import MonacoEditor from './components/MonacoEditor.vue'
import plantumlEncoder from 'plantuml-encoder'
import axios from 'axios'

export default {
  components: {
    MonacoEditor
  },
  setup() {
    const userInput = ref('')
    const chats = ref([])
    const currentChatIndex = ref(0)
    const zoomLevel = ref(1)
    const isFullscreen = ref(false)
    const umlImageContainer = ref(null)

    const currentChat = computed(() => chats.value[currentChatIndex.value] || null)

    const createNewChat = () => ({
      messages: [],
      plantUMLCode: '@startuml\nclass User\nclass Order\nUser -- Order\n@enduml',
      umlImageUrl: ''
    })

    const sendMessage = async () => {
      if (!currentChat.value || userInput.value.trim() === '') return
      
      currentChat.value.messages.push({ type: 'user', content: userInput.value })
      
      // TODO: Implement AI interaction
      console.log('Sending message:', userInput.value)
      
      // Simulating AI response
      currentChat.value.messages.push({ type: 'ai', content: 'AI response to: ' + userInput.value, aiType: 'chatgpt' })
      
      userInput.value = ''
    }

    const updateUMLDiagram = () => {
      if (!currentChat.value) return
      const encoded = plantumlEncoder.encode(currentChat.value.plantUMLCode)
      currentChat.value.umlImageUrl = `http://www.plantuml.com/plantuml/img/${encoded}`
    }

    const copyCode = () => {
      if (!currentChat.value) return
      navigator.clipboard.writeText(currentChat.value.plantUMLCode)
        .then(() => alert('Code copied to clipboard!'))
        .catch(err => console.error('Failed to copy code: ', err))
    }

    const copyMessageCode = (code) => {
      navigator.clipboard.writeText(code)
        .then(() => alert('Code copied to clipboard!'))
        .catch(err => console.error('Failed to copy code: ', err))
    }

    const exportJPEG = () => {
      const umlImage = document.querySelector('.uml-diagram img')
      if (umlImage) {
        const canvas = document.createElement('canvas')
        canvas.width = umlImage.naturalWidth
        canvas.height = umlImage.naturalHeight
        canvas.getContext('2d').drawImage(umlImage, 0, 0)
        
        canvas.toBlob((blob) => {
          const url = URL.createObjectURL(blob)
          const a = document.createElement('a')
          a.href = url
          a.download = 'uml-diagram.jpg'
          a.click()
          URL.revokeObjectURL(url)
        }, 'image/jpeg')
      }
    }

    const newChat = () => {
      chats.value.push(createNewChat())
      currentChatIndex.value = chats.value.length - 1
      updateUMLDiagram()
    }

    const switchChat = (index) => {
      currentChatIndex.value = index
    }

    const startVoiceInput = () => {
      // TODO: Implement voice input functionality
      alert('Voice input feature is not implemented yet.')
    }

    const zoomIn = () => {
      zoomLevel.value = Math.min(zoomLevel.value + 0.1, 3)
    }

    const zoomOut = () => {
      zoomLevel.value = Math.max(zoomLevel.value - 0.1, 0.5)
    }

    const resetZoom = () => {
      zoomLevel.value = 1
    }

    const handleWheel = (event) => {
      if (event.ctrlKey) {
        event.preventDefault()
        const delta = event.deltaY > 0 ? -0.1 : 0.1
        zoomLevel.value = Math.max(0.5, Math.min(zoomLevel.value + delta, 3))
      }
    }

    const toggleFullscreen = () => {
      if (!document.fullscreenElement) {
        umlImageContainer.value.requestFullscreen().catch(err => {
          console.error(`Error attempting to enable full-screen mode: ${err.message}`)
        })
      } else {
        document.exitFullscreen()
      }
    }

    const getAIColor = (aiType) => {
      switch (aiType) {
        case 'chatgpt':
          return '#10a37f'
        case 'bing':
          return '#0078d4'
        default:
          return '#8e44ad'
      }
    }

    onMounted(() => {
      newChat() // Create the first chat
      document.addEventListener('fullscreenchange', () => {
        isFullscreen.value = !!document.fullscreenElement
      })
    })

    watch(currentChat, () => {
      if (currentChat.value) {
        updateUMLDiagram()
      }
    }, { deep: true })

    return {
      userInput,
      chats,
      currentChatIndex,
      currentChat,
      sendMessage,
      updateUMLDiagram,
      copyCode,
      copyMessageCode,
      exportJPEG,
      newChat,
      switchChat,
      startVoiceInput,
      zoomLevel,
      zoomIn,
      zoomOut,
      resetZoom,
      handleWheel,
      toggleFullscreen,
      isFullscreen,
      umlImageContainer,
      getAIColor
    }
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/icon?family=Material+Icons');
@import url('https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500&display=swap');

:root {
  --primary-color: #8e44ad;
  --secondary-color: #3498db;
  --background-color: #f3e5f5;
  --text-color: #333333;
  --light-gray: #e0e0e0;
  --white: #ffffff;
  --card-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  --send-button-color: #3498db;
}

body {
  margin: 0;
  padding: 0;
  background-color: var(--background-color);
}

#app {
  font-family: 'Roboto', sans-serif;
  display: flex;
  flex-direction: column;
  height: 100vh;
  color: var(--text-color);
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  box-shadow: var(--card-shadow);
}

main {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.chat-nav {
  width: 200px;
  padding: 1rem;
  background-color: var(--white);
  overflow-y: auto;
  box-shadow: var(--card-shadow);
  display: flex;
  flex-direction: column;
}

.chat-list {
  list-style-type: none;
  padding: 0;
  flex-grow: 1;
}

.chat-list li {
  cursor: pointer;
  padding: 0.75rem;
  margin-bottom: 0.75rem;
  background-color: var(--light-gray);
  border-radius: 12px;
  color: var(--text-color);
  transition: all 0.3s ease;
}

.chat-list li:hover {
  background-color: var(--secondary-color);
  color: var(--white);
  transform: translateY(-2px);
}

.chat-list li.active {
  background-color: var(--primary-color);
  color: var(--white);
  transform: translateY(-2px);
}

.chat-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 1rem;
  overflow: hidden;
  background-color: var(--white);
  margin: 1rem;
  border-radius: 16px;
  box-shadow: var(--card-shadow);
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  padding: 1rem;
}

.message {
  max-width: 80%;
  margin-bottom: 1rem;
  border-radius: 16px;
  word-wrap: break-word;
  box-shadow: var(--card-shadow);
}

.message-content {
  display: flex;
  align-items: flex-start;
  padding: 0.75rem 1rem;
}

.ai-icon {
  margin-right: 0.5rem;
}

.message.user {
  align-self: flex-end;
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
}

.message.ai {
  align-self: flex-start;
  background-color: var(--light-gray);
  color: var(--text-color);
}

.code-block {
  background-color: var(--white);
  border-radius: 12px;
  padding: 0.75rem;
  margin-top: 0.75rem;
  position: relative;
  color: var(--text-color);
  box-shadow: var(--card-shadow);
}

.code-block pre {
  margin: 0;
  white-space: pre-wrap;
}

.code-block .btn-small {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
}

.chat-input {
  display: flex;
  margin-top: 1rem;
  position: relative;
}

.chat-input input {
  flex: 1;
  padding: 0.75rem;
  padding-right: 3rem;
  border: 1px solid var(--light-gray);
  border-radius: 24px;
  font-size: 1rem;
}

.btn-send {
  position: absolute;
  right: 0.5rem;
  bottom: 0.5rem;
  background-color: var(--send-button-color);
  color: var(--white);
  border-radius: 50%;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s ease;
}

.btn-send:hover {
  transform: scale(1.1);
}

.editor-preview {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: var(--white);
  margin: 1rem;
  border-radius: 16px;
  box-shadow: var(--card-shadow);
}

.code-preview,
.uml-diagram {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.uml-controls {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 10;
  display: flex;
  gap: 5px;
}

.uml-image-container {
  width: 100%;
  height: 100%;
  overflow: auto;
  display: flex;
  justify-content: center;
  align-items: center;
}

.uml-image-container.fullscreen {
  background-color: var(--white);
}

.uml-image-container img {
  max-width: 100%;
  height: auto;
  transition: transform 0.2s ease-out;
}

.btn {
  padding: 0.75rem 1rem;
  border: none;
  border-radius: 24px;
  cursor: pointer;
  margin-left: 0.5rem;
  transition: all 0.3s ease;
  font-weight: 500;
}

.btn-icon {
  width: 36px;
  height: 36px;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
}

.btn-small {
  padding: 0.5rem 0.75rem;
  font-size: 0.9rem;
}

.btn-primary {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.btn-secondary {
  background-color: var(--light-gray);
  color: var(--text-color);
}

.btn-secondary:hover {
  background-color: var(--secondary-color);
  color: var(--white);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.new-chat {
  margin-bottom: 1rem;
  width: 100%;
}

.settings-btn {
  margin-top: auto;
  align-self: center;
  background-color: var(--light-gray);
  color: var(--text-color);
}

.material-icons {
  font-size: 1.2rem;
  vertical-align: middle;
}
</style>