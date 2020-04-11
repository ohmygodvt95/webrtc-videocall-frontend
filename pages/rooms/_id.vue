<template>
  <div id="mainFrame">
    <video id="localVideo" ref="localVideo" autoplay muted>LocalVideo</video>
    <video id="remoteVideo" ref="remoteVideo" autoplay>RemoteVideo</video>
    <div class="bottom-bar d-flex justify-center">
      <v-btn class="mx-2" fab @click="offCamera()">
        <v-icon dark>
          mdi-camera
        </v-icon>
      </v-btn>
      <v-btn class="mx-2" fab>
        <v-icon dark>
          mdi-microphone
        </v-icon>
      </v-btn>
    </div>
  </div>
</template>

<style lang="scss">
  #mainFrame {
    #localVideo {
      z-index: 100;
      position: absolute;
      right: 25px;
      bottom: 25px;
      background-color: #47494e;
      height: 150px;
      width: 200px;
    }

    #remoteVideo {
      z-index: 50;
      height: calc(100vh - 64px);
      width: 100vw;
      background-color: #7f828b;
    }

    .bottom-bar {
      position: absolute;
      bottom: 25px;
      width: 100vw;
      text-align: center;
    }
  }
</style>

<script>
const servers = {
  configuration: {
    offerToReceiveAudio: true,
    offerToReceiveVideo: true
  },
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' }
  ]
}
const RPC = window.RTCPeerConnection || window.mozRTCPeerConnection || window.webkitRTCPeerConnection
const localPC = new RPC(servers)

export default {
  data () {
    return {
      room: this.$route.params.id
    }
  },
  async beforeMount () {
    const lastId = localStorage.getItem('lastId')
    if (lastId) {
      await this.$socket.emit('leave', this.room)
    }
    await this.$socket.emit('join', lastId)
    localStorage.setItem('lastId', this.room)
  },
  async mounted () {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true })

    this.$refs.localVideo.srcObject = stream
    this.$store.commit('setting/setCamera', {
      camera: stream
    })

    stream.getTracks().forEach((track) => {
      localPC.addTrack(track, stream)
    })

    const offer = await localPC.createOffer()
    await localPC.setLocalDescription(offer)
    await this.$socket.emit('message', JSON.stringify({
      room: this.room,
      data: localPC.localDescription
    }))

    localPC.onicecandidate = async (event) => {
      if (event.candidate) {
        await this.$socket.emit('message', JSON.stringify({
          room: this.room,
          data: event.candidate
        }))
      } else {
        // eslint-disable-next-line
        console.log('allhasbeensent')
      }
    }

    localPC.ontrack = (event) => {
      if (event.streams[0]) {
        this.$refs.remoteVideo.srcObject = event.streams[0]
      }
    }

    this.$socket.on('message', async (data) => {
      if (data.type === 'offer') {
        await localPC.setRemoteDescription(new RTCSessionDescription(data))
        const answer = await localPC.createAnswer()
        await localPC.setLocalDescription(answer)
        await this.$socket.emit('message', JSON.stringify({
          room: this.room,
          data: localPC.localDescription
        }))
      } else if (data.type === 'answer') {
        await localPC.setRemoteDescription(new RTCSessionDescription(data))
      } else {
        await localPC.addIceCandidate(new RTCIceCandidate(data))
      }
    })
  },
  async beforeDestroy () {
    await this.$socket.emit('leave', this.room)
  },
  methods: {
    offCamera () {
      this.$store.state.setting.camera.getVideoTracks().forEach((track) => {
        track.enabled = !track.enabled
      })
    }
  }
}
</script>
