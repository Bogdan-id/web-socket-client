<template>
  <div class="about">
    <!-- Sign-in card -->
    <div 
      v-if="!register"
      class="card">
      <h4 class="sign-in-title">Укажите имя пользователя</h4>
      <div class="input-wrapper">
        <input 
          v-if="!register"
          v-model="user"
          class="form-input"
          lable="текст"
          placeholder="Имя пользователя..."
          type="text"/>
      </div>
      <div class="action-wrapper">
        <button 
          v-if="!register"
          @click="submitUser"
          class="form-btn">
          Зарегистрировать
        </button>
      </div>
    </div>
    <!-- Chat window -->
    <div 
      v-if="register"
      class="card">
      <h1 class="chat-title">Веб-сокет: {{ user }}</h1>
      <div class="socket-block">
        <div 
          class="socket-message"
          @mouseenter="showReply = k"
          @mouseleave="showReply = null"
          v-for="(user, k) in messages"
          :key="k"
          :class="{ 'socket-self-message': key === user.key }">
          <button 
            v-if="showReply === k && key !== user.key"
            @click="activateReply(user)"
            class="reply-btn">
            <Reply />
          </button>
          <div v-if="!user.toMessage">
            <span>{{ user.name + ": " + user.message }}</span>
            <div class="msg-date-wrapper">
              <span>{{ user.dateTime }}</span>
            </div>
          </div>
          <div v-if="user.toMessage">
            <div class="repl-msg">
              <span>{{ user.toMessage }}</span>
            </div>
            <div class="ansr-msg">
              <span>{{ user.name + ": " + user.message }}</span>
            </div>
            <div class="msg-date-wrapper">
              <span>{{ user.dateTime }}</span>
            </div>
          </div>
        </div>
      </div>
      <div 
        v-if="replyState" 
        class="reply-section">
        <button 
          @click="replyState = !replyState"
          class="close-reply-btn">
          <CloseIcon />
        </button>
        <span>{{ replyMessage }}</span>
      </div>
      <div class="message-wrapper">
        <textarea 
          @keyup.enter="submit()"
          rows="3" 
          id="area"
          v-model="message"
          lable="текст"
          placeholder="message..."
          type="text">
        </textarea>
        <div class="sbmt-btn-wrap">
          <button 
            v-if="message"
            @click="submit()"
            id="submit-btn">
            <SendIcon />
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
// @ts-check
import { defineComponent, ref } from 'vue'
/* eslint-disable no-unused-vars */
import { TypeSocket } from 'bogdan-t-type-ws'
/* eslint-enable no-unused-vars */
import SendIcon from '@/assets/send.vue'
import CloseIcon from '@/assets/close.vue'
import Reply from '@/assets/reply.vue'

export default defineComponent({
  name: 'sign-in',
  components: { SendIcon, CloseIcon, Reply },
  setup() {
    const socket = connectSocket()

    const user = ref('')
    const message = ref('')
    const replyMessage = ref('')
    const messages = ref([])
    const register = ref(false)
    const replyState = ref(false)
    const showReply = ref('')
    const key = ref('')
    const keyTo = ref('')

    function toggleView() {
      const el = document.querySelector('.socket-block')
      if (!el) return
      el.scrollTo({
        top: el.scrollHeight,
        left: 0,
      })
    }

    function submit() {
      if (message.value) {
        const date = new Date().toLocaleDateString('Ru')
        const time = new Date().toLocaleTimeString('Ru') 
        const replyObject = replyState.value
          ? {
              to: keyTo.value,
              toMessage: replyMessage.value,
            }
          : {}
        const object = {
          user: {
            key: key.value,
            message: message.value,
            name: user.value,
            dateTime: date + ' ' + time,
            ...replyObject,
          }, 
          room: 'chat-room',
        }

        socket.send(JSON.stringify(object))
        message.value = null
        if (replyState.value) replyState.value = false
      }
    }

    const submitUser = () => {
      if (user.value.length >= 3) {
        register.value = !register.value
        console.log('User successfuly signed in')
      } else {
        console.log('Invalid user name')
      }
    }

    const activateReply = (user) => {
      replyState.value = true
      replyMessage.value = user.name + ": " + user.message 
      keyTo.value = user.key
      document.getElementById('area').focus()
    }

    function connectSocket() {
      const socket = new WebSocket('ws://obscure-anchorage-43966.herokuapp.com/') // localhost:8000

      socket.onopen = function(e) {
        console.log('connection opened', e)
        socket.send(JSON.stringify({
            room: 'chat-room', 
            meta: 'join'
          })
        )
      }
      socket.onmessage = function(event) {
        const message = JSON.parse(event.data)
        if (message.authorize) {
          key.value = message.authorize
        }
        if (Array.isArray(message)) {
          messages.value.push(...message)
        } else if (!message.authorize) {
          messages.value.push(message)
        }
        setTimeout(() => {
          toggleView()
        }, 0)
      };

      socket.onclose = function(event) {
        if (event.wasClean) {
          console.log(`Connection closed cleanly, code=${event.code} reason=${event.reason}`)
        } else {
          console.log('Connection died')
        }
      };

      /** @param {any} error */
      socket.onerror = function(error) {
        console.log(`[error] ${error.message}`)
      }
      return socket
    }

    return {
      message,
      messages,
      user,
      submit,
      register,
      submitUser,
      showReply,
      replyState,
      activateReply,
      replyMessage,
      key,
    }
  }
})
</script>
<style>
body {
  font-family: sans-serif;
  height: 100%;
  background: #77889994;
}
body::-webkit-scrollbar {
  width: 1em;
}
 
