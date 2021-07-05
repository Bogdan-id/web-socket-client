<template>
  <div class="about">
    <!-- Sign-in card -->
    <div 
      v-if="!register"
      class="card">
      <h4 class="sign-in-title">Укажите имя пользователя</h4>
      <div class="input-wrapper">
        <input 
          v-model="user"
          class="form-input"
          lable="текст"
          placeholder="Имя пользователя..."
          type="text"/>
      </div>
      <div class="action-wrapper">
        <button 
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
      <h1 class="chat-title">
        Веб-сокет: {{ user }}
      </h1>
      <div class="socket-block">
        <div 
          v-for="(user, k) in messages"
          :key="k"
          class="socket-message"
          @mouseenter="showButtons = k"
          @mouseleave="showButtons = null"
          :class="{ 'socket-self-message': key === user.key }">
          <button 
            v-if="showButtons === k && key !== user.key"
            @click="activateReply(user)"
            class="reply-btn">
            <Reply />
          </button>
          <button 
            v-if="showButtons === k && key !== user.key"
            @click="createPeerConnection(user.key)"
            class="call-btn">
            <Call />
          </button>
          <!-- simple message -->
          <div v-if="!user.toMessage">
            <span>{{ user.name + ": " + user.message }}</span>
            <div class="msg-date-wrapper">
              <span>{{ user.dateTime }}</span>
            </div>
          </div>
          <!-- reply message -->
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
      <!-- reply section -->
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
      <!-- text-area -->
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
        <!-- submit button -->
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
    <!-- Video window -->
    <div class="camerabox">
      <div 
        v-show="showReceived" 
        class="received-video-wrapper">
        <div 
          @click="
            closeVideoCall(), 
            closeRemotePeer()"
          class="video-close-btn">
          &#10005;
        </div>
        <video 
          class="received-video" 
          id="received_video" 
          autoplay>
        </video>
      </div>
      <!-- <video id="local_video" autoplay muted></video> -->
      <!-- <button id="hangup-button" @click="hangUpCall()" role="button" disabled>
        Hang Up
      </button> -->
    </div>
  </div>
