<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import Peer from 'peerjs'

interface Message {
  id: string;
  sender: string;
  senderId: string;
  content: string;
  timestamp: number;
}

interface Participant {
  id: string;
  username: string;
  isConnected: boolean;
}

const route = useRoute()
const router = useRouter()
const roomId = computed(() => route.params.roomId as string)
const username = ref(localStorage.getItem('username') || 'Anonymous')
const message = ref('')
const messages = ref<Message[]>([])
const participants = ref<Participant[]>([])
const peer = ref<Peer | null>(null)
const connections = ref<Record<string, any>>({})
const peerId = ref('')
const isConnected = ref(false)
const errorMessage = ref('')
const lastActivityTime = ref(Date.now())
const sessionTimeout = 15 * 60 * 1000 // 15 minutes in milliseconds
const timeoutCheckInterval = ref<number | null>(null)
const copySuccess = ref(false)
const isRoomCreator = ref(false)

// Check if username exists, if not redirect to home
if (!username.value) {
  router.push('/')
}

// Initialize PeerJS connection
onMounted(() => {
  initializePeer()
  setupTimeoutCheck()
  
  // Handle page unload
  window.addEventListener('beforeunload', handleDisconnect)
})

onBeforeUnmount(() => {
  handleDisconnect()
  if (timeoutCheckInterval.value) {
    clearInterval(timeoutCheckInterval.value)
  }
  window.removeEventListener('beforeunload', handleDisconnect)
})

// Watch for activity to reset timeout
watch(messages, () => {
  lastActivityTime.value = Date.now()
})

const initializePeer = () => {
  // Check if we're the room creator by comparing URL with localStorage
  const storedRoomId = localStorage.getItem('createdRoomId')
  isRoomCreator.value = storedRoomId === roomId.value
  
  // If we're the room creator, use the roomId as our peerId
  // Otherwise, generate a random peerId
  const customPeerId = isRoomCreator.value ? roomId.value : undefined
  
  peer.value = new Peer(customPeerId, {
    debug: 2,
    config: {
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        { urls: 'stun:global.stun.twilio.com:3478' }
      ]
    }
  })

  peer.value.on('open', (id) => {
    peerId.value = id
    isConnected.value = true
    
    // Add self to participants
    participants.value.push({
      id: peerId.value,
      username: username.value,
      isConnected: true
    })
    
    // Join the room if we're not the creator
    if (!isRoomCreator.value) {
      joinRoom()
    }
  })

  peer.value.on('connection', (conn) => {
    handleConnection(conn)
  })

  peer.value.on('error', (err) => {
    console.error('PeerJS error:', err)
    
    // Handle specific error types
    if (err.type === 'peer-unavailable') {
      errorMessage.value = 'Could not connect to the room. The room may not exist or the host is offline.'
    } else if (err.type === 'network' || err.type === 'disconnected') {
      errorMessage.value = 'Network connection issue. Please check your internet connection.'
    } else if (err.type === 'server-error') {
      errorMessage.value = 'PeerJS server error. Please try again later.'
    } else {
      errorMessage.value = `Connection error: ${err.message || 'Unknown error'}`
    }
    
    // Clear error message after 5 seconds
    setTimeout(() => {
      errorMessage.value = ''
    }, 5000)
  })
}

const joinRoom = () => {
  // Only try to connect if we're not the room creator
  if (!isRoomCreator.value) {
    const conn = peer.value?.connect(roomId.value, {
      reliable: true,
      metadata: {
        username: username.value,
        joinRoom: true
      }
    })
    
    if (conn) {
      handleConnection(conn)
    } else {
      errorMessage.value = 'Failed to establish connection to the room.'
    }
  }
}

const handleConnection = (conn: any) => {
  connections.value[conn.peer] = conn
  
  conn.on('open', () => {
    // Clear any error messages when connection is successful
    errorMessage.value = ''
    
    // Send our user info
    conn.send({
      type: 'USER_INFO',
      data: {
        id: peerId.value,
        username: username.value
      }
    })
    
    // If this is a new peer joining, send them all current messages
    if (conn.metadata?.joinRoom) {
      conn.send({
        type: 'INIT_MESSAGES',
        data: messages.value
      })
      
      conn.send({
        type: 'INIT_PARTICIPANTS',
        data: participants.value
      })
    }
  })
  
  conn.on('data', (data: any) => {
    handlePeerData(conn.peer, data)
  })
  
  conn.on('close', () => {
    // Remove connection
    delete connections.value[conn.peer]
    
    // Update participant status
    const participantIndex = participants.value.findIndex(p => p.id === conn.peer)
    if (participantIndex !== -1) {
      participants.value[participantIndex].isConnected = false
    }
  })
  
  conn.on('error', (err: any) => {
    console.error('Connection error:', err)
    errorMessage.value = `Connection error: ${err.message || 'Unknown error'}`
    
    // Clear error message after 5 seconds
    setTimeout(() => {
      errorMessage.value = ''
    }, 5000)
  })
}

