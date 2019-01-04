<template>
  <div id="app">
    <Whis v-bind="{
      src, controls, loop, muted,
      volumeBoost, playbackRate, trimSilence
    }" />
    <div class="control">
      <label>
        <input type="checkbox" v-model="volumeBoost">
        Volume Boost
      </label>
      <label>
        <input type="checkbox" v-model="trimSilence">
        Trim Silence
      </label>
      <label>
        Speed
        <input type="number" min="0.8" step="0.1" max="5" v-model.number="playbackRate">
      </label>
    </div>
  </div>
</template>

<script>
import Whis from '../lib/Whis'
import sampleMp3 from "./assets/sample.mp3";

export default {
  name: 'App',
  components: {
    Whis
  },
  data() {
    return {
      src: sampleMp3,

      volumeBoost: false,
      playbackRate: 1.0,
      trimSilence: true,

      controls: true,
      loop: true,
      muted: false
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
#app audio {
  width: 600px;
}
#app .control {
  margin: 1em 0;
}
#app .control label {
  margin: 0 1em;
}
#app .control input[type=number] {
  font-size: 1em;
  text-align: center;
}
</style>
