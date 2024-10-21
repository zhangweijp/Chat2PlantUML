<template>
  <div id="app">
    <header>
      <div class="header-left">
        <button @click="toggleChatNav" class="btn btn-icon hamburger-menu">
          <span class="material-icons">menu</span>
        </button>
        <h1>PlantUML AI Assistant</h1>
      </div>
      <!-- 移除 header-right 部分 -->
    </header>
    <main>
      <div class="content-wrapper">
        <nav class="chat-nav" :class="{ 'chat-nav-hidden': !isChatNavVisible }">
          <div class="chat-nav-header">
            <h2>Chats</h2>
            <button @click="newChat" class="btn btn-icon new-chat" title="New Chat">
              <span class="material-icons">add</span>
            </button>
          </div>
          <ul class="chat-list">
            <li v-for="(chat, index) in chats" :key="index" @click="switchChat(index)" :class="{ active: currentChatIndex === index }">
              <span class="material-icons">chat</span>
              Chat {{ index + 1 }}
            </li>
          </ul>
        </nav>
        <div class="main-content" :class="{ 'nav-open': isChatNavVisible }">
          <div v-if="currentChat" class="chat-area">
            <div class="chat-messages" ref="chatMessagesRef">
              <div v-for="(group, groupIndex) in groupedMessages" :key="groupIndex" class="message-group">
                <div v-for="(message, messageIndex) in group" :key="messageIndex" 
                     :class="['message', message.type]">
                  <div class="message-content">
                    <div v-if="message.type === 'ai' && messageIndex === 0" class="ai-icon">
                      <span class="material-icons">smart_toy</span>
                    </div>
                    <div>
                      <div v-if="!message.content.includes('@startuml')">{{ message.content }}</div>
                      <div v-else class="code-block">
                        <pre><code>{{ message.content }}</code></pre>
                        <button @click="copyMessageCode(message.content)" class="btn-small">Copy</button>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
            <div class="chat-input">
              <textarea
                v-model="userInput"
                @keydown.enter.exact.prevent="sendMessage"
                @keydown.shift.enter.exact="newline"
                placeholder="Type your message...
