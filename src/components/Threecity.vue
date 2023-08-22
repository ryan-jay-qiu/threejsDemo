<template>
  <canvas ref="canvas"></canvas>
  <el-progress class="process" v-if="process !== 101" :text-inside="true" :stroke-width="26" :percentage="process" />
</template>

<script setup>
import { onMounted, ref, watch } from 'vue'
import { ElProgress } from 'element-plus'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import * as THREE from 'three'
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { ShaderPass } from 'three/examples/jsm/postprocessing/ShaderPass.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
const canvas = ref()

const ENTIRE_SCENE = 0, BLOOM_SCENE = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_SCENE);
const darkMaterial = new THREE.MeshBasicMaterial({ color: "black" });


let camera, controls, renderer, scene, composer,  labelRenderer, loader, bloomPass, t1, t2, finalComposer

const process = ref(0)
watch(process, (newV) => {
  if (newV == 100) setTimeout(() => process.value = 101, 1000)
})
onMounted(() => {
  creatMesh()
  creatPass()
  init()
  initOrbitControls()
  creatAxesHelper()
  render()
  resize()
})

function creatAxesHelper() {
  const axesHelper = new THREE.AxesHelper(1500);
  // axesHelper.layers.enable(0);
  scene.add(axesHelper);
}
function creatPass() {
  const renderScene = new RenderPass(scene, camera);
  bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
  bloomPass.threshold = 0;
  bloomPass.strength = 0.5;
  bloomPass.radius = 0;
  composer = new EffectComposer(renderer);
  composer.renderToScreen = false;
  composer.addPass(renderScene);
  composer.addPass(bloomPass);
  const finalPass = new ShaderPass(
    new THREE.ShaderMaterial({
      uniforms: {
        baseTexture: { value: null },
        bloomTexture: { value: composer.renderTarget2.texture }
      },
      vertexShader: `
				varying vec2 vUv;
			void main() {
				vUv = uv;
				gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
			}
				`,
      fragmentShader: `
				uniform sampler2D baseTexture;
			uniform sampler2D bloomTexture;
			varying vec2 vUv;
			void main() {
				gl_FragColor = ( texture2D( baseTexture, vUv ) + vec4( 1.0 ) * texture2D( bloomTexture, vUv ) );
			}
				`,
      defines: {}
    }), "baseTexture"
  );
  finalPass.needsSwap = true;
  finalComposer = new EffectComposer(renderer);
  finalComposer.addPass(renderScene);
  finalComposer.addPass(finalPass);
}


function creatMesh() {
  // 使用 dracoLoader 加载用blender压缩过的模型
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('./gltf/');//设置解压库文件路径
  dracoLoader.setDecoderConfig({ type: 'js' })  //使用js方式解压
  dracoLoader.preload()  //初始化_initDecoder 解码器


  loader = new GLTFLoader();
  loader.setDRACOLoader(dracoLoader);
  THREE.Cache.enabled = true
  const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
  }

  // renderer.setPixelRatio(window.devicePixelRatio);
  renderer = new THREE.WebGLRenderer({
    canvas: canvas.value,
    antialias: true
  });
  renderer.shadowMap.enabled = true;
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(sizes.width, sizes.height);
  renderer.autoClear = false;
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(45, sizes.width / sizes.height, .1, 100000);
  scene.add(camera);
}

function init() {
  loader.load('./gltf/lowpoly.glb', glb => {
    const { scene: mesh } = glb
    mesh.traverse(function (obj) {
        if (obj.name == '月球') {
          obj.material = new THREE.MeshPhongMaterial({
            color: 0xffff00, 
            emissive: 0xffff00, // 设置自发光颜色
            emissiveIntensity: 2, // 自发光强度，默认为 1.0
          });
          // obj.renderOrder=99;
          obj.layers.enable(BLOOM_SCENE);
        } else {
          obj.layers.set(0)
        }
    });
    mesh.layers.set(0)
    t1 = mesh.getObjectByName('锥体左')
    t2 = mesh.getObjectByName('锥体右')
    mesh.scale.set(10, 10, 10)
    camera.position.set(0, 100, 200);
    camera.lookAt(0, 0, 0);
    scene.add(mesh);
    console.log(scene)
    process.value = 100
  }, (progress) => {
    console.log('progress.loaded', progress.loaded, "progress.total", progress.total)
    console.log(Math.min(progress.loaded / progress.total, 0.8))
    if(!progress.total) return
    const res = Math.min(progress.loaded / progress.total, 0.8)
    process.value = Math.floor(res * 100)
  }, (error) => {
    console.log(error)
  })
}

function initOrbitControls() {
  controls = new OrbitControls(camera, renderer.domElement);
  controls.screenSpacePanning = true;
  controls.enableRotate = true;
  controls.enableZoom = true;
  controls.enablePan = true; // 启用拖动
  controls.minDistance = 1;
  controls.maxDistance = 200000;
}





const clock = new THREE.Clock();
// 开始渲染
function render() {
  controls.update()


  scene.traverse(darkenNonBloomed);
  composer.render();
  scene.traverse(restoreMaterial);
  // render the entire scene, then render bloom scene on top
  finalComposer.render();


  const elapsed = clock.getElapsedTime();
  if (t1) {
    t1.position.y += Math.cos(elapsed) * .0015
    t2.position.y += Math.cos(elapsed) * .0015
    t1.rotation.y += 0.05
    t2.rotation.y += 0.05
  }

  requestAnimationFrame(render)
}
const materials = {};
function darkenNonBloomed(obj) {
  if (obj && bloomLayer.test(obj.layers) === false) {
    materials[obj.uuid] = obj.material;
    obj.material = darkMaterial;
  }
}
function restoreMaterial(obj) {
  if (materials[obj.uuid]) {
    obj.material = materials[obj.uuid];
    delete materials[obj.uuid];
  }
}

function updateDom() {
  let width = window.innerWidth
  let height = window.innerHeight
  controls.update()
  // 更新参数
  camera.aspect = width / height // 摄像机视锥体的长宽比，通常是使用画布的宽/画布的高
  camera.updateProjectionMatrix() // 更新摄像机投影矩阵。在任何参数被改变以后必须被调用,来使得这些改变生效
  renderer.setSize(width, height)
  renderer.setPixelRatio(window.devicePixelRatio) // 设置设备像素比
  labelRenderer.setSize(width, height)
}

function resize() {
  window.addEventListener('resize', () => {
    updateDom()

  })
}


</script>

<style lang="scss" scoped>
canvas {
  width: 100vw;
  height: 100vh;
  left: 0;
  position: absolute;
  top: 0;
}

.process {
  position: fixed !important;
  z-index: 3 !important;
  top: 50% !important;
  left: 50% !important;
  width: 500px !important;
  transform: translate(-50%, -50%) !important;

  :deep(.el-progress-bar__inner) {
    transition: width .2s ease;
    background-color: rgb(14, 50, 122);
  }

  ;
}
</style>