</template>
<script>
// @ts-check
import { defineComponent, ref } from 'vue'
import SendIcon from '@/assets/send.vue'
import CloseIcon from '@/assets/close.vue'
import Reply from '@/assets/reply.vue'
import Call from '@/assets/call.vue'
export default defineComponent({
  name: 'sign-in',
  components: { SendIcon, CloseIcon, Reply, Call },
  setup() {
    const socket = connectSocket()

    const user = ref('')
    const message = ref('')
    const replyMessage = ref('')
    const messages = ref([])
    const register = ref(false)
    const replyState = ref(false)
    const showButtons = ref('')
    const showReceived = ref('')
    const key = ref('')
    const keyTo = ref('')
    // const baseURL = '127.0.0.1:8080'
    let pc

    function toggleView() {
      const el = document.querySelector('.socket-block')
      if (!el) return
      el.scrollTo({
        top: el.scrollHeight,
        left: 0,
      })
    }

    function submit() {
      if (message.value.trim()) {
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
          meta: 'broadcast-msg'
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
    
    /** @param { Object } user - user from message */
    const activateReply = (user) => {
      replyState.value = true
      replyMessage.value = user.name + ": " + user.message 
      keyTo.value = user.key
      document.getElementById('area').focus()
    }

    function connectSocket() {
      const socket = new WebSocket('wss://obscure-anchorage-43966.herokuapp.com/') // wss://obscure-anchorage-43966.herokuapp.com/, ws://127.0.0.1:8000

      socket.onopen = function(e) {
        console.log('connection opened', e)
        socket.send(JSON.stringify({
            room: 'chat-room', 
            meta: 'join'
          })
        )
      }

      socket.onmessage = async function(event) {
        try {
          const message = JSON.parse(event.data)
          const AUTHORIZE = message?.meta === 'authorize',
            INIT_MESSAGE = message?.meta === 'init-messages',
            BROADCASE_MSG = message?.meta === 'broadcast-msg',
            VIDEO_OFFER = message?.meta === 'video-offer',
            VIDEO_ANSWER = message?.meta === 'video-answer',
            NEW_ICE_CANDIDATE = message?.meta === 'new-ice-candidate',
            CLOSE_VIDEO_CALL = message?.meta === 'close-video-call'

          setTimeout(() => { toggleView() }, 0)

          switch (true) {
            case AUTHORIZE: {
              console.log('AUTHORIZE')
              return key.value = message?.authorize
            }
            case INIT_MESSAGE: {
              console.log('INIT_MESSAGE')
              if (Array.isArray(message?.messages)) {
                messages.value.push(...message?.messages)
                return
              } else return console.log('Not')
            }
            case BROADCASE_MSG: {
              console.log('BROADCASE_MSG')
              if (Array.isArray(message)) {
                return messages.value.push(...message)
              } else { 
                console.log('MESSAGE', message)
                return messages.value.push(message) 
              }
            }
            case VIDEO_OFFER: {
              console.log('VIDEO_OFFER')
              if (!pc) createPeerConnection(message.target, message.desc)
              keyTo.value = message.caller
              await pc.setRemoteDescription(message.desc)
              const stream = await navigator.mediaDevices.getUserMedia(constraints)
              stream.getTracks().forEach((track) => pc.addTrack(track, stream))
              await pc.setLocalDescription(await pc.createAnswer())
              getUserMedia()
              return socket.send(JSON.stringify({ 
                desc: pc.localDescription,
                room: 'chat-room',
                target: keyTo.value,
                meta: 'video-answer',
              }))
            }
            case VIDEO_ANSWER: {
              console.log('VIDEO_ANSWER', message.meta)
              return await pc.setRemoteDescription(message.desc)
            }
            case NEW_ICE_CANDIDATE: {
              console.log('NEW_ICE_CANDIDATE', message) // Interactive Connectivity Establishment
              if (!pc) pc = new RTCPeerConnection(configuration)
              const candidate = new RTCIceCandidate(message.candidate)
              try {
                return await pc.addIceCandidate(candidate)
              } catch(err) {
                return console.log(err)
              }
            }
            case CLOSE_VIDEO_CALL: {
              return closeVideoCall()
            }
            default: {
              return console.log('Unsuported meta descriptor')
            }
          }
        } catch (err) {
          console.log(err)
        }
      }

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

    /* WebRTC */
    // const signaling = new SignalingChannel();
    const constraints = { audio: true, video: true }
    const configuration = { 
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        { urls: 'stun:stun1.l.google.com:19302' },
        { urls: 'stun:stun2.l.google.com:19302' },
        { urls: 'stun:stun3.l.google.com:19302' },
        { urls: 'stun:stun4.l.google.com:19302' },
      ] 
    }

    async function createPeerConnection(targetKey, desc) {
      keyTo.value = targetKey
      if (!pc) pc = new RTCPeerConnection(configuration)

      pc.onicecandidate = ({ candidate }) => {
        if(!candidate) return
        console.log('onicecandidate', candidate)
        socket.send(JSON.stringify({ 
          candidate: candidate, 
          meta: 'new-ice-candidate',
          target: keyTo.value,
          userId: key.value,
          room: 'chat-room',
        }))
      }

      pc.onnegotiationneeded = async () => {
        console.log('onnegotiationneeded')
        if (desc) return
        try {
          await pc.setLocalDescription(await pc.createOffer())
          socket.send(JSON.stringify({ 
            desc: pc.localDescription,
            meta: 'video-offer',
            room: 'chat-room',
            target: keyTo.value,
            caller: key.value,
          }))
        } catch (err) {
          console.error(err)
        }
      }

      pc.ontrack = (event) => {
        console.log('ontrack', event)
        document.getElementById('received_video').srcObject = event.streams[0]
        showReceived.value = true
      }

      getUserMedia()
    }

    async function getUserMedia() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia(constraints)
        stream.getTracks().forEach((track) => pc.addTrack(track, stream))
        // const el = document.getElementById('local_video').srcObject = stream
        // el.volume = 0
        return 
      } catch (err) {
        console.error(err)
      }
    }

    function closeVideoCall() {
      if (pc) {
        pc.ontrack = null
        pc.onnicecandidate = null
        pc.onnotificationneeded = null;
        pc.getTransceivers().forEach(transceiver => { transceiver.stop() })
        pc.close()
        pc = null
        showReceived.value = false
      }
    }

    function closeRemotePeer() {
      socket.send(JSON.stringify({
        room: 'chat-room',
        target: keyTo.value,
        meta: 'close-video-call',
      }))
    }

    return {
      message,
      messages,
      user,
      register,
      showButtons,
      replyState,
      replyMessage,
      key,
      showReceived,
      submitUser,
      activateReply,
      submit,
      createPeerConnection,
      closeVideoCall,
      closeRemotePeer,
    }
  }
})
</script>
<style>
body {
  font-family: sans-serif;
  height: 100vh;
  background: linear-gradient(45deg, black, transparent);
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
  font-size: 1rem;
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
  background: linear-gradient(45deg, black, transparent);
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
  background: steelblue;
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
.call-btn {
  position: absolute;
  transition: all 0.3s ease;
  top: 2px;
  right: 22px;
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

.received-video-wrapper {
  color: white;
  text-align: right;
  position: absolute;
  top: 0;
  right: 0;
  background: tomato;
}
.received-video {
  display: block;
  width: 250px;
}
.video-close-btn {
  content: '&#10005;';
  padding-right: 4px;
  cursor: pointer;
}
</style>