(Shift+Enter for new line)"
                rows="1"
                ref="inputArea"
              ></textarea>
              <button @click="sendMessage" class="btn btn-send">
                <span class="material-icons">send</span>
              </button>
            </div>
          </div>
          <div v-if="currentChat" class="editor-preview">
            <div class="editor-navbar">
              <div class="editor-switch">
                <button @click="toggleView" class="btn btn-icon" :title="isCodeView ? 'Switch to Diagram' : 'Switch to Code'">
                  <span class="material-icons">{{ isCodeView ? 'image' : 'code' }}</span>
                </button>
              </div>
              <div class="editor-controls">
                <template v-if="isCodeView">
                  <button @click="copyCode" class="btn btn-icon" title="Copy Code">
                    <span class="material-icons">content_copy</span>
                  </button>
                  <button @click="toggleTheme" class="btn btn-icon" title="Toggle Theme">
                    <span class="material-icons">{{ isDarkTheme ? 'light_mode' : 'dark_mode' }}</span>
                  </button>
                </template>
                <template v-else>
                  <button @click="exportJPEG" class="btn btn-icon" title="Download">
                    <span class="material-icons">download</span>
                  </button>
                  <button @click="zoomIn" class="btn btn-icon" title="Zoom In">
                    <span class="material-icons">zoom_in</span>
                  </button>
                  <button @click="zoomOut" class="btn btn-icon" title="Zoom Out">
                    <span class="material-icons">zoom_out</span>
                  </button>
                  <button @click="resetZoom" class="btn btn-icon" title="Reset Zoom">
                    <span class="material-icons">center_focus_strong</span>
                  </button>
                  <button @click="toggleFullscreen" class="btn btn-icon" title="Fullscreen">
                    <span class="material-icons">fullscreen</span>
                  </button>
                </template>
              </div>
            </div>
            <div class="editor-content">
              <div v-if="isCodeView" class="code-preview">
                <MonacoEditor
                  v-model="currentChat.plantUMLCode"
                  language="plantuml"
                  @change="updateUMLDiagram"
                  :theme="isDarkTheme ? 'vs-dark' : 'vs-light'"
                />
              </div>
              <div v-else class="uml-diagram">
                <div class="uml-image-container" ref="umlImageContainer" :class="{ 'fullscreen': isFullscreen }">
                  <img 
                    :src="currentChat.umlImageUrl" 
                    alt="UML Diagram" 
                    ref="umlImage" 
                    :style="{ 
                      transform: `scale(${zoomLevel})`,
                      transition: 'transform 0.2s ease-out'
                    }" 
                    @wheel="handleWheel"
                  />
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
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
    const isChatNavVisible = ref(true)
    const inputArea = ref(null)
    const isCodeView = ref(false) // 改为 false，示 diagram
    const isDarkTheme = ref(false)
    const chatMessagesRef = ref(null)

    const currentChat = computed(() => chats.value[currentChatIndex.value] || null)

    const createNewChat = () => {
      const newChat = {
        messages: [],
        plantUMLCode: '@startuml\nclass User\nclass Order\nUser -- Order\n@enduml',
        umlImageUrl: ''
      }
      const encoded = plantumlEncoder.encode(newChat.plantUMLCode)
      newChat.umlImageUrl = `http://www.plantuml.com/plantuml/img/${encoded}`
      return newChat
    }

    const scrollToBottom = () => {
      nextTick(() => {
        if (chatMessagesRef.value) {
          chatMessagesRef.value.scrollTop = chatMessagesRef.value.scrollHeight
        }
      })
    }

    const sendMessage = () => {
      if (userInput.value.trim() !== '') {
        if (!currentChat.value) return
        
        currentChat.value.messages.push({ type: 'user', content: userInput.value })
        
        // TODO: Implement AI interaction
        console.log('Sending message:', userInput.value)
        
        // Simulating AI response
        currentChat.value.messages.push({ type: 'ai', content: 'AI response to: ' + userInput.value, aiType: 'chatgpt' })
        
        userInput.value = ''
        nextTick(() => {
          resetTextareaHeight()
          scrollToBottom()
        })
      }
    }

    const newline = (event) => {
      event.preventDefault()
      userInput.value += '\n'
      nextTick(() => adjustTextareaHeight())
    }

    const adjustTextareaHeight = () => {
      if (inputArea.value) {
        inputArea.value.style.height = 'auto'
        inputArea.value.style.height = `${inputArea.value.scrollHeight}px`
      }
    }

    const resetTextareaHeight = () => {
      if (inputArea.value) {
        inputArea.value.style.height = '48px' // 重置为默认高度
      }
    }

    watch(userInput, (newValue) => {
      if (newValue.trim() === '') {
        resetTextareaHeight()
      } else {
        adjustTextareaHeight()
      }
    })

    onMounted(() => {
      resetTextareaHeight()
    })

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
    }

    const switchChat = (index) => {
      currentChatIndex.value = index
      if (!currentChat.value.umlImageUrl) {
        updateUMLDiagram()
      }
    }

    const startVoiceInput = () => {
      // TODO: Implement voice input functionality
      alert('Voice input feature is not implemented yet.')
    }

    const zoomIn = () => {
      zoomLevel.value = Math.min(zoomLevel.value * 1.2, 5) // 限制最大缩放级别
    }

    const zoomOut = () => {
      zoomLevel.value = Math.max(zoomLevel.value / 1.2, 0.1) // 限制最小缩放级别
    }

    const resetZoom = () => {
      zoomLevel.value = 1
    }

    const handleWheel = (event) => {
      if (event.ctrlKey) {
        event.preventDefault()
        const delta = event.deltaY > 0 ? 0.9 : 1.1
        zoomLevel.value = Math.max(0.1, Math.min(zoomLevel.value * delta, 5))
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

    const toggleChatNav = () => {
      isChatNavVisible.value = !isChatNavVisible.value
    }

    const groupedMessages = computed(() => {
      if (!currentChat.value) return [];
      return currentChat.value.messages.reduce((groups, message) => {
        if (groups.length === 0 || groups[groups.length - 1][0].type !== message.type) {
          groups.push([message]);
        } else {
          groups[groups.length - 1].push(message);
        }
        return groups;
      }, []);
    })

    const toggleView = () => {
      isCodeView.value = !isCodeView.value
    }

    const toggleTheme = () => {
      isDarkTheme.value = !isDarkTheme.value
      if (window.monaco) {
        window.monaco.editor.setTheme(isDarkTheme.value ? 'vs-dark' : 'custom-light')
      }
    }

    onMounted(() => {
      newChat() // Create the first chat
      document.addEventListener('fullscreenchange', () => {
        isFullscreen.value = !!document.fullscreenElement
      })
      scrollToBottom()
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
      getAIColor,
      isChatNavVisible,
      toggleChatNav,
      groupedMessages,
      inputArea,
      newline,
      isCodeView,
      toggleView,
      isDarkTheme,
      toggleTheme,
      adjustTextareaHeight,
      resetTextareaHeight,
      chatMessagesRef,
      scrollToBottom,
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
  --chat-nav-background: #E1BEE7; /* 改回淡紫色 */
  --text-color: #333333; /* 稍微加深文字颜色 */
  --light-gray: #e0e0e0;
  --white: #ffffff;
  --card-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  --send-button-color: #3498db;
  --chat-nav-header-background: #D1C4E9; /* 较深的淡紫色 */
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
  padding: 0.5rem 1rem;
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  box-shadow: var(--card-shadow);
}

.header-left, .header-right {
  display: flex;
  align-items: center;
}

h1 {
  font-size: 1.2rem;
  margin: 0 0 0 0.5rem;
}

.btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(255, 255, 255, 0.1);
  color: var(--white);
  margin-left: 0.5rem;
}

.btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
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

.btn-primary {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.1), rgba(255, 255, 255, 0.2));
}

