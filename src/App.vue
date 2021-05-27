<template>
  <div>
    <Index></Index>
  </div>
</template>

<script>
import { defineComponent } from 'vue'
import Index from './views/index.vue'

export default defineComponent({
  name: "App",
  components: {
    Index,
  },
})
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