body::-webkit-scrollbar-track {
  -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,0.3);
}
 
body::-webkit-scrollbar-thumb {
  background-color: darkgrey;
  outline: 1px solid slategrey;
}
.sign-in-title {
  color: beige;
}
.about {
  padding-top: 20px;
}
.chat-title {
  color: whitesmoke;
  font-size: 1.13rem;
  margin-top: 5px
}
.socket-message {
  margin: 10px 0;
  padding: 9px;
  background: cadetblue;
  color: white;
  border-radius: 4px 0px 12px 4px;
  margin-left: 12px;
  position: relative;
  font-family: sans-serif;
  font-size: 0.83rem;
}
.socket-message.socket-self-message {
  background: darkseagreen;
  border-radius: 12px 8px 8px 0;
  margin: 12px 12px 12px 0px;
}
.repl-msg {
  font-size: 0.8rem;
  color: lightgoldenrodyellow;
}
.ansr-msg {
  padding-top: 3px;
  font-size: 0.83rem;
}
.socket-block {
  border-top: 1px solid #355455;
  text-align: left;
  padding: 5px 8px;
  background: slategray;
  overflow: auto;
  height: 300px;
  border-radius: 5px 5px 0 0;
  box-shadow: inset 1px 0 1px #355455;
}
.input-wrapper {

}
.action-wrapper {
  padding: 10px 0 3px 0;
}
.card {
  box-shadow: 0px 3px 5px -1px rgb(0 0 0 / 20%), 
    0px 5px 8px 0px rgb(0 0 0 / 14%), 
    0px 1px 14px 0px rgb(0 0 0 / 12%);
  padding: 12px 7px;
  width: 320px;
  display: inline-block;
  margin: 0 auto;
  text-align: center;
  border-radius: 4px;
  background: slategrey;
}
.form-input {
  margin: 6px 0;
  width: 85%;
  line-height: 1.46rem;
  font-size: 0.83rem;
  padding: 3px 5px;
  border-radius: 3px;
  outline: none;
  border: none;
}
.form-btn {
  margin: 5px 0;
  line-height: 1.4rem;
}
textarea {
  padding: 8px 6px;
  padding: 8px;
  margin: 0;
  resize: none;
  outline: none;
  font-size: 0.83rem;
  border: none;
  flex: 1 1 auto;
  font-family: sans-serif;
  border-radius: 0 0 3px 3px;
  color: unset;
}
.message-wrapper {
  display: flex;
  align-items: center;
}
.reply-btn {
  position: absolute;
  transition: all 0.3s ease;
  top: 2px;
  right: 0;
  line-height: 0.8rem;
  border-radius: 5px 0 5px 0;
  cursor: pointer;
  border: none;
  background: transparent;
}
.reply-section {
  position: relative;
  text-align: left;
  color: white;
  background: dimgrey;
  padding: 12px;
  border-left: 3px solid;
  font-size: 0.83rem;
  border-radius: 0 11px 0 0;
}
.close-reply-btn {
  cursor: pointer;
  position: absolute;
  top: 0;
  right: 0;
  background: transparent;
  outline: none;
  border: none;
  top: 3px;
  right: 2px;
}
.sbmt-btn-wrap {
  align-self: stretch;
  background: white;
  display: flex;
  align-items: center;
  border-radius: 0 0 3px 0;
}
.msg-date-wrapper {
  font-size: 0.7rem;
  text-align: right;
  padding-top: 5px;
}
#submit-btn {
  width: 35px;
  height: 35px;
  border-radius: 50%;
  border: none;
  background: transparent;
}
    
#submit-btn svg {
  width: 25px;
  height: 25px;
  cursor: pointer;
}
</style>
