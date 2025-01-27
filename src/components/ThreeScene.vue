<template>
  <div class="scene-container">
    <div ref="container" class="three-container"></div>
    <div class="control-panel">
      <label for="width">柜子宽度 (mm):</label>
      <input 
        type="range" 
        id="width" 
        min="500" 
        max="1500" 
        step="10" 
        v-model="cabinetWidth"
        @input="updateCabinetWidth"
      >
      <span>{{ cabinetWidth }}</span>
    </div>
  </div>
</template>

<script setup>
import { ref, shallowRef, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// Three.js相关变量
const container = ref(null);
const cabinetWidth = ref(1000);
const scene = new THREE.Scene();
const camera = shallowRef(null);
const renderer = shallowRef(null);
let animationFrameId = null;
let controls;

// 引用板对象
const bottomMesh = shallowRef(null);
const topMesh = shallowRef(null);
const leftPanelMesh = shallowRef(null);
const rightPanelMesh = shallowRef(null);

// 初始化Three.js场景
function initThree() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  
  // 相机
  camera.value = new THREE.PerspectiveCamera(75, width / height, 0.1, 100000);
  camera.value.position.z = 5000;

  // 添加网格辅助
  const gridHelper = new THREE.GridHelper(10000, 10);
  scene.add(gridHelper);

  // 添加环境光
  const ambientLight = new THREE.AmbientLight(0xffffff, 1);
  scene.add(ambientLight);

  // 添加日光
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(1, 1, 1).normalize();
  scene.add(directionalLight);

  // 添加点光源
  const pointLight = new THREE.PointLight(0xffffff, 10, 2000);
  pointLight.position.set(500, 1000, 500);
  scene.add(pointLight);

  // 渲染器
  renderer.value = new THREE.WebGLRenderer({ antialias: true });
  renderer.value.setSize(width, height);
  container.value.appendChild(renderer.value.domElement);

  // 控制器
  controls = new OrbitControls(camera.value, renderer.value.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.screenSpacePanning = true;

  // 创建木纹材质
  const woodMaterial = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    roughness: 0.7,
    metalness: 0.1,
    bumpScale: 0.05
  });

  // 创建木纹纹理
  const textureLoader = new THREE.TextureLoader();
  const woodTexture = textureLoader.load('/light_cleaf_512_albedo.jpg');
  const woodBumpTexture = textureLoader.load('/light_cleaf_512_bump.jpg');
  woodMaterial.map = woodTexture;
  woodMaterial.bumpMap = woodBumpTexture;

  // 创建底板
  const createBottomMesh = () => {
    const geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, 350);
    bottomMesh.value = new THREE.Mesh(geometry, woodMaterial);
    bottomMesh.value.position.set(0, 10, 0);
    scene.add(bottomMesh.value);
  };

  // 创建顶板
  const createTopMesh = () => {
    const geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, 350);
    topMesh.value = new THREE.Mesh(geometry, woodMaterial);
    topMesh.value.position.set(0, 980, 0);
    scene.add(topMesh.value);
  };

  // 创建左侧隔板
  const createLeftPanelMesh = () => {
    const geometry = new THREE.BoxGeometry(20, 960, 350);
    leftPanelMesh.value = new THREE.Mesh(geometry, woodMaterial);
    updatePanelPosition();
    scene.add(leftPanelMesh.value);
  };

  // 创建右侧隔板
  const createRightPanelMesh = () => {
    const geometry = new THREE.BoxGeometry(20, 960, 350);
    rightPanelMesh.value = new THREE.Mesh(geometry, woodMaterial);
    updatePanelPosition();
    scene.add(rightPanelMesh.value);
  };


  // 初始化创建所有板
  createBottomMesh();
  createTopMesh();
  createLeftPanelMesh();
  createRightPanelMesh();
}

// 更新隔板位置
function updatePanelPosition() {
  if (leftPanelMesh.value && rightPanelMesh.value) {
    const offset = cabinetWidth.value / 2;
    leftPanelMesh.value.position.set(-offset - 10, 500, 0);
    rightPanelMesh.value.position.set(offset + 10, 500, 0);
  }
}

// 更新柜子宽度
const updateCabinetWidth = () => {
  if (bottomMesh.value && topMesh.value) {
    // 更新底板和顶板几何体
    bottomMesh.value.geometry.dispose();
    bottomMesh.value.geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, 350);
    
    topMesh.value.geometry.dispose();
    topMesh.value.geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, 350);
    
    // 更新隔板位置
    updatePanelPosition();
  }
};

// 动画循环
function animate() {
  controls.update();
  animationFrameId = requestAnimationFrame(animate);
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
.scene-container {
  position: relative;
  width: 100%;
  height: 100%;
}

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

.control-panel {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background: rgba(255, 255, 255, 0.8);
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  z-index: 1;
}

.control-panel label {
  margin-right: 10px;
}

.control-panel input[type="range"] {
  width: 200px;
  margin-right: 10px;
}
</style>
