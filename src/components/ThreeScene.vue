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
        >
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
const cabinetWidth = ref(1000); // 内部单位：mm
const cabinetHeight = ref(1030);
const cabinetDepth = ref(240);
const heightOptions = [330,530,730,1030,1330,1630,1930];
const depthOptions = [240,320,400];

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

// 引用板对象
const bottomMesh = shallowRef(null);
const topMesh = shallowRef(null);
const leftPanelMesh = shallowRef(null);
const rightPanelMesh = shallowRef(null);

// 初始化Three.js场景
function initThree() {
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  
  camera.value = new THREE.PerspectiveCamera(75, width / height, 0.1, 100000);
  camera.value.position.z = 5000;

  const gridHelper = new THREE.GridHelper(10000, 10);
  scene.add(gridHelper);

  const ambientLight = new THREE.AmbientLight(0xffffff, 1);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(1, 1, 1).normalize();
  scene.add(directionalLight);

  const pointLight = new THREE.PointLight(0xffffff, 10, 2000);
  pointLight.position.set(500, 1000, 500);
  scene.add(pointLight);

  renderer.value = new THREE.WebGLRenderer({ antialias: true });
  renderer.value.setSize(width, height);
  container.value.appendChild(renderer.value.domElement);

  controls = new OrbitControls(camera.value, renderer.value.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.screenSpacePanning = true;

  // 材质
  const woodMaterial = new THREE.MeshStandardMaterial({
    color: 0xffffff,
    roughness: 0.7,
    metalness: 0.1,
    bumpScale: 0.05
  });

  const textureLoader = new THREE.TextureLoader();
  const woodTexture = textureLoader.load('/light_cleaf_512_albedo.jpg');
  const woodBumpTexture = textureLoader.load('/light_cleaf_512_bump.jpg');
  woodMaterial.map = woodTexture;
  woodMaterial.bumpMap = woodBumpTexture;

  // 创建组件
  const createBottomMesh = () => {
    const geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
    bottomMesh.value = new THREE.Mesh(geometry, woodMaterial);
    bottomMesh.value.position.set(0, 10, 0);
    scene.add(bottomMesh.value);
  };

  const createTopMesh = () => {
    const geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
    topMesh.value = new THREE.Mesh(geometry, woodMaterial);
    topMesh.value.position.set(0, cabinetHeight.value - 10, 0);
    scene.add(topMesh.value);
  };

  const createLeftPanelMesh = () => {
    const geometry = new THREE.BoxGeometry(20, cabinetHeight.value - 40, cabinetDepth.value);
    leftPanelMesh.value = new THREE.Mesh(geometry, woodMaterial);
    updatePanelPosition();
    scene.add(leftPanelMesh.value);
  };

  const createRightPanelMesh = () => {
    const geometry = new THREE.BoxGeometry(20, cabinetHeight.value - 40, cabinetDepth.value);
    rightPanelMesh.value = new THREE.Mesh(geometry, woodMaterial);
    updatePanelPosition();
    scene.add(rightPanelMesh.value);
  };

  createBottomMesh();
  createTopMesh();
  createLeftPanelMesh();
  createRightPanelMesh();
}

// 更新逻辑
function updatePanelPosition() {
  if (leftPanelMesh.value && rightPanelMesh.value) {
    const offset = cabinetWidth.value / 2;
    leftPanelMesh.value.position.set(-offset + 10, cabinetHeight.value / 2, 0);
    rightPanelMesh.value.position.set(offset - 10, cabinetHeight.value / 2, 0);
  }
}

const updateCabinetWidth = () => {
  if (bottomMesh.value && topMesh.value) {
    bottomMesh.value.geometry.dispose();
    bottomMesh.value.geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
    
    topMesh.value.geometry.dispose();
    topMesh.value.geometry = new THREE.BoxGeometry(cabinetWidth.value, 20, cabinetDepth.value);
    
    updatePanelPosition();
  }
};

const updateHeight = (newHeight) => {
  cabinetHeight.value = newHeight;
  
  if (topMesh.value && leftPanelMesh.value && rightPanelMesh.value) {
    // 更新顶板位置
    topMesh.value.position.y = newHeight - 10;
    
    // 更新侧板高度
    leftPanelMesh.value.geometry.dispose();
    leftPanelMesh.value.geometry = new THREE.BoxGeometry(20, newHeight - 40, cabinetDepth.value);
    
    rightPanelMesh.value.geometry.dispose();
    rightPanelMesh.value.geometry = new THREE.BoxGeometry(20, newHeight - 40, cabinetDepth.value);
    
    updatePanelPosition();
  }
};

const updateDepth = (newDepth) => {
  cabinetDepth.value = newDepth;
  
  // 更新所有板的深度
  [bottomMesh.value, topMesh.value, leftPanelMesh.value, rightPanelMesh.value].forEach(mesh => {
    if (mesh) {
      mesh.geometry.dispose();
      mesh.geometry = new THREE.BoxGeometry(
        mesh === bottomMesh.value || mesh === topMesh.value ? cabinetWidth.value : 20,
        mesh === bottomMesh.value || mesh === topMesh.value ? 20 : cabinetHeight.value - 40,
        newDepth
      );
    }
  });
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

.control-group:last-child {
  margin-bottom: 0;
}

.control-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
  color: #333;
}

.button-group {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.button-group button {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  background: #f8f9fa;
  cursor: pointer;
  transition: all 0.2s;
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