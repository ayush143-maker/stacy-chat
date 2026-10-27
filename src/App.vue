<script setup>
import { computed, nextTick, onMounted, ref, watch } from 'vue'
import {
  ArrowUpRight, CheckCircle2, CircleHelp, Copy, Eye, EyeOff, KeyRound,
  LoaderCircle, Menu, MessageCircle, MessageSquare, Plus, RefreshCw, Search,
  Send, ShieldCheck, Sparkles, Square, Trash2, X
} from 'lucide-vue-next'
import { marked } from 'marked'
import DOMPurify from 'dompurify'

marked.setOptions({ gfm: true, breaks: true })
const CHAT_STORE = 'stacy.chats.v1'
const activeTab = ref('chat')
const chats = ref([])
const activeChatId = ref(null)
const searchQuery = ref('')
const prompt = ref('')
const sidebarOpen = ref(false)
const showApiKey = ref(false)
const draftApiKey = ref('')
const apiKey = ref('')
const models = ref([])
const selectedModel = ref('')
const temperature = ref(0.7)
const isConnecting = ref(false)
const isGenerating = ref(false)
const controller = ref(null)
const statusMessage = ref('')
const connectionError = ref('')
const messagesEl = ref(null)
const composerEl = ref(null)

const activeChat = computed(() => chats.value.find(item => item.id === activeChatId.value) || null)
const activeMessages = computed(() => activeChat.value?.messages || [])
const filteredChats = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  const list = [...chats.value].sort((a, b) => b.updatedAt - a.updatedAt)
  return query ? list.filter(item => item.title.toLowerCase().includes(query) || item.messages.some(m => m.content?.toLowerCase().includes(query))) : list
})
const connected = computed(() => Boolean(apiKey.value && models.value.length))
const currentModel = computed(() => models.value.find(model => model.name === selectedModel.value))
const canSend = computed(() => prompt.value.trim().length > 0 && !isGenerating.value)