.btn-primary:hover {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.2), rgba(255, 255, 255, 0.3));
}

.hamburger-menu {
  background: none;
  color: var(--white);
  font-size: 24px;
  padding: 0;
  margin-right: 0;
}

.material-icons {
  font-size: 1.2rem;
}

main {
  display: flex;
  flex: 1;
  overflow: auto; /* 改为 auto 以允许滚动 */
  padding: 1rem;
}

.content-wrapper {
  display: flex;
  flex: 1;
  position: relative;
  overflow: hidden;
}

.chat-nav {
  width: 250px;
  background-color: var(--white);
  overflow: hidden;
  box-shadow: var(--card-shadow);
  display: flex;
  flex-direction: column;
  transition: all 0.3s ease;
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  z-index: 1000;
  border-radius: 16px;
  margin-right: 1rem;
}

.chat-nav-hidden {
  transform: translateX(-110%);
}

.main-content {
  display: flex;
  flex: 1;
  transition: margin-left 0.3s ease-in-out;
  overflow: auto;
}

.main-content.nav-open {
  margin-left: 266px; /* 250px (nav width) + 16px (margin) */
}

.chat-nav-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background-color: var(--chat-nav-header-background); /* 保持头部的背景色不变 */
  border-bottom: 1px solid #B39DDB;
  height: 56px;
  box-sizing: border-box;
}

.chat-nav-header h2 {
  margin: 0;
  font-size: 1.2rem;
  color: var(--text-color);
}

.chat-list {
  list-style-type: none;
  padding: 15px; /* 增加内边距 */
  margin: 0;
  flex-grow: 1;
  overflow-y: auto;
  background-color: var(--white);
}

