<template>
  <div class="scene-container">
    <div ref="container" class="three-container"></div>
    <div class="control-panel">
      <div class="control-group">
        <label>Width:</label>
        <input
          type="range"
          min="30"
          max="450"
          step="1"
          v-model="cabinetWidthCm"
          @input="updateCabinetWidth"
        />
        <span>{{ cabinetWidthCm }}cm</span>
      </div>

      <div class="control-group">
        <label>Height:</label>
        <div class="button-group">
          <button
            v-for="h in heightOptions"
            :key="h"
            @click="updateHeight(h)"
            :class="{ active: cabinetHeight === h }"
          >
            {{ h / 10 }}cm
          </button>
        </div>
      </div>

      <div class="control-group">
        <label>Depth:</label>
        <div class="button-group">
          <button
            v-for="d in depthOptions"
            :key="d"
            @click="updateDepth(d)"
            :class="{ active: cabinetDepth === d }"
          >
            {{ d / 10 }}cm
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, shallowRef, computed, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// Three.js相关变量
const container = ref(null);
const cabinetWidth = ref(300); // 默认宽度：300mm
const cabinetHeight = ref(330); // 默认高度：330mm
const cabinetDepth = ref(240); // 默认深度：240mm
const heightOptions = [330, 530, 730, 1030, 1330, 1630, 1930];
const depthOptions = [240, 320, 400];

// 计算属性：将宽度从mm转换为cm
const cabinetWidthCm = computed({
  get: () => cabinetWidth.value / 10, // mm -> cm
  set: (value) => (cabinetWidth.value = value * 10), // cm -> mm
});

const scene = new THREE.Scene();
const camera = shallowRef(null);
const renderer = shallowRef(null);
let animationFrameId = null;
let controls;

// 材质
const woodMaterial = new THREE.MeshStandardMaterial();
function loadWoodTexture() {
  const textureLoader = new THREE.TextureLoader();
  woodMaterial.map = textureLoader.load('/light_cleaf_512_albedo.jpg');
  woodMaterial.bumpMap = textureLoader.load('/light_cleaf_512_bump.jpg');
  woodMaterial.roughness = 0.7;
  woodMaterial.metalness = 0.1;
}

// 初始化Three.js场景
function initThree() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;

  camera.value = new THREE.PerspectiveCamera(75, width / height, 0.1, 10000);
  camera.value.position.z = 2000;

  const gridHelper = new THREE.GridHelper(10000, 10);
  scene.add(gridHelper);

  const ambientLight = new THREE.AmbientLight(0xffffff, 1);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(1, 1, 1).normalize();
  scene.add(directionalLight);

  renderer.value = new THREE.WebGLRenderer({ antialias: true });
  renderer.value.setSize(width, height);
  container.value.appendChild(renderer.value.domElement);

  controls = new OrbitControls(camera.value, renderer.value.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.screenSpacePanning = true;

  loadWoodTexture();
}

// 清理场景中的自定义部件
function clearCustomObjects() {
  scene.children = scene.children.filter(
    (obj) => !(obj.userData?.isPanel || obj.userData?.isDivider)
  );
}

// 生成水平板件（底板、顶板、侧板）
function generateHorizontalPanels() {
  // 生成底板
  const bottomGeometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
  const bottomMesh = new THREE.Mesh(bottomGeometry, woodMaterial);
  bottomMesh.position.set(0, 10, 0);
  bottomMesh.userData.isPanel = true;
  scene.add(bottomMesh);

  // 生成各层结构
  const currentHeightIndex = heightOptions.indexOf(cabinetHeight.value);
  for (let i = 0; i <= currentHeightIndex; i++) {
    const topHeight = heightOptions[i];

    // 顶板
    const topGeometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
    const topMesh = new THREE.Mesh(topGeometry, woodMaterial);
    topMesh.position.set(0, topHeight - 10, 0);
    topMesh.userData.isPanel = true;
    scene.add(topMesh);

    // 侧板（左右各一个）
    const sideHeight = topHeight - (i > 0 ? heightOptions[i - 1] : 0) - 20;
    const sideGeometry = new THREE.BoxGeometry(20, sideHeight, cabinetDepth.value);

    const leftPanel = new THREE.Mesh(sideGeometry, woodMaterial);
    leftPanel.position.set(-cabinetWidth.value / 2 + 10, (topHeight + (i > 0 ? heightOptions[i - 1] : 0)) / 2 - 10, 0);
    leftPanel.userData.isPanel = true;
    scene.add(leftPanel);

    const rightPanel = new THREE.Mesh(sideGeometry, woodMaterial);
    rightPanel.position.set(cabinetWidth.value / 2 - 10, (topHeight + (i > 0 ? heightOptions[i - 1] : 0)) / 2 - 10, 0);
    rightPanel.userData.isPanel = true;
    scene.add(rightPanel);
  }
}

// 生成垂直隔断
function generateVerticalDividers() {
  const sectionWidth = 300; // 每个窗口宽度的临界值 (300mm = 30cm)
  const sections = Math.max(1, Math.floor(cabinetWidth.value / sectionWidth)); // 确保至少1个隔间
  const actualSectionWidth = cabinetWidth.value / sections;

  for (let i = 0; i < sections - 1; i++) {
    const xOffset = -cabinetWidth.value / 2 + (i + 1) * actualSectionWidth;
    for (let j = 0; j < heightOptions.length && heightOptions[j] <= cabinetHeight.value; j++) {
      const bottomHeight = j === 0 ? 0 : heightOptions[j - 1];
      const topHeight = heightOptions[j];
      const dividerHeight = topHeight - bottomHeight - 20;
      const dividerGeometry = new THREE.BoxGeometry(20, dividerHeight, cabinetDepth.value);
      const dividerMesh = new THREE.Mesh(dividerGeometry, woodMaterial);
      dividerMesh.position.set(xOffset, (bottomHeight + topHeight) / 2 - 10, 0);
      dividerMesh.userData.isDivider = true;
      scene.add(dividerMesh);
    }
  }
}

// 更新所有结构
function updateAllStructures() {
  clearCustomObjects();
  generateHorizontalPanels();
  generateVerticalDividers();
}

// 动态更新宽度
const updateCabinetWidth = () => {
  updateAllStructures();
};

// 动态更新高度
const updateHeight = (newHeight) => {
  cabinetHeight.value = newHeight;
  updateAllStructures();
};

// 动态更新深度
const updateDepth = (newDepth) => {
  cabinetDepth.value = newDepth;
  updateAllStructures();
};

// 动画和生命周期
function animate() {
  controls.update();
  animationFrameId = requestAnimationFrame(animate);
  renderer.value.render(scene, camera.value);
}

function onWindowResize() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  camera.value.aspect = width / height;
  camera.value.updateProjectionMatrix();
  renderer.value.setSize(width, height);
}

onMounted(() => {
  initThree();
  updateAllStructures(); // 初始化默认结构
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
}

.control-panel {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background: rgba(255, 255, 255, 0.9);
  padding: 15px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(4px);
  z-index: 1;
}

.control-group {
  margin-bottom: 15px;
}

.control-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #333;
}

.button-group button {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: #f8f9fa;
  cursor: pointer;
  font-size: 14px;
}

.button-group button:hover {
  background: #e9ecef;
}

.button-group button.active {
  background: #007AFF;
  color: white;
  border-color: #007AFF;
}

input[type="range"] {
  width: 200px;
  margin-right: 10px;
  accent-color: #007AFF;
}
</style>