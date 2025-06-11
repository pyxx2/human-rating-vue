<template>
  <div id="app">
    <ImageRating />
  </div>
</template>

<script>
import ImageRating from './components/ImageRating.vue'

export default {
  name: 'App',
  components: {
    ImageRating
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin: 0;
  padding: 0;
}

body {
  margin: 0;
  background-color: #f5f5f5;
}
</style>