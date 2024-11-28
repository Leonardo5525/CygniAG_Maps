<template>
  <div id="app">
    <GoogleMap />
  </div>
</template>

<script>
import GoogleMap from './components/GoogleMap.vue'

export default {
  components: {
    GoogleMap,
  },
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  /* margin-top: 60px; */
}
</style>
