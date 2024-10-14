<script setup>
/**** Lissajous.vue 
Lissajous Curve Visualization Component

This Vue component creates a 3D visualization of Lissajous curves based on MIDI note inputs.
It uses Three.js for rendering and can display curves in 1D (circle), 2D, or 3D space.

Key features:
- Dynamically adjusts between 1D, 2D, and 3D representations based on the number of input MIDI notes
- Uses custom shaders for curve coloring and animation
- Supports camera controls for 3D view (orbit, pan, zoom)

Current limitations:
1. The entire curve is redrawn when MIDI notes change, which can be computationally expensive.
2. There's a bug where frequent note changes can cause the animation to speed up.
3. Replacing the Three.js scene on each update can impact CPU performance during playback.

Future improvements could include:
- Optimizing curve updates to avoid full redraws
- Fixing the animation speed bug
- Implementing a more efficient scene update mechanism


2024, félix ménard
felix@aura.town
https://github.com/AuraTown/3D-Lissajous-Curve

****/

import { ref, onMounted, onBeforeUnmount, watch, computed } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import * as Tone from "tone";

// Constants for visualization parameters
const BASE_SCALE_3D = 0.8;
const BASE_SCALE_2D = 0.5;
const BASE_SCALE_CIRCLE = 0.5;
const ROTATION_SPEED = 0.010105;
const FOV = 30;
const CAMERA_DISTANCE = 5;

// Component props
const props = defineProps({
  midiNotes: { type: Array, default: () => [60, 64, 67] },
  fov: { type: Number, default: FOV },
  cameraDistance: { type: Number, default: CAMERA_DISTANCE },
});

// Refs and variables for Three.js objects
const container = ref(null);
let scene, camera, renderer, curve, controls;

// Compute frequency ratios based on MIDI notes
const ratios = computed(() => {
  if (props.midiNotes.length === 0) return [1];
  const baseFreq = Tone.Frequency(props.midiNotes[0], "midi").toFrequency();
  return props.midiNotes.map((midi) => {
    // NOTE:  could be because floating point inacuracy that the loops dont connect.
    const freq = Tone.Frequency(midi, "midi").toFrequency();
    return freq / baseFreq;
  });
});

// Determine the dimensionality of the visualization
const is3D = computed(() => props.midiNotes.length >= 3);
const is2D = computed(() => props.midiNotes.length === 2);
const isCircle = computed(() => props.midiNotes.length <= 1);

// Function to clear the Three.js scene and dispose of objects
const clearScene = () => {
  if (scene) {
    scene.clear();
  }
  if (renderer) {
    renderer.dispose();
  }
  if (controls) {
    controls.dispose();
  }
  if (container.value) {
    const canvas = container.value.querySelector("canvas");
    if (canvas) {
      container.value.removeChild(canvas);
    }
  }
  scene = null;
  camera = null;
  renderer = null;
  curve = null;
  controls = null;
};

// Initialize the Three.js scene
const init = () => {
  clearScene();

  const width = Math.max(container.value.clientWidth, 300);
  const height = Math.max(container.value.clientHeight, 300);

  console.log("Container size:", width, "x", height);

  scene = new THREE.Scene();

  const aspect = width / height;
  camera = new THREE.PerspectiveCamera(props.fov, aspect, 0.1, 1000);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setClearColor(0x000000, 0);
  container.value.appendChild(renderer.domElement);

  if (is3D.value) {
    camera.position.z = props.cameraDistance / BASE_SCALE_3D;
  } else {
    camera.position.z = 1 / BASE_SCALE_2D;
  }

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.screenSpacePanning = false;
  controls.minDistance = props.cameraDistance / (2 * BASE_SCALE_3D);
  controls.maxDistance = (props.cameraDistance * 2) / BASE_SCALE_3D;
  controls.maxPolarAngle = Math.PI;

  if (!is3D.value) {
    controls.enableRotate = false;
    controls.enableZoom = false;
    controls.enablePan = false;
  }

  const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
  directionalLight.position.set(1, 1, 1);
  scene.add(directionalLight);

  createCurve();
  animate();
};

