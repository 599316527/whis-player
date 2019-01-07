<template>
  <audio ref="audio" v-bind="{controls, loop, muted}"
    @play="handleAudioPlay"
    @playing="handleAudioPlaying"
    @timeupdate="handleAudioTimeUpdate">
    <source v-for="({src, type}) in realSrc" :key="src" :src="src" :type="type" >
  </audio>
</template>

<script>
export default {
  name: 'Whis',

  props: {
    src: [String, Array],

    volumeBoost: Boolean,
    trimSilence: Boolean,

    controls: Boolean,
    loop: Boolean,
    muted: Boolean,

    playbackRate: Number
  },

  data() {
    return {
      delayInterval: 0.1,
      sampleRate: 44100,
      fftSize: Math.pow(2, 10)
    }
  },

  computed: {
    realSrc() {
      return [].concat(this.src).map(function (src) {
        return typeof src === 'string' ? {
          src,
          type: `audio/${src.substring(src.lastIndexOf('.') + 1)}`
        } : src
      })
    }
  },

  watch: {
    src(val) {
      this.sourceNode = null
    },
    muted(val) {
      this.$refs.audio.muted = val
    },
    controls(val) {
      this.$refs.audio.controls = val
    },
    loop(val) {
      this.$refs.audio.loop = val
    },
    playbackRate(val) {
      this.$refs.audio.playbackRate = val
    },
    volumeBoost(val) {
      this.updateAudioContext()
    },
    trimSilence(val) {
      this.updateAudioContext()
    }
  },

  methods: {
    handleAudioPlay() {
      if (this.audioContext.state === 'suspended') {
        this.audioContext.resume();
      }
    },
    handleAudioPlaying() {
      this.updateAudioContext()
    },
    handleAudioTimeUpdate() {
      if (this.seekFlag) {
        this.seekFlag = false
        return;
      }
      if (!this.trimSilence) {
        return;
      }
      let currentTime = this.$refs.audio.currentTime
      this.analyserNode.getByteFrequencyData(this.analyserDataArray);
      let bytes = this.analyserDataArray.slice(this.freqRangeStartIndex, this.freqRangeEndIndex + 1)
      // console.log(bytes)
      let minBytes = Math.min(...bytes)
      if (
          // TODO: more complex VAD algorithm
          bytes.map(byte => byte - minBytes)
              .every(function (val) {
                  return val < 16
              })
      ) {
          this.seekFlag = true;
          this.$nextTick(() => {
            this.$refs.audio.currentTime += this.delayInterval
            console.log('Skip: ', currentTime, '+' + this.delayInterval)
          })
      }
    },

    updateAudioContext() {
      if (!this.sourceNode) {
        this.sourceNode = this.audioContext.createMediaElementSource(this.$refs.audio)
      }

      let nodes = [this.sourceNode, ...this.audioMiddleNodes, this.audioContext.destination]
      for (let i = 0; i < nodes.length; i++) {
        for (let j = 0; j < nodes.length; j++) {
          if (i !== j && nodes[i].numberOfOutputs && nodes[j].numberOfInputs) {
            try {
              // There is no way to detect is a node is connected to another
              // Just try to disconnect
              nodes[i].disconnect(nodes[j])
            }
            catch (err) {
              // skip
            }
          }
        }
      }

      nodes = [this.sourceNode]
      if (this.trimSilence) {
        nodes.push(this.analyserNode)
      }
      if (this.volumeBoost) {
        nodes.push(this.warmthFilter)
        nodes.push(this.clarityFilter)
      }
      if (this.trimSilence) {
        nodes.push(this.delayNode)
      }
      nodes.push(this.audioContext.destination)

      for (let i = 0; i < nodes.length - 1; i++) {
        nodes[i].connect(nodes[i + 1])
      }

    },

    initAudioContext() {
      this.$refs.audio.playbackRate = this.playbackRate
      this.audioContext = new AudioContext();

      this.delayNode = this.audioContext.createDelay()
      this.delayNode.delayTime.value = this.delayInterval

      this.warmthFilter = this.audioContext.createBiquadFilter()
      this.warmthFilter.type = 'peaking'
      this.warmthFilter.frequency.value = 400
      this.warmthFilter.gain.value = 4
      this.warmthFilter.Q.value = 2

      this.clarityFilter = this.audioContext.createBiquadFilter()
      this.clarityFilter.type = 'peaking'
      this.clarityFilter.frequency.value = 2000
      this.clarityFilter.gain.value = 12
      this.clarityFilter.Q.value = 2

      this.analyserNode = this.audioContext.createAnalyser();
      this.analyserNode.fftSize = this.fftSize
      this.analyserNode.minDecibels = -90;
      this.analyserNode.maxDecibels = -30;
      let bufferLength = this.analyserNode.frequencyBinCount
      this.analyserDataArray = new Uint8Array(bufferLength)
      this.freqRangeStartIndex = Math.ceil(110 / (this.sampleRate / bufferLength))
      this.freqRangeEndIndex = Math.floor(440 / (this.sampleRate / bufferLength))

      this.audioMiddleNodes = [
        this.analyserNode,
        this.warmthFilter,
        this.clarityFilter,
        this.delayNode
      ]
    }
  },

  mounted() {
    this.seekFlag = true
    this.initAudioContext()
  }

}
</script>