.chat-list li {
  cursor: pointer;
  padding: 0.8rem 1rem; /* 增加内边距 */
  margin-bottom: 0.8rem; /* 增加底部间距 */
  background-color: #F3E5F5;
  border-radius: 12px;
  color: var(--text-color);
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  font-size: 0.95rem; /* 稍微增大字体 */
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  width: 85%; /* 增加宽度 */
  max-width: 220px; /* 增加最大宽度 */
  margin-left: auto;
  margin-right: auto;
}

.chat-list li .material-icons {
  margin-right: 0.8rem;
  font-size: 1.2rem;
  color: var(--primary-color);
  flex-shrink: 0;
  transition: color 0.3s ease;
}

.chat-list li:hover .material-icons,
.chat-list li.active .material-icons {
  color: var(--white); /* 确保在悬停和激活状态下图标变为白色 */
}

.chat-list li:hover, .chat-list li.active {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.chat-area {
  width: 30%;
  min-width: 300px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: var(--white);
  margin-right: 1rem;
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

.message-group {
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
}

.message {
  max-width: 85%;
  border-radius: 18px;
  word-wrap: break-word;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  margin-bottom: 0.5rem;
  padding: 0.75rem 1rem;
  font-size: 0.95rem;
  line-height: 1.4;
}

.message.user {
  align-self: flex-end;
  background-color: #E3F2FD; /* 淡蓝色 */
  color: var(--text-color);
  border-bottom-right-radius: 4px;
}

.message.ai {
  align-self: flex-start;
  background-color: #F3E5F5; /* 淡紫色 */
  color: var(--text-color);
  border-bottom-left-radius: 4px;
}

.message-content {
  display: flex;
  align-items: center;
}

.ai-icon {
  flex-shrink: 0;
  margin-right: 0.75rem;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 28px;
  height: 28px;
  background-color: #9C27B0; /* 深紫色,与主题色调一致 */
  border-radius: 50%;
}

.ai-icon .material-icons {
  font-size: 16px;
  color: var(--white);
}

.code-block {
  background-color: #f8f8f8;
  border-radius: 8px;
  padding: 0.75rem;
  margin-top: 0.5rem;
  position: relative;
  color: var(--text-color);
  font-family: 'Courier New', Courier, monospace;
  font-size: 0.9rem;
  overflow-x: auto;
}

.code-block pre {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
}

.code-block .btn-small {
  position: absolute;
  top: 0.5rem;
  right: 0.5rem;
  padding: 0.25rem 0.5rem;
  font-size: 0.8rem;
  background-color: var(--primary-color);
  color: var(--white);
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.code-block .btn-small:hover {
  background-color: var(--secondary-color);
}

.chat-input {
  display: flex;
  margin: 1rem;
  position: relative;
  align-items: center; /* 改为 center */
}

.chat-input textarea {
  flex: 1;
  padding: 0.75rem 1rem;
  padding-right: 3rem;
  border: 1px solid var(--light-gray);
  border-radius: 24px;
  font-size: 0.95rem;
  font-family: 'Roboto', sans-serif;
  resize: none;
  overflow-y: hidden;
  min-height: 48px;
  max-height: 150px;
  line-height: 1.4;
  transition: all 0.3s ease;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.05);
  background-color: #f9f9f9;
}

.chat-input textarea:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 2px 8px rgba(142, 68, 173, 0.2);
  background-color: var(--white);
}

.chat-input textarea::placeholder {
  color: #aaa;
  font-style: italic;
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  left: 1rem;
  right: 3rem;
  white-space: pre-wrap; /* 改为 pre-wrap 允许换行 */
  overflow: hidden;
  text-overflow: ellipsis;
  line-height: 1.2; /* 添加行高 */
  text-align: left; /* 确保文本左对齐 */
}

.btn-send {
  position: absolute;
  right: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  background-color: var(--send-button-color);
  color: var(--white);
  border-radius: 50%;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.3s ease;
  border: none;
  cursor: pointer;
}

.btn-send:hover {
  transform: translateY(-50%) scale(1.1); /* 修改这里 */
  background-color: var(--primary-color);
}

.btn-send .material-icons {
  font-size: 1.2rem;
}

.editor-preview {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background-color: var(--white);
  border-radius: 16px;
  box-shadow: var(--card-shadow);
  position: relative;
}

.editor-navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background-color: #BBDEFB; /* 保持为淡蓝色 */
  border-bottom: 1px solid #90CAF9; /* 稍微深一点的边框颜色 */
  z-index: 10;
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 56px;
  box-sizing: border-box;
}

