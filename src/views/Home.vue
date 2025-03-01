<script setup lang="ts">
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { v4 as uuidv4 } from 'uuid'

const router = useRouter()
const username = ref('')
const joinRoomId = ref('')
const errorMessage = ref('')

const createNewRoom = () => {
  if (!username.value.trim()) {
    errorMessage.value = 'Please enter a username'
    return
  }
  
  const roomId = uuidv4()
  localStorage.setItem('username', username.value)
  // Store that we created this room
  localStorage.setItem('createdRoomId', roomId)
  router.push(`/chat/${roomId}`)
}

const joinRoom = () => {
  if (!username.value.trim()) {
    errorMessage.value = 'Please enter a username'
    return
  }
  
  if (!joinRoomId.value.trim()) {
    errorMessage.value = 'Please enter a room ID'
    return
  }
  
  localStorage.setItem('username', username.value)
  // Clear any previously created room ID
  localStorage.removeItem('createdRoomId')
  router.push(`/chat/${joinRoomId.value}`)
}

const copyInviteLink = () => {
  if (!joinRoomId.value.trim()) {
    errorMessage.value = 'Please enter a room ID to generate an invite link'
    return
  }
  
  const inviteLink = `${window.location.origin}/chat/${joinRoomId.value}`
  navigator.clipboard.writeText(inviteLink)
    .then(() => {
      errorMessage.value = 'Invite link copied to clipboard!'
      setTimeout(() => {
        errorMessage.value = ''
      }, 3000)
    })
    .catch(err => {
      errorMessage.value = 'Failed to copy invite link'
      console.error('Failed to copy: ', err)
    })
}
</script>

<template>
  <div class="container home-container">
    <div class="row justify-content-center">
      <div class="col-md-6">
        <div class="card shadow-lg p-4 mt-5">
          <h1 class="text-center mb-4">P2P Chat</h1>
          <p class="text-center text-muted mb-4">
            Secure, decentralized chat with no server storage
          </p>
          
          <div class="mb-3">
            <label for="username" class="form-label">Your Name</label>
            <input 
              type="text" 
              class="form-control bg-dark text-light" 
              id="username" 
              v-model="username" 
              placeholder="Enter your name"
            >
          </div>
          
          <div class="d-grid gap-2 mb-4">
            <button 
              class="btn btn-primary" 
              @click="createNewRoom"
            >
              Create New Chat Room
            </button>
          </div>
          
          <div class="separator text-center mb-4">
            <span class="bg-dark px-2">OR</span>
          </div>
          
          <div class="mb-3">
            <label for="roomId" class="form-label">Room ID</label>
            <div class="input-group">
              <input 
                type="text" 
                class="form-control bg-dark text-light" 
                id="roomId" 
                v-model="joinRoomId" 
                placeholder="Enter room ID to join"
              >
              <button 
                class="btn btn-outline-secondary" 
                type="button" 
                @click="copyInviteLink"
              >
                Copy Link
              </button>
            </div>
          </div>
          
          <div class="d-grid gap-2">
            <button 
              class="btn btn-success" 
              @click="joinRoom"
            >
              Join Existing Room
            </button>
          </div>
          
          <div v-if="errorMessage" class="alert alert-warning mt-3" role="alert">
            {{ errorMessage }}
          </div>
          
          <p class="text-center text-muted mt-4 small">
            Chat sessions automatically expire 15 minutes after the last message
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.home-container {
  height: 100%;
  display: flex;
  align-items: center;
}

.separator {
  display: flex;
  align-items: center;
  text-align: center;
  color: #6c757d;
}

.separator::before,
.separator::after {
  content: '';
  flex: 1;
  border-bottom: 1px solid #444;
}

.separator::before {
  margin-right: 0.5em;
}

.separator::after {
  margin-left: 0.5em;
}

.form-control {
  border-color: #333;
}

.form-control:focus {
  border-color: #646cff;
  box-shadow: 0 0 0 0.25rem rgba(100, 108, 255, 0.25);
}
</style>