function makeId() {
  return globalThis.crypto?.randomUUID?.() || ('stacy-' + Date.now() + '-' + Math.random().toString(36).slice(2, 9))
}
function createChat() {
  const item = { id: makeId(), title: 'New chat', messages: [], createdAt: Date.now(), updatedAt: Date.now() }
  chats.value.unshift(item)
  activeChatId.value = item.id
  sidebarOpen.value = false
  activeTab.value = 'chat'
  nextTick(() => composerEl.value?.focus())
  return item
}
function newChat() {
  if (isGenerating.value) stopGeneration()
  createChat()
}
function chooseChat(id) {
  activeChatId.value = id
  sidebarOpen.value = false
  activeTab.value = 'chat'
  nextTick(scrollToBottom)
}
function removeChat(id) {
  const index = chats.value.findIndex(item => item.id === id)
  if (index < 0) return
  if (activeChatId.value === id && isGenerating.value) stopGeneration()
  chats.value.splice(index, 1)
  if (activeChatId.value === id) activeChatId.value = chats.value[0]?.id || null
}
function clearAllChats() {
  if (!chats.value.length || !window.confirm('Delete all conversations stored on this device? This cannot be undone.')) return
  if (isGenerating.value) stopGeneration()
  chats.value = []
  activeChatId.value = null
}
function displayTime(timestamp) {
  if (!timestamp) return ''
  const date = new Date(timestamp)
  if (date.toDateString() === new Date().toDateString()) return date.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' })
  return date.toLocaleDateString([], { month: 'short', day: 'numeric' })
}
function makeTitle(text) {
  const clean = text.replace(/\s+/g, ' ').trim()
  return clean.length > 34 ? clean.slice(0, 34).trimEnd() + '…' : clean
}
function renderMarkdown(text) {
  return DOMPurify.sanitize(marked.parse(text || ''))
}
function saveChats() {
  try { localStorage.setItem(CHAT_STORE, JSON.stringify(chats.value)) }
  catch { statusMessage.value = 'Local storage is full' }
}
function loadChats() {
  try {
    const parsed = JSON.parse(localStorage.getItem(CHAT_STORE) || '[]')
    chats.value = Array.isArray(parsed) ? parsed.filter(item => item && typeof item.id === 'string' && Array.isArray(item.messages)) : []
  } catch { chats.value = [] }
  activeChatId.value = chats.value[0]?.id || null
}
watch(chats, saveChats, { deep: true })
watch(activeMessages, () => nextTick(scrollToBottom), { deep: true })
watch(activeChatId, () => nextTick(scrollToBottom))
function scrollToBottom() {
  if (messagesEl.value) messagesEl.value.scrollTop = messagesEl.value.scrollHeight
}
function goToKeys() {
  activeTab.value = 'key'
  sidebarOpen.value = false
}
function forgetKey() {
  stopGeneration()
  apiKey.value = ''
  draftApiKey.value = ''
  models.value = []
  selectedModel.value = ''
  connectionError.value = ''
  statusMessage.value = ''
}
async function connectKey() {
  const key = draftApiKey.value.trim()
  if (!key) { connectionError.value = 'Paste your API key first.'; return }
  isConnecting.value = true
  connectionError.value = ''
  statusMessage.value = 'Checking key and loading models…'
  try {
    const response = await fetch('https://generativelanguage.googleapis.com/v1beta/models?pageSize=100', {
      headers: { 'x-goog-api-key': key }
    })
    const data = await response.json()
    if (!response.ok) throw new Error(data?.error?.message || 'Google rejected this key. Check the key and API restrictions.')
    const supported = (data.models || []).filter(model => (model.supportedGenerationMethods || []).includes('generateContent'))
    if (!supported.length) throw new Error('The key worked, but no models supporting generateContent were returned.')
    supported.sort((a, b) => {
      const score = item => {
        const name = (item.name || '').toLowerCase()
        if (name.includes('flash') && !name.includes('lite')) return 0
        if (name.includes('flash')) return 1
        if (name.includes('pro')) return 2
        return 3
      }
      return score(a) - score(b) || (a.displayName || a.name).localeCompare(b.displayName || b.name)
    })
    apiKey.value = key
    models.value = supported
    if (!supported.some(item => item.name === selectedModel.value)) selectedModel.value = supported[0].name
    statusMessage.value = ''
    activeTab.value = 'chat'
  } catch (error) {
    connectionError.value = error.message || 'Could not connect to Google.'
    statusMessage.value = ''
  } finally { isConnecting.value = false }
}
function stopGeneration() {
  if (controller.value) controller.value.abort()
  controller.value = null
  isGenerating.value = false
  statusMessage.value = 'Generation stopped'
}
function buildContents(chat) {
  return chat.messages.filter(item => item.role === 'user' || item.role === 'assistant')
    .map(item => ({ role: item.role === 'assistant' ? 'model' : 'user', parts: [{ text: item.content }] }))
}
async function completeResponse(chat) {
  if (!apiKey.value) { activeTab.value = 'key'; statusMessage.value = 'Add your Gemini API key to start chatting.'; return }
  if (!selectedModel.value) { statusMessage.value = 'Choose a model on the Chat screen.'; return }
  if (isGenerating.value) return
  isGenerating.value = true
  const thisController = new AbortController()
  controller.value = thisController
  statusMessage.value = 'Stacy is thinking…'
  await nextTick(scrollToBottom)
  try {
    const modelPath = selectedModel.value.replace(/^models\//, '')
    const response = await fetch('https://generativelanguage.googleapis.com/v1beta/models/' + encodeURIComponent(modelPath) + ':generateContent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'x-goog-api-key': apiKey.value },
      body: JSON.stringify({ contents: buildContents(chat), generationConfig: { temperature: Number(temperature.value), maxOutputTokens: 8192 } }),
      signal: thisController.signal
    })
    const data = await response.json()
    if (!response.ok) throw new Error(data?.error?.message || 'Request failed with status ' + response.status)
    const answer = (data.candidates?.[0]?.content?.parts || []).map(part => part.text || '').join('').trim()
    if (!answer) {
      const reason = data.promptFeedback?.blockReason
      throw new Error(reason ? 'Google blocked this prompt: ' + reason : 'The selected model returned no text. Try another model or prompt.')
    }
    chat.messages.push({ id: makeId(), role: 'assistant', content: answer, createdAt: Date.now() })
    chat.updatedAt = Date.now()
    statusMessage.value = 'Reply received'
  } catch (error) {
    if (error.name === 'AbortError') statusMessage.value = 'Generation stopped'
    else {
      chat.messages.push({ id: makeId(), role: 'error', content: error.message || 'Something went wrong.', createdAt: Date.now() })
      chat.updatedAt = Date.now()
      statusMessage.value = 'Request failed'
    }
  } finally {
    if (controller.value === thisController) controller.value = null
    isGenerating.value = false
    await nextTick(scrollToBottom)
    composerEl.value?.focus()
  }
}
async function sendMessage() {
  const text = prompt.value.trim()
  if (!text || isGenerating.value) return
  if (!apiKey.value) { goToKeys(); statusMessage.value = 'Add your Gemini API key before sending a message.'; return }
  let chat = activeChat.value
  if (!chat) chat = createChat()
  chat.messages.push({ id: makeId(), role: 'user', content: text, createdAt: Date.now() })
  if (chat.title === 'New chat') chat.title = makeTitle(text)
  chat.updatedAt = Date.now()
  prompt.value = ''
  await completeResponse(chat)
}
async function regenerate() {
  const chat = activeChat.value
  if (!chat || isGenerating.value) return
  const last = chat.messages[chat.messages.length - 1]
  if (last?.role === 'assistant' || last?.role === 'error') chat.messages.pop()
  if (!chat.messages.some(item => item.role === 'user')) return
  await completeResponse(chat)
}
async function copyText(text) {
  try {
    await navigator.clipboard.writeText(text)
    statusMessage.value = 'Copied to clipboard'
    window.setTimeout(() => { if (statusMessage.value === 'Copied to clipboard') statusMessage.value = '' }, 1800)
  } catch { statusMessage.value = 'Clipboard unavailable in this browser' }
}
function onComposerKeydown(event) {
  if (event.key === 'Enter' && !event.shiftKey && !event.isComposing) {
    event.preventDefault()
    sendMessage()
  }
}
function useSuggestion(text) {
  prompt.value = text
  nextTick(() => composerEl.value?.focus())
}
onMounted(loadChats)
</script>