.editor-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  margin-top: 56px; /* 为导航栏留出空间 */
}

.code-preview,
.uml-diagram {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.editor-switch,
.editor-controls {
  display: flex;
  gap: 8px;
}

.editor-switch .btn-icon,
.editor-controls .btn-icon {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  border: none;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
  cursor: pointer;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.editor-switch .btn-icon:hover,
.editor-controls .btn-icon:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
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
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  border: none;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
  cursor: pointer;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.new-chat:hover,
.new-chat:active,
.new-chat:focus {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.new-chat:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.editor-switch .btn-icon,
.editor-controls .btn-icon {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  border: none;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
  cursor: pointer;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.editor-switch .btn-icon:hover,
.editor-controls .btn-icon:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.editor-switch .btn-icon:active,
.editor-controls .btn-icon:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
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

.hamburger-menu {
  background: none;
  border: none;
  color: var(--white);
  font-size: 24px;
  cursor: pointer;
  padding: 0;
  margin-right: 15px;
}

@media (max-width: 1200px) {
  .main-content {
    flex-direction: row;
  }

  .chat-area {
    width: 40%;
    min-width: 250px;
  }

  .editor-preview {
    width: 60%;
  }
}

@media (max-width: 768px) {
  .main-content {
    flex-direction: column;
  }

  .chat-area,
  .editor-preview {
    width: 100%;
    margin-right: 0;
    margin-bottom: 1rem;
  }
}

@media (max-width: 300px) {
  .chat-nav {
    width: 100%;
    position: relative;
    margin-bottom: 1rem;
  }

  .chat-nav-hidden {
    transform: translateY(-110%);
  }

  .main-content.nav-open {
    margin-left: 0;
  }
}

/* 自定义滚动条样式 */
.chat-messages::-webkit-scrollbar {
  width: 8px;  /* 滚动条宽度 */
}

.chat-messages::-webkit-scrollbar-track {
  background: #f1f1f1;  /* 滚动条轨道背景色 */
  border-radius: 4px;
}

.chat-messages::-webkit-scrollbar-thumb {
  background: #E1F5FE;  /* 滚动条滑块颜色（更淡的蓝色） */
  border-radius: 4px;
}

.chat-messages::-webkit-scrollbar-thumb:hover {
  background: #B3E5FC;  /* 悬停时的滚条滑块颜色（稍深一点的淡蓝色） */
}

.editor-switch {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 10;
}

.editor-controls {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 10;
  display: flex;
  gap: 8px;
}

.editor-switch .btn-icon,
.editor-controls .btn-icon {
  background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
  color: var(--white);
  border: none;
  width: 36px;
  height: 36px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.editor-switch .btn-icon:hover,
.editor-controls .btn-icon:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.uml-diagram {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.uml-image-container {
  width: 100%;
  height: 100%;
  overflow: auto;
  display: flex;
  justify-content: center;
  align-items: center;
}

.uml-image-container img {
  max-width: 100%; /* 确保图片不会超出容器宽度 */
  max-height: 100%; /* 确保图片不会超出容器高度 */
  transform-origin: center center; /* 确保缩放是从中心开始的 */
}
</style>
























