// Create the Lissajous curve based on current ratios
const createCurve = () => {
  let scale;
  if (isCircle.value) {
    scale = BASE_SCALE_CIRCLE;
  } else if (is2D.value) {
    scale = BASE_SCALE_2D;
  } else {
    scale = BASE_SCALE_3D;
  }

  // Determine the complexity of the ratios
  const complexity = ratios.value.reduce(
    (sum, ratio) => sum + Math.abs(ratio - 1),
    0
  );

  // Adjust number of points based on complexity
  const basePoints = 1000;
  const numPoints = Math.max(
    basePoints,
    Math.min(10000, Math.floor(basePoints * (1 + complexity)))
  );

  // Use a fixed number of cycles
  // TODO: What's a math way to figure out the optimal number of cycles? so the loop can connect, expecially if simple.
  const numCycles = 20; //was 10 initially
  const timeFrame = 2 * Math.PI * numCycles;

  const points = [];
  for (let i = 0; i < numPoints; i++) {
    const t = (i / numPoints) * timeFrame;
    let x, y, z;
    if (isCircle.value) {
      x = Math.cos(t) * scale;
      y = Math.sin(t) * scale;
      z = 0;
    } else if (is2D.value) {
      x = Math.sin(ratios.value[0] * t) * scale;
      y = Math.sin(ratios.value[1] * t) * scale;
      z = 0;
    } else {
      x = Math.sin(ratios.value[0] * t) * scale;
      y = Math.sin(ratios.value[1] * t) * scale;
      z = Math.sin(ratios.value[2] * t) * scale;
    }
    points.push(new THREE.Vector3(x, y, z));
  }

  const geometry = new THREE.BufferGeometry().setFromPoints(points);

  const material = new THREE.ShaderMaterial({
    uniforms: {
      color: { value: new THREE.Color(0xff0000) },
      time: { value: 0 },
    },
    vertexShader: `
      varying vec3 vNormal;
      varying vec3 vPosition;
      void main() {
        vNormal = normalize(normalMatrix * normal);
        vPosition = position;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      uniform vec3 color;
      uniform float time;
      varying vec3 vNormal;
      varying vec3 vPosition;
      void main() {
        vec3 light = normalize(vec3(sin(time), cos(time), 0.5));
        float dProd = max(0.0, dot(vNormal, light));
        vec3 baseColor = mix(color, vec3(1.0), dProd * 0.5);
        float pulse = sin(time * 3.0 + vPosition.x * 10.0 + vPosition.y * 10.0 + vPosition.z * 10.0) * 0.5 + 0.5;
        baseColor *= 0.8 + pulse * 0.2;
        gl_FragColor = vec4(baseColor, 1.0);
      }
    `,
    transparent: true,
  });

  curve = new THREE.Mesh(
    new THREE.TubeGeometry(
      new THREE.CatmullRomCurve3(points),
      numPoints,
      0.005 * scale,
      8,
      false
    ),
    material
  );
  scene.add(curve);

  console.log(
    "Curve created with ratios:",
    ratios.value,
    "numPoints:",
    numPoints,
    "timeFrame:",
    timeFrame
  );
};

// Animation loop
const animate = () => {
  if (!renderer || !scene || !camera) return;

  requestAnimationFrame(animate);
  if (controls) controls.update();
  if (curve && is3D.value) {
    curve.rotation.x += ROTATION_SPEED;
    curve.rotation.y += ROTATION_SPEED;
  }
  if (curve) curve.material.uniforms.time.value += 0.01;
  renderer.render(scene, camera);
};

// Handle window resize
const handleResize = () => {
  if (container.value && renderer && camera) {
    const width = Math.max(container.value.clientWidth, 300);
    const height = Math.max(container.value.clientHeight, 300);

    console.log("Resized to:", width, "x", height);

    camera.aspect = width / height;
    camera.updateProjectionMatrix();
    renderer.setSize(width, height);
  }
};

// Lifecycle hooks
onMounted(() => {
  console.log("Component mounted");
  init();
  //window.addEventListener("resize", handleResize);

  // const resizeObserver = new ResizeObserver(handleResize);
  // resizeObserver.observe(container.value);
});

onBeforeUnmount(() => {
  console.log("Component unmounting");
  window.removeEventListener("resize", handleResize);
  clearScene();
});

// Watch for changes in props
watch(
  () => props.midiNotes,
  () => {
    console.log("MIDI notes changed, recreating curve");
    init(); // Reinitialize to handle dimension changes
  },
  { deep: true }
);

watch([() => props.fov, () => props.cameraDistance], () => {
  console.log("Camera properties changed, reinitializing");
  init();
});
</script>

<template>
  <div ref="container" class="lissajous-container"></div>
</template>

<style scoped>
.lissajous-container {
  width: 100%;
  height: 100%;
  min-width: 150px;
  min-height: 150px;
}
</style>
