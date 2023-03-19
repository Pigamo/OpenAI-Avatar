<script setup lang="ts">
import { TransitionPresets, useTransition } from '@vueuse/core'

// const { pause, resume, isActive } = useIntervalFn(() => {
//     const num = randomNumGen(50, 200)
//     console.log(num)
//     MotionControl(num);
//     ComplexityControl(num);

// }, 4000)

import * as THREE from 'three'
import { isDark } from '~/composables'
const { messages } = defineProps<{ messages: Array<{ type: 'system' | 'user' | 'assistant'; text: string }> }>()
const time = ref(0)
const answers = ref(0)
const timeMotion = useTransition(time, {
  duration: 4000,
  transition: TransitionPresets.easeInOutQuad,
})
const answerMotion = useTransition(answers, {
  duration: 4000,
  transition: TransitionPresets.easeInOutQuad,
})
const isAnimate = ref(false)

const ComplexityControl = (length: number) => {
  const base = 4 // You can change the base to control the growth rate
  const scaleFactor = 0.01 // You can change the scaleFactor to control the output range
  answers.value = scaleFactor * Math.log(length + 1) / Math.log(base)
}
const MotionControl = (length: number) => {
  const base = 2 // You can change the base to control the growth rate
  const scaleFactor = 1.5 // You can change the scaleFactor to control the output range
  time.value += scaleFactor * Math.log(length + 1) / Math.log(base)
}
const idleMotion = ref(0)
const idleComplex = ref(0)

const randomNumGen = (min: number, max: number) => {
  return Math.floor(Math.random() * (max - min + 1) + min)
}
const container = ref()

let renderer: THREE.WebGLRenderer
let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let face: THREE.Mesh

onMounted(() => {
  init()
  animate()
})
onUnmounted(() => {
  renderer.dispose()
  scene.dispose()
  camera.dispose()
})
function createFace() {
  const geometry = new THREE.SphereGeometry(1, 32, 32)

  const uniforms = {
    time: { value: 0 },
    complete: { value: 0 },

  }

  const vertexShader = `
    uniform float time;
    uniform float complete;
    
    varying vec2 vUv;

    void main() {
      vUv = uv;

      vec3 newPosition = position;
      newPosition.z += abs(sin(position.y * 10.0 + time  ) * complete);
      newPosition.z += abs(sin(position.x * 5.0 + time  ) * complete);
      newPosition *= -1.0;
     

      gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
    }
  `

  const fragmentShader = `
    
    varying vec2 vUv;

      
      
    void main() {
      gl_FragColor = vec4(vec3(1.3*vUv, 0.9), 1.0);
    }
  `

  const material = new THREE.ShaderMaterial({
    uniforms,
    vertexShader,
    fragmentShader,
  })

  face = new THREE.Mesh(geometry, material)
  scene.add(face)
}

// Update face shader time uniform based on chat input
watch(
  messages,
  () => {
    if (messages.length > 0) {
      const lastMessage = messages[messages.length - 1]
      if (lastMessage.type === 'assistant') {
        MotionControl(lastMessage.text.length)
        ComplexityControl(lastMessage.text.length)
      }
    }
  },
  { deep: true },
)
function init() {
  const width = container.value.clientWidth
  const height = container.value.clientHeight

  // Setup renderer
  renderer = new THREE.WebGLRenderer({ antialias: true })

  renderer.setSize(width, height)
  container.value.appendChild(renderer.domElement)

  // Setup scene
  scene = new THREE.Scene()
  scene.background = !isDark ? new THREE.Color(0x374151) : new THREE.Color(0xFFE5E7EB)

  // Setup camera
  camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000)
  camera.position.z = 2

  // Create face model
  createFace()
}
watch(isDark, (newValue) => {
  if (scene)
    scene.background = newValue ? new THREE.Color(0x374151) : new THREE.Color(0xFFE5E7EB)
})
function animate() {
  requestAnimationFrame(animate)
  idleMotion.value += 0.01
  idleComplex.value = 0.06 + Math.sin(idleMotion.value * 0.1) * 0.0001

  if (face) {
    face.traverse((child) => {
      if (child instanceof THREE.Mesh && child.material instanceof THREE.ShaderMaterial) {
        child.material.uniforms.time.value = timeMotion.value + idleMotion.value
        child.material.uniforms.complete.value = answerMotion.value + idleComplex.value
        // console.log(child.material.uniforms.time.value, child.material.uniforms.complete.value)
      }
    })
  }

  renderer.render(scene, camera)
}
</script>

<template>
  <div ref="container" />
</template>

<style scoped>
.canvas-container {
  width: 100%;
  height: 100%;
}
</style>
