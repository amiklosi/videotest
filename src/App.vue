<template>
  <div id="app">
    <VideoRecorder />
  </div>
</template>

<script>
import VideoRecorder from "./components/VideoRecorder";

export default {
  name: "App",
  components: {
    VideoRecorder,
  },
  data() {
    return {
      videoUrl: null,
    };
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>

<style lang="scss">
</style>
