<template>
  <div ref="container" class="three-container"></div>
</template>

<script setup>
import { ref, shallowRef, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const container = ref(null);

// Three.js相关变量
const scene = new THREE.Scene();
const camera = shallowRef(null);
const renderer = shallowRef(null);
const cube = shallowRef(null);
let animationFrameId = null;

// 初始化Three.js场景
function initThree() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  
  // 相机
  camera.value = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
  camera.value.position.z = 5;

  // 渲染器
  renderer.value = new THREE.WebGLRenderer({ antialias: true });
  renderer.value.setSize(width, height);
  container.value.appendChild(renderer.value.domElement);

  // 立方体
  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
  const mesh = new THREE.Mesh(geometry, material);
  cube.value = mesh;
  scene.add(mesh);
}

// 动画循环
function animate() {
  animationFrameId = requestAnimationFrame(animate);
  const currentCube = cube.value;
  currentCube.rotation.x += 0.01;
  currentCube.rotation.y += 0.01;
  renderer.value.render(scene, camera.value);
}

// 窗口大小调整处理
function onWindowResize() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  camera.value.aspect = width / height;
  camera.value.updateProjectionMatrix();
  renderer.value.setSize(width, height);
}

// 生命周期钩子
onMounted(() => {
  initThree();
  animate();
  window.addEventListener('resize', onWindowResize);
});

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize);
  cancelAnimationFrame(animationFrameId);
  renderer.value.dispose();
});
</script>

<style scoped>
.three-container {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
