<template>
  <div class="w-full h-full" id="container">
    <canvas id="canvas"></canvas>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted } from "vue"
import * as THREE from "three"
import WebGL from "three/addons/capabilities/WebGL.js"
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js"
import { OrbitControls } from "three/addons/controls/OrbitControls.js"
import PickHelper from "../utils/picker"

const pickHelper = new PickHelper()

let camera: any = null
let scene: any = null
let renderer: any = null
let geometry: any = null
let material: any = null
let controls: any = null
let model: any = null
let renderRequested: boolean = false
let canvas: any = null

const loadModel = () => {
  const loader = new GLTFLoader()

  loader.load(
    "/shirt.glb",
    function (gltf) {
      model = gltf.scene
      model.position.set(0, 0.5, 0)
      scene.add(model)
      //ARROW HELPER
      const Ox = new THREE.Vector3(0.6, 0, 0)
      const Oy = new THREE.Vector3(0, 0.6, 0)
      const Oz = new THREE.Vector3(0, 0, 0.6)
      //normalize the direction vector (convert to vector of length 1)
      Ox.normalize()
      Oy.normalize()
      Oz.normalize()

      //Origin of vector direction
      const origin = model.position
      const length = 1
      const OxColor = 0xff343e
      const OyColor = 0x04dd21
      const OzColor = 0x4394fc

      const OxHelper = new THREE.ArrowHelper(Ox, origin, length, OxColor)
      const OyHelper = new THREE.ArrowHelper(Oy, origin, length, OyColor)
      const OzHelper = new THREE.ArrowHelper(Oz, origin, length, OzColor)

      scene.add(OxHelper)
      scene.add(OyHelper)
      scene.add(OzHelper)

      //LOAD INITIAL MODEL
      render()
    },
    undefined,
    function (error) {
      console.error(error)
    },
  )
}

function render() {
  renderRequested = false
  renderer && renderer.render(scene, camera)
}

function requestRenderIfNotRequested() {
  if (!renderRequested) {
    renderRequested = true
    requestAnimationFrame(render)
  }
}

function resizeRendererToDisplaySize(renderer) {
  const canvas = renderer.domElement
  const width = canvas.clientWidth
  const height = canvas.clientHeight
  const needResize = canvas.width !== width || canvas.height !== height
  if (needResize) {
    renderer.setSize(width, height, false)
  }
  return needResize
}

const initial = () => {
  //SCENE
  canvas = document.querySelector("#canvas")
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xefffff)
  scene.backgroundBlurriness = 0.5

  //CAMERA
  camera = new THREE.PerspectiveCamera(90, window.innerWidth / window.innerHeight, 0.1, 10)
  camera.position.set(0, 0.8, 1.5)
  camera.lookAt(0, 0, 0)

  //RENDERER
  renderer = new THREE.WebGLRenderer({ antialias: true, canvas })
  renderer.setSize(window.innerWidth, window.innerHeight, false)

  // GRID HELPER
  const size = 100
  const divisions = 400
  const colorGrid = 0xd9e9fe
  const colorCenter = 0xed8e91
  const gridHelper = new THREE.GridHelper(size, divisions, colorCenter, colorGrid)
  scene.add(gridHelper)

  controls = new OrbitControls(camera, canvas)
  controls.enableDamping = true
  controls.target.set(0, 0, 0)
  controls.update()
  controls.addEventListener("change", requestRenderIfNotRequested)

  // Create a transparent ground plane
  const planeGeometry = new THREE.PlaneGeometry(100, 100)
  const planeMaterial = new THREE.MeshBasicMaterial({
    color: 0xe0e7ff,
    side: THREE.DoubleSide,
    transparent: true,
    opacity: 0.2,
  })
  const plane = new THREE.Mesh(planeGeometry, planeMaterial)
  plane.rotation.x = -Math.PI / 2 // Rotate the plane to be horizontal

  scene.add(plane)

  //AXES HELPER DEFAULT (Oxyz)
  // const axesHelper = new THREE.AxesHelper(5)
  // scene.add(axesHelper)

  //BOX HELPER (box wrapper area)
  // const sphere = new THREE.SphereGeometry(1, 1, 1)
  // const object = new THREE.Mesh(sphere, new THREE.MeshBasicMaterial(0xff0000))
  // const box = new THREE.BoxHelper(object, 0xffff00)
  // scene.add(box)

  //CAMERA HELPER
  // const helper = new THREE.CameraHelper(camera)
  // scene.add(helper)

  //ADD LIGHT & LIGHT HELPER

  const light = new THREE.DirectionalLight(0xffffff)
  scene.add(light)
  const helper = new THREE.DirectionalLightHelper(light, 5)
  scene.add(helper)

  //POLAR GRID HELPER
  // const radius = 10
  // const sectors = 16
  // const rings = 8
  // const divisions = 64

  // const helper = new THREE.PolarGridHelper(radius, sectors, rings, divisions)
  // scene.add(helper)
}

const pickPosition = { x: 0, y: 0 }
clearPickPosition()

function getCanvasRelativePosition(event) {
  const rect = canvas.getBoundingClientRect()
  return {
    x: ((event.clientX - rect.left) * canvas.width) / rect.width,
    y: ((event.clientY - rect.top) * canvas.height) / rect.height,
  }
}

function setPickPosition(event) {
  const pos = getCanvasRelativePosition(event)
  pickPosition.x = (pos.x / canvas.width) * 2 - 1
  pickPosition.y = (pos.y / canvas.height) * -2 + 1 // note we flip Y
}

function clearPickPosition() {
  // unlike the mouse which always has a position
  // if the user stops touching the screen we want
  // to stop picking. For now we just pick a value
  // unlikely to pick something
  pickPosition.x = -100000
  pickPosition.y = -100000
}

onMounted(() => {
  initial()
  loadModel()

  if (WebGL.isWebGLAvailable()) {
    // Initiate function or other initializations here
    render()
  } else {
    const warning = WebGL.getWebGLErrorMessage()
    document.getElementById("container")?.appendChild(warning)
  }

  //RENDER ON DEMAND
  window.addEventListener("resize", requestRenderIfNotRequested)
  window.addEventListener("mousemove", setPickPosition)
  window.addEventListener("mouseout", clearPickPosition)
  window.addEventListener("mouseleave", clearPickPosition)
  window.addEventListener(
    "touchstart",
    (event) => {
      // prevent the window from scrolling
      event.preventDefault()
      setPickPosition(event.touches[0])
    },
    { passive: false },
  )

  window.addEventListener("touchmove", (event) => {
    setPickPosition(event.touches[0])
  })

  window.addEventListener("touchend", clearPickPosition)
})

onBeforeUnmount(() => {
  window.removeEventListener("resize", requestRenderIfNotRequested)
  window.removeEventListener("mousemove", setPickPosition)
  window.removeEventListener("mouseout", clearPickPosition)
  window.removeEventListener("mouseleave", clearPickPosition)
  window.removeEventListener("touchstart", setPickPosition)
  window.removeEventListener("touchmove", setPickPosition)
  window.removeEventListener("touchend", clearPickPosition)
})
</script>

<style scoped></style>
