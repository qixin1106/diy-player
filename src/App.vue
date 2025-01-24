<template>
  <div id="app">
    <ThreeScene />
  </div>
</template>

<script setup>
import ThreeScene from './components/ThreeScene.vue'
</script>

<style>
html, body {
  margin: 0;
  padding: 0;
  overflow: hidden;
  width: 100%;
  height: 100%;
}

#app {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
