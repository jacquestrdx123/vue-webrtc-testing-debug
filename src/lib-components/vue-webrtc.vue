<template>
  <div class="video-list">
    <div v-for="item in videoList"
         v-bind:video="item"
         v-bind:key="item.id"
         class="video-item">
      <video controls autoplay playsinline ref="videos" :height="cameraHeight" :muted="item.muted" :id="item.id"></video>
    </div>
  </div>
</template>
<script>
import { defineComponent } from 'vue';
import { io } from "socket.io-client";
const SimpleSignalClient = require('simple-signal-client');

export default /*#__PURE__*/defineComponent({
  name: 'vue-webrtc',
  data() {
    return {
      signalClient: null,
      audioContext: null,
      gainNode: null,
      ctx: null,
      videoList: [],
      canvas: null,
      socket: null,
      localVideoStream: null,
      localAudioStream: null,
    };
  },
  props: {
    roomId: {
      type: String,
      default: 'public-room-v2'
    },
    socketURL: {
      type: String,
      default: 'https://weston-vue-webrtc-lobby.azurewebsites.net'
    },
    cameraHeight: {
      type: [Number, String],
      default: 160
    },
    autoplay: {
      type: Boolean,
      default: true
    },
    screenshotFormat: {
      type: String,
      default: 'image/jpeg'
    },
    enableAudio: {
      type: Boolean,
      default: true
    },
    enableVideo: {
      type: Boolean,
      default: true
    },
    enableLogs: {
      type: Boolean,
      default: false
    },
    peerOptions: {
      type: Object,
      default() {
        return {};
      }
    },
    ioOptions: {
      type: Object,
      default() {
        return { rejectUnauthorized: false, transports: ['polling', 'websocket'] };
      }
    },
    deviceId: {
      type: String,
      default: null
    }
  },
  mounted() {
    this.join();
  },
  methods: {
    async join() {
      if (this.joined) return; // Prevent joining multiple times

      this.socket = io(this.socketURL, this.ioOptions);
      this.signalClient = new SimpleSignalClient(this.socket);

      const videoConstraints = {
        video: this.enableVideo,
        audio: false
      };
      if (this.deviceId && this.enableVideo) {
        videoConstraints.video = { deviceId: { exact: this.deviceId } };
      }
      const audioConstraints = {
        video: false,
        audio: this.enableAudio
      };

      this.localVideoStream = await navigator.mediaDevices.getUserMedia(videoConstraints);
      this.localAudioStream = await navigator.mediaDevices.getUserMedia(audioConstraints);

      // Initialize the audio context and manipulate the audio stream
      this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
      try {
        const source = this.audioContext.createMediaStreamSource(this.localAudioStream);
        this.gainNode = this.audioContext.createGain();
        this.gainNode.gain.value = 0.5;
        source.connect(this.gainNode);
        const destination = this.audioContext.createMediaStreamDestination();
        this.gainNode.connect(destination);

        const newAudioTracks = destination.stream.getAudioTracks();
        this.localAudioStream.getAudioTracks().forEach((track) => this.localAudioStream.removeTrack(track));
        newAudioTracks.forEach((track) => this.localAudioStream.addTrack(track));
      } catch (e) {
        this.log(e);
      }

      this.joinedRoom(this.localVideoStream, true, 'video');
      this.joinedRoom(this.localAudioStream, true, 'audio');

      this.signalClient.once('discover', (discoveryData) => {
        discoveryData.peers.forEach((peerID) => this.connectToPeer(peerID));
        this.$emit('opened-room', this.roomId);
      });

      this.signalClient.on('request', async (request) => {
        const { peer } = await request.accept({}, this.peerOptions);
        this.onPeer(peer);
      });

      this.signalClient.discover(this.roomId);
      this.joined = true; // Set joined to true after setting up everything
    },
    async connectToPeer(peerID) {
      if (peerID === this.socket.id) return;
      try {
        const { peer } = await this.signalClient.connect(peerID, this.roomId, this.peerOptions);
        this.onPeer(peer);
      } catch (e) {
        this.log('Error connecting to peer', e);
      }
    },
    onPeer(peer) {
      peer.addStream(this.localVideoStream);
      peer.addStream(this.localAudioStream);

      peer.on('stream', (remoteStream) => {
        const streamType = remoteStream.getVideoTracks().length > 0 ? 'video' : 'audio';
        this.joinedRoom(remoteStream, false, streamType);

        peer.on('close', () => {
          this.videoList = this.videoList.filter(item => item.id !== remoteStream.id);
          this.$emit('left-room', remoteStream.id);
        });

        peer.on('error', (err) => {
          this.log('peer error', err);
        });
      });
    },
    joinedRoom(stream, isLocal, streamType) {
      if (!this.videoList.find(video => video.id === stream.id)) {
        this.videoList.push({ id: stream.id, muted: isLocal, stream, isLocal, streamType });
      }

      setTimeout(() => {
        for (let i = 0, len = this.$refs.videos.length; i < len; i++) {
          if (this.$refs.videos[i].id === stream.id) {
            this.$refs.videos[i].srcObject = stream;
            break;
          }
        }
      }, 500);

      this.$emit('joined-room', stream.id);
    },
    leave() {
      this.videoList.forEach(v => v.stream.getTracks().forEach(t => t.stop()));
      this.videoList = [];
      this.signalClient.peers().forEach(peer => peer.removeAllListeners());
      this.signalClient.destroy();
      this.signalClient = null;
      this.socket.destroy();
      this.socket = null;
    },
    capture() {
      return this.getCanvas().toDataURL(this.screenshotFormat);
    },
    getCanvas() {
      let video = this.$refs.videos[0];
      if (video !== null && !this.ctx) {
        let canvas = document.createElement('canvas');
        canvas.height = video.clientHeight;
        canvas.width = video.clientWidth;
        this.canvas = canvas;
        this.ctx = canvas.getContext('2d');
      }
      this.ctx.drawImage(video, 0, 0, this.canvas.width, this.canvas.height);
      return this.canvas;
    },
    async shareScreen() {
      if (!navigator.mediaDevices) {
        this.log('Error: https is required to load cameras');
        return;
      }
      try {
        const screenStream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: false });
        this.joinedRoom(screenStream, true, 'video');
        this.signalClient.peers().forEach(p => this.onPeer(p, screenStream));
        this.$emit('share-started', screenStream.id);
      } catch (e) {
        this.log('Media error', e);
      }
    },
    log(message, data) {
      if (this.enableLogs) {
        console.log(message);
        if (data != null) {
          console.log(data);
        }
      }
    }
  }
});
</script>

<style scoped>
.video-list {
  background: whitesmoke;
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
  justify-items: center;
  padding: 20px;
}

.video-item {
  background: #c5c4c4;
  width: 100%;
  aspect-ratio: 16 / 9;
  display: flex;
  align-items: center;
  justify-content: center;
  text-align: center;
  border: 1px solid #aaa;
}

video {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
</style>
