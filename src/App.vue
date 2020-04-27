<template>
  <div id="app">
    <!-- <my-video></my-video> -->
    <my-chat></my-chat>
  </div>
</template>

<script>
import MyVideo from './components/video.vue'
import MyChat from './components/chat.vue'
export default {
  name: 'App',
  components: {
    // MyVideo,
    MyChat
  },
  data() {
    return {
    }
  },
  created() {
  },
  methods:{}

}
</script>

<style lang='scss'>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