<template>
  <div class="app-shell">
    <header class="topbar">
      <div class="brand-cluster"><div class="brand-name">Stacy</div></div>
      <nav class="main-tabs" aria-label="Main sections">
        <button :class="['nav-tab', { selected: activeTab === 'chat' }]" @click="activeTab = 'chat'"><MessageSquare :size="16" /><span>Chat</span></button>
        <button :class="['nav-tab', { selected: activeTab === 'key' }]" @click="activeTab = 'key'"><KeyRound :size="16" /><span>API key</span><span :class="['connection-dot', { online: connected }]"></span></button>
      </nav>
    </header>

    <div class="workspace">
      <div v-if="sidebarOpen && activeTab === 'chat'" class="mobile-scrim" @click="sidebarOpen = false"></div>
      <aside v-if="activeTab === 'chat'" :class="['sidebar', { 'sidebar-open': sidebarOpen }]">
        <div class="sidebar-heading"><span>Your chats</span><button class="icon-button small" title="Start a new chat" aria-label="New chat" @click="newChat"><Plus :size="17" /></button></div>
        <button class="new-chat-button" @click="newChat"><Plus :size="17" /><span>New conversation</span></button>
        <label class="search-field"><Search :size="15" /><input v-model="searchQuery" aria-label="Search conversations" placeholder="Find a conversation" /><button v-if="searchQuery" class="mini-clear" aria-label="Clear search" @click="searchQuery = ''"><X :size="13" /></button></label>
        <div class="conversation-label">RECENT</div>
        <div class="conversation-list">
          <div v-if="!filteredChats.length" class="no-chats">{{ searchQuery ? 'No matching conversations.' : 'Your conversations will show up here.' }}</div>
          <div v-for="item in filteredChats" :key="item.id" :class="['conversation-row', { active: item.id === activeChatId }]" @click="chooseChat(item.id)">
            <MessageSquare :size="16" class="conversation-icon" />
            <button class="conversation-main" :title="item.title" @click.stop="chooseChat(item.id)"><span class="conversation-title">{{ item.title }}</span><span class="conversation-time">{{ displayTime(item.updatedAt) }}</span></button>
            <button class="icon-button delete-chat" title="Delete conversation" aria-label="Delete conversation" @click.stop="removeChat(item.id)"><Trash2 :size="14" /></button>
          </div>
        </div>
        <div class="sidebar-bottom">
          <button class="account-row" @click="goToKeys"><div class="account-avatar"><KeyRound :size="16" /></div><div class="account-copy"><strong>Gemini connection</strong><span>{{ connected ? models.length + ' models available' : 'Add your API key' }}</span></div><ArrowUpRight :size="15" class="account-arrow" /></button>
          <div class="storage-line"><ShieldCheck :size="13" /><span>Chats stay on this device</span></div>
          <button v-if="chats.length" class="clear-all-button" @click="clearAllChats">Clear all conversations</button>
        </div>
      </aside>

      <main class="main-panel">
        <section v-if="activeTab === 'chat'" class="chat-view">
          <div class="chat-toolbar">
            <div class="chat-heading"><div class="chat-heading-titleline"><button class="icon-button history-toggle" title="Chat history" aria-label="Open chat history" @click="sidebarOpen = !sidebarOpen"><Menu :size="16" /></button><div class="heading-copy"><span class="eyebrow">{{ activeChat ? 'CONVERSATION' : 'YOUR SPACE' }}</span><h1>{{ activeChat?.title || 'What’s on your mind?' }}</h1></div></div></div>
            <div class="toolbar-controls"><div class="model-control"><span class="control-caption">MODEL</span><select v-model="selectedModel" :disabled="!connected" aria-label="Choose Gemini model"><option v-if="!models.length" value="">Connect a key first</option><option v-for="model in models" :key="model.name" :value="model.name">{{ model.displayName || model.name.replace('models/', '') }}</option></select></div><div class="temperature-control"><label for="temperature">Creativity <strong>{{ Number(temperature).toFixed(1) }}</strong></label><input id="temperature" v-model.number="temperature" type="range" min="0" max="2" step="0.1" /></div></div>
          </div>
          <div ref="messagesEl" class="messages-scroll">
            <div v-if="!activeMessages.length" class="welcome-panel">
              <div class="welcome-orbit"><span></span><Sparkles :size="24" /></div>
              <div class="welcome-copy"><p class="welcome-kicker">A little space to think</p><h2>Hi. I’m Stacy.</h2><p class="welcome-description">Ask a question, work through an idea, or just start typing. We can pick it up from there.</p></div>
              <div class="suggestion-grid">
                <button @click="useSuggestion('Explain a difficult concept to me in simple steps.')"><span>Understand something</span><ArrowUpRight :size="16" /></button>
                <button @click="useSuggestion('Help me brainstorm a few original ideas for a project.')"><span>Think of new ideas</span><ArrowUpRight :size="16" /></button>
                <button @click="useSuggestion('Review this text and suggest ways to improve it:')"><span>Write or improve text</span><ArrowUpRight :size="16" /></button>
                <button @click="useSuggestion('Help me plan my next steps for this goal:')"><span>Make a plan</span><ArrowUpRight :size="16" /></button>
              </div>
              <div v-if="!connected" class="welcome-connection"><KeyRound :size="16" /><span>Add your Gemini API key to start chatting.</span><button @click="goToKeys">Set up <ArrowUpRight :size="14" /></button></div>
            </div>
            <div v-else class="message-list">
              <article v-for="(message, index) in activeMessages" :key="message.id" :class="['message-row', 'message-' + message.role]">
                <div v-if="message.role === 'assistant'" class="message-avatar assistant-avatar"><MessageCircle :size="16" /></div>
                <div v-else-if="message.role === 'error'" class="message-avatar error-avatar"><CircleHelp :size="16" /></div>
                <div v-else class="message-avatar user-avatar">Y</div>
                <div class="message-content-wrap"><div class="message-meta"><strong>{{ message.role === 'user' ? 'You' : message.role === 'error' ? 'Couldn’t reply' : 'Stacy' }}</strong><time>{{ displayTime(message.createdAt) }}</time></div>
                  <div v-if="message.role === 'assistant'" class="markdown-body" v-html="renderMarkdown(message.content)"></div><p v-else class="plain-message">{{ message.content }}</p>
                  <div v-if="message.role !== 'user'" class="message-actions"><button class="text-action" @click="copyText(message.content)"><Copy :size="13" /> Copy</button><button v-if="message.role === 'assistant' && index === activeMessages.length - 1" class="text-action" :disabled="isGenerating" @click="regenerate"><RefreshCw :size="13" /> Try again</button></div>
                </div>
              </article>
              <div v-if="isGenerating" class="message-row message-assistant typing-row"><div class="message-avatar assistant-avatar"><MessageCircle :size="16" /></div><div class="typing-card"><span class="typing-label">Stacy is thinking</span><span class="typing-dots"><i></i><i></i><i></i></span></div></div>
            </div>
          </div>
          <div class="composer-wrap">
            <div v-if="!connected" class="composer-key-hint"><KeyRound :size="14" /> Connect your API key to send a message. <button @click="goToKeys">Open settings <ArrowUpRight :size="12" /></button></div>
            <div class="composer-box"><textarea ref="composerEl" v-model="prompt" :disabled="isGenerating" rows="1" placeholder="Message Stacy…" aria-label="Message Stacy" @keydown="onComposerKeydown"></textarea><div class="composer-footer"><div class="composer-left"><span class="safe-note"><ShieldCheck :size="13" /> Your key isn’t saved</span><span class="composer-divider">·</span><span>Enter to send</span></div><button v-if="isGenerating" class="send-button stop-button" aria-label="Stop response" @click="stopGeneration"><Square :size="14" fill="currentColor" /><span>Stop</span></button><button v-else class="send-button" :disabled="!canSend" aria-label="Send message" @click="sendMessage"><Send :size="15" /><span>Send</span></button></div></div>
            <div class="composer-disclaimer">Stacy can make mistakes. Check important information.</div><div v-if="statusMessage" class="status-message">{{ statusMessage }}</div>
          </div>
        </section>

        <section v-else class="settings-view">
          <div class="settings-container">
            <div class="settings-title-row"><div class="settings-icon"><KeyRound :size="21" /></div><div><span class="eyebrow">CONNECTION</span><h1>API key</h1><p>Connect your own Gemini API key to start a conversation.</p></div></div>
            <div class="settings-card">
              <div class="card-heading"><div><h2>Google Gemini</h2><p>Use a key from Google AI Studio.</p></div><span :class="['connection-badge', { active: connected }]"><span class="status-dot"></span>{{ connected ? 'Connected' : 'Not connected' }}</span></div>
              <label class="form-label" for="apiKey">API key</label>
              <div class="key-input-row"><input id="apiKey" v-model="draftApiKey" :type="showApiKey ? 'text' : 'password'" placeholder="Paste your Gemini API key" autocomplete="off" spellcheck="false" @keydown.enter="connectKey" /><button class="icon-button input-eye" :aria-label="showApiKey ? 'Hide API key' : 'Show API key'" @click="showApiKey = !showApiKey"><EyeOff v-if="showApiKey" :size="17" /><Eye v-else :size="17" /></button></div>
              <p class="field-help">The key is held in this page’s memory only. Refreshing or closing the tab clears it.</p>
              <div v-if="connectionError" class="error-note"><X :size="15" /><span>{{ connectionError }}</span></div><div v-if="statusMessage && connected" class="success-note"><CheckCircle2 :size="15" /><span>{{ statusMessage }}</span></div>
              <div class="key-actions"><button class="primary-button" :disabled="isConnecting || !draftApiKey.trim()" @click="connectKey"><LoaderCircle v-if="isConnecting" :size="16" class="spin" /><ShieldCheck v-else :size="16" />{{ isConnecting ? 'Checking key…' : connected ? 'Reconnect & refresh models' : 'Connect key' }}</button><button v-if="connected" class="secondary-button" @click="forgetKey">Forget key</button></div>
              <div v-if="connected" class="model-summary"><div><span class="summary-number">{{ models.length }}</span><span class="summary-label">compatible models</span></div><div class="summary-divider"></div><div><span class="summary-number">{{ currentModel?.displayName || 'Ready' }}</span><span class="summary-label">selected in chat</span></div></div>
            </div>
            <div class="security-card"><div class="security-icon"><ShieldCheck :size="18" /></div><div class="security-copy"><h3>A note on your key</h3><p>This is a browser-only app. The key is sent directly from your browser to Google’s Gemini API and is not sent to Stacy’s server. Anyone who can inspect this browser session may be able to see it, so use this as a personal tool, restrict the key where Google supports it, and monitor usage and quota.</p><a href="https://aistudio.google.com/apikey" target="_blank" rel="noreferrer">Get or manage a key <ArrowUpRight :size="13" /></a></div></div>
            <div class="settings-footnote"><span class="footnote-dot"></span><span>Conversation history is saved locally in this browser; API keys are not stored with it.</span></div>
            <button class="back-to-chat" @click="activeTab = 'chat'"><MessageSquare :size="16" /> Back to chat <ArrowUpRight :size="14" /></button>
          </div>
        </section>
      </main>
    </div>
  </div>
</template>