const handlePeerData = (peerId: string, data: any) => {
  switch (data.type) {
    case 'USER_INFO':
      // Add user to participants if not already there
      if (!participants.value.some(p => p.id === data.data.id)) {
        participants.value.push({
          id: data.data.id,
          username: data.data.username,
          isConnected: true
        })
      }
      break
      
    case 'INIT_MESSAGES':
      // Merge received messages with our messages
      const newMessages = data.data.filter((msg: Message) => 
        !messages.value.some(m => m.id === msg.id)
      )
      messages.value = [...messages.value, ...newMessages].sort((a, b) => a.timestamp - b.timestamp)
      break
      
    case 'INIT_PARTICIPANTS':
      // Merge received participants with our participants
      const newParticipants = data.data.filter((p: Participant) => 
        !participants.value.some(existing => existing.id === p.id)
      )
      participants.value = [...participants.value, ...newParticipants]
      break
      
    case 'NEW_MESSAGE':
      // Add new message if we don't already have it
      if (!messages.value.some(m => m.id === data.data.id)) {
        messages.value.push(data.data)
        // Sort messages by timestamp
        messages.value.sort((a, b) => a.timestamp - b.timestamp)
      }
      break
  }
}

const sendMessage = () => {
  if (!message.value.trim() || !isConnected.value) return
  
  const newMessage: Message = {
    id: `${peerId.value}-${Date.now()}`,
    sender: username.value,
    senderId: peerId.value,
    content: message.value,
    timestamp: Date.now()
  }
  
  // Add to our messages
  messages.value.push(newMessage)
  
  // Send to all connected peers
  Object.values(connections.value).forEach(conn => {
    conn.send({
      type: 'NEW_MESSAGE',
      data: newMessage
    })
  })
  
  // Reset message input
  message.value = ''
  
  // Update last activity time
  lastActivityTime.value = Date.now()
}

const handleDisconnect = () => {
  // Close all connections
  Object.values(connections.value).forEach((conn: any) => {
    if (conn.open) {
      conn.close()
    }
  })
  
  // Close peer connection
  if (peer.value && !peer.value.destroyed) {
    peer.value.destroy()
  }
}

const setupTimeoutCheck = () => {
  timeoutCheckInterval.value = setInterval(() => {
    const currentTime = Date.now()
    const timeSinceLastActivity = currentTime - lastActivityTime.value
    
    if (timeSinceLastActivity > sessionTimeout) {
      // Session expired, redirect to home
      router.push('/')
    }
  }, 60000) // Check every minute
}

const formatTime = (timestamp: number) => {
  const date = new Date(timestamp)
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
}

const copyInviteLink = () => {
  const inviteLink = `${window.location.origin}/chat/${roomId.value}`
  navigator.clipboard.writeText(inviteLink)
    .then(() => {
      copySuccess.value = true
      setTimeout(() => {
        copySuccess.value = false
      }, 3000)
    })
    .catch(err => {
      console.error('Failed to copy: ', err)
    })
}

const getTimeLeft = computed(() => {
  const timeSinceLastActivity = Date.now() - lastActivityTime.value
  const timeLeft = sessionTimeout - timeSinceLastActivity
  
  if (timeLeft <= 0) return '0m'
  
  const minutes = Math.floor(timeLeft / 60000)
  return `${minutes}m`
})
</script>

<template>
  <div class="chat-container">
    <!-- Header -->
    <div class="chat-header">
      <div class="container">
        <div class="d-flex justify-content-between align-items-center py-2">
          <h2 class="mb-0">P2P Chat</h2>
          <div class="d-flex align-items-center">
            <span class="badge bg-secondary me-2">
              Session expires in: {{ getTimeLeft }}
            </span>
            <button 
              class="btn btn-sm btn-outline-light me-2" 
              @click="copyInviteLink"
              :class="{ 'btn-success': copySuccess }"
            >
              {{ copySuccess ? 'Copied!' : 'Invite' }}
            </button>
            <button 
              class="btn btn-sm btn-outline-danger" 
              @click="router.push('/')"
            >
              Leave
            </button>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Main chat area -->
    <div class="container chat-content">
      <div class="row h-100">
        <!-- Participants sidebar -->
        <div class="col-md-3 d-none d-md-block">
          <div class="participants-container">
            <h5 class="mb-3">Participants</h5>
            <ul class="list-group participants-list">
              <li 
                v-for="participant in participants" 
                :key="participant.id"
                class="list-group-item d-flex justify-content-between align-items-center"
              >
                <span>{{ participant.username }}</span>
                <span 
                  class="status-indicator" 
                  :class="{ 'online': participant.isConnected, 'offline': !participant.isConnected }"
                ></span>
              </li>
            </ul>
          </div>
        </div>
        
        <!-- Messages area -->
        <div class="col-md-9">
          <div class="messages-container">
            <div class="messages-list" ref="messagesList">
              <div 
                v-for="msg in messages" 
                :key="msg.id"
                class="message-bubble"
                :class="{ 'my-message': msg.senderId === peerId, 'other-message': msg.senderId !== peerId }"
              >
                <div class="message-sender" v-if="msg.senderId !== peerId">
                  {{ msg.sender }}
                </div>
                <div class="message-content">
                  {{ msg.content }}
                </div>
                <div class="message-time">
                  {{ formatTime(msg.timestamp) }}
                </div>
              </div>
              
              <div v-if="messages.length === 0" class="empty-state">
                <p>No messages yet. Start the conversation!</p>
                <p v-if="isRoomCreator" class="mt-2">
                  <small class="text-muted">You are the room creator. Share the invite link for others to join.</small>
                </p>
              </div>
            </div>
            
            <!-- Message input -->
            <div class="message-input">
              <div class="input-group">
                <input 
                  type="text" 
                  class="form-control bg-dark text-light" 
                  placeholder="Type your message..." 
                  v-model="message"
                  @keyup.enter="sendMessage"
                >
                <button 
                  class="btn btn-primary" 
                  type="button"
                  @click="sendMessage"
                  :disabled="!isConnected || !message.trim()"
                >
                  Send
                </button>
              </div>
              <div v-if="errorMessage" class="alert alert-danger mt-2 p-2 small">
                {{ errorMessage }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.chat-header {
  background-color: #1a1a1a;
  border-bottom: 1px solid #333;
  flex-shrink: 0;
}

.chat-content {
  flex-grow: 1;
  overflow: hidden;
  padding-top: 1rem;
  padding-bottom: 1rem;
}

.participants-container {
  background-color: #1e1e1e;
  border-radius: 8px;
  padding: 1rem;
  height: 100%;
  overflow-y: auto;
}

.participants-list {
  max-height: calc(100% - 40px);
  overflow-y: auto;
}

.participants-list .list-group-item {
  background-color: #2a2a2a;
  border-color: #333;
  color: #f5f5f5;
  margin-bottom: 0.5rem;
}

.status-indicator {
  width: 10px;
  height: 10px;
  border-radius: 50%;
}

.status-indicator.online {
  background-color: #28a745;
}

.status-indicator.offline {
  background-color: #dc3545;
}

.messages-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #1e1e1e;
  border-radius: 8px;
  overflow: hidden;
}

.messages-list {
  flex-grow: 1;
  overflow-y: auto;
  padding: 1rem;
  display: flex;
  flex-direction: column;
}

.message-bubble {
  max-width: 75%;
  margin-bottom: 1rem;
  padding: 0.75rem;
  border-radius: 1rem;
  position: relative;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.my-message {
  align-self: flex-end;
  background-color: #0d6efd;
  color: white;
  border-bottom-right-radius: 0.25rem;
}

.other-message {
  align-self: flex-start;
  background-color: #2a2a2a;
  color: #f5f5f5;
  border-bottom-left-radius: 0.25rem;
}

.message-sender {
  font-size: 0.8rem;
  margin-bottom: 0.25rem;
  font-weight: bold;
  color: #aaa;
}

.message-content {
  word-break: break-word;
}

.message-time {
  font-size: 0.7rem;
  text-align: right;
  margin-top: 0.25rem;
  opacity: 0.7;
}

.message-input {
  padding: 1rem;
  background-color: #1a1a1a;
  border-top: 1px solid #333;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #6c757d;
  text-align: center;
  padding: 2rem;
}

/* Responsive adjustments */
@media (max-width: 767.98px) {
  .messages-container {
    height: calc(100vh - 70px);
  }
  
  .message-bubble {
    max-width: 85%;
  }
}
</style>