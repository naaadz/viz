# AudioVisualizer.vue
<template>
  <div>
    <div class="audio-visualizer">
      <div class="visualization-container" ref="container">
        <!-- Main image with filters -->
        <img
          :src="imageUrl || placeholderUrl"
          :style="imageStyle"
          class="the-image"
          alt="Visualization"
          ref="visualizerImage"
        />

        <!-- SVG Overlay for morphing shapes -->
        <svg class="svg-overlay" :viewBox="`0 0 ${containerWidth} ${containerHeight}`">
          <path :d="morphPath" fill="none" stroke="rgba(255,255,255,0.5)" stroke-width="2" />
        </svg>

        <!-- Canvas for particles -->
        <canvas
          ref="particleCanvas"
          class="particle-overlay"
          :width="containerWidth"
          :height="containerHeight"
        ></canvas>

        <!-- Waveform overlay -->
        <canvas
          ref="waveformCanvas"
          class="waveform-overlay"
          :width="containerWidth"
          :height="containerHeight"
        ></canvas>
      </div>
    </div>
    <div class="controls">
      <div class="upload-controls">
        <label class="upload-btn">
          Upload Audio
          <input type="file" accept="audio/*" @change="handleAudioUpload" class="file-input" />
        </label>

        <label class="upload-btn">
          Upload Image
          <input type="file" accept="image/*" @change="handleImageUpload" class="file-input" />
        </label>
      </div>

      <div class="effect-toggles">
        <label v-for="effect in effects" :key="effect.name">
          <input type="checkbox" v-model="effect.active" @change="updateEffects" />
          {{ effect.name }}
        </label>
      </div>

      <button @click="togglePlay" :disabled="!hasAudio" class="play-btn">
        {{ isPlaying ? 'Pause' : 'Play' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import {ref, computed, onMounted, onBeforeUnmount} from 'vue';

// Reactive state
const audioContext = ref(null);
const analyser = ref(null);
const audioSource = ref(null);
const audioBuffer = ref(null);
const isPlaying = ref(false);
const hasAudio = ref(false);
const imageUrl = ref(null);
const placeholderUrl = ref('/api/placeholder/500/500');
const animationId = ref(null);
const container = ref(null);
const particleCanvas = ref(null);
const waveformCanvas = ref(null);
const containerWidth = ref(500);
const containerHeight = ref(500);

// Visualization state
const particles = ref([]);
const beatDetected = ref(false);
const energyHistory = ref([]);
const ENERGY_THRESHOLD = 0.8;

// Effect toggles
const effects = ref([
  {name: 'Filters', active: true},
  {name: 'Transform', active: false},
  {name: 'Particles', active: false},
  {name: 'Waveform', active: true},
  {name: 'Morphing', active: true}
]);

// Animation state
const filters = ref({
  hueRotate: 0,
  brightness: 100,
  contrast: 100,
  blur: 0,
  scale: 1,
  rotate: 0,
  translateZ: 0,
  perspective: 1000,
  rotateY: 0
});

// SVG morph state
const morphPoints = ref([]);
const morphPath = computed(() => {
  if (morphPoints.value.length === 0) return '';
  return `M ${morphPoints.value.join(' L ')} Z`;
});

// Computed styles
const imageStyle = computed(() => ({
  filter: effects.value.find((e) => e.name === 'Filters')?.active
    ? `
      hue-rotate(${filters.value.hueRotate}deg)
      brightness(${filters.value.brightness}%)
      contrast(${filters.value.contrast}%)
      blur(${filters.value.blur}px)
    `
    : '',
  transform: effects.value.find((e) => e.name === 'Transform')?.active
    ? `
      perspective(${filters.value.perspective}px)
      translateZ(${filters.value.translateZ}px)
      scale(${filters.value.scale})
      rotate(${filters.value.rotate}deg)
      rotateY(${filters.value.rotateY}deg)
    `
    : ''
}));

// Initialize particles
const initParticles = () => {
  const count = 100;
  particles.value = Array.from({length: count}, () => ({
    x: Math.random() * containerWidth.value,
    y: Math.random() * containerHeight.value,
    size: Math.random() * 5,
    speedX: Math.random() * 2 - 1,
    speedY: Math.random() * 2 - 1
  }));
};

// Update particle positions
const updateParticles = (bassAvg, midAvg, highAvg) => {
  const ctx = particleCanvas.value.getContext('2d');
  ctx.clearRect(0, 0, containerWidth.value, containerHeight.value);
  ctx.fillStyle = 'rgba(255,255,255,0.5)';

  particles.value.forEach((particle) => {
    // Update size and speed based on frequencies
    particle.size = Math.max(2, bassAvg / 30);
    particle.x += particle.speedX * (midAvg / 10);
    particle.y += particle.speedY * (highAvg / 10);

    // Wrap around edges
    if (particle.x < 0) particle.x = containerWidth.value;
    if (particle.x > containerWidth.value) particle.x = 0;
    if (particle.y < 0) particle.y = containerHeight.value;
    if (particle.y > containerHeight.value) particle.y = 0;

    // Draw particle
    ctx.beginPath();
    ctx.arc(particle.x, particle.y, particle.size, 0, Math.PI * 2);
    ctx.fill();
  });
};

// Draw waveform
const drawWaveform = (timeData) => {
  const ctx = waveformCanvas.value.getContext('2d');
  ctx.clearRect(0, 0, containerWidth.value, containerHeight.value);

  ctx.beginPath();
  ctx.strokeStyle = 'rgba(255,255,255,0.5)';
  ctx.lineWidth = 2;

  const sliceWidth = containerWidth.value / timeData.length;
  let x = 0;

  timeData.forEach((v, i) => {
    const y = (v / 128.0) * (containerHeight.value / 2);
    if (i === 0) {
      ctx.moveTo(x, y);
    } else {
      ctx.lineTo(x, y);
    }
    x += sliceWidth;
  });

  ctx.stroke();
};

// Update SVG morph
const updateMorph = (bassAvg) => {
  const points = [];
  const numPoints = 8;
  const centerX = containerWidth.value / 2;
  const centerY = containerHeight.value / 2;
  const radius = 50 + bassAvg / 2;

  for (let i = 0; i < numPoints; i++) {
    const angle = (i / numPoints) * Math.PI * 2;
    const x = centerX + Math.cos(angle) * radius;
    const y = centerY + Math.sin(angle) * radius;
    points.push(`${x},${y}`);
  }

  morphPoints.value = points;
};

// Beat detection
const detectBeat = (dataArray) => {
  const average = dataArray.reduce((a, b) => a + b) / dataArray.length;
  const energy = average / 256;
  energyHistory.value.push(energy);

  if (energyHistory.value.length > 30) {
    energyHistory.value.shift();
  }

  const averageEnergy = energyHistory.value.reduce((a, b) => a + b) / energyHistory.value.length;
  beatDetected.value = energy > averageEnergy * ENERGY_THRESHOLD;
};

const animate = () => {
  const freqData = new Uint8Array(analyser.value.frequencyBinCount);
  const timeData = new Uint8Array(analyser.value.frequencyBinCount);

  analyser.value.getByteFrequencyData(freqData);
  analyser.value.getByteTimeDomainData(timeData);

  // Split frequency data into bands
  const bassData = freqData.slice(0, 8);
  const midData = freqData.slice(8, 24);
  const highData = freqData.slice(24, 48);

  const bassAvg = bassData.reduce((a, b) => a + b) / bassData.length;
  const midAvg = midData.reduce((a, b) => a + b) / midData.length;
  const highAvg = highData.reduce((a, b) => a + b) / highData.length;

  // Detect beats
  detectBeat(bassData);

  // Update all visualizations
  if (
    effects.value.find((e) => e.name === 'Filters')?.active ||
    effects.value.find((e) => e.name === 'Transform')?.active
  ) {
    filters.value = {
      hueRotate: (midAvg * 1.5) % 360,
      brightness: 100 + highAvg / 2,
      contrast: 100 + midAvg / 2,
      blur: beatDetected.value ? highAvg / 30 : 0,
      scale: 1 + bassAvg / 500 + (beatDetected.value ? 0.1 : 0),
      rotate: midAvg / 5 - 10,
      translateZ: -100 + midAvg,
      perspective: 1000 + bassAvg * 2,
      rotateY: highAvg / 2
    };
  }

  if (effects.value.find((e) => e.name === 'Particles')?.active) {
    updateParticles(bassAvg, midAvg, highAvg);
  }

  if (effects.value.find((e) => e.name === 'Waveform')?.active) {
    drawWaveform(timeData);
  }

  if (effects.value.find((e) => e.name === 'Morphing')?.active) {
    updateMorph(bassAvg);
  }

  animationId.value = requestAnimationFrame(animate);
};

// Audio handling methods
const handleAudioUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  // Stop any existing playback
  stopPlayback();

  // Initialize audio context if needed
  if (!audioContext.value) {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
    analyser.value = audioContext.value.createAnalyser();
    analyser.value.fftSize = 256;
  }

  try {
    const arrayBuffer = await file.arrayBuffer();
    audioBuffer.value = await audioContext.value.decodeAudioData(arrayBuffer);
    hasAudio.value = true;
  } catch (error) {
    console.error('Error loading audio:', error);
    hasAudio.value = false;
  }
};

const handleImageUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = (e) => {
    imageUrl.value = e.target.result;
  };
  reader.readAsDataURL(file);
};

const setupAudioSource = () => {
  audioSource.value = audioContext.value.createBufferSource();
  audioSource.value.buffer = audioBuffer.value;
  audioSource.value.connect(analyser.value);
  analyser.value.connect(audioContext.value.destination);
};

const startPlayback = () => {
  if (!hasAudio.value) return;

  setupAudioSource();
  audioSource.value.start(0);
  animate();
  isPlaying.value = true;
};

const stopPlayback = () => {
  if (audioSource.value) {
    try {
      audioSource.value.stop();
    } catch (error) {
      console.error('Error stopping audio source:', error);
    }
  }
  if (animationId.value) {
    cancelAnimationFrame(animationId.value);
  }
  isPlaying.value = false;
};

const togglePlay = () => {
  if (isPlaying.value) {
    stopPlayback();
  } else {
    startPlayback();
  }
};

const updateEffects = () => {
  // Clear canvases if effects are disabled
  if (!effects.value.find((e) => e.name === 'Particles')?.active) {
    const ctx = particleCanvas.value.getContext('2d');
    ctx.clearRect(0, 0, containerWidth.value, containerHeight.value);
  }
  if (!effects.value.find((e) => e.name === 'Waveform')?.active) {
    const ctx = waveformCanvas.value.getContext('2d');
    ctx.clearRect(0, 0, containerWidth.value, containerHeight.value);
  }
};

onMounted(() => {
  if (container.value) {
    containerWidth.value = container.value.offsetWidth;
    containerHeight.value = container.value.offsetHeight;
  }
  initParticles();
});

onBeforeUnmount(() => {
  stopPlayback();
  if (audioContext.value) {
    audioContext.value.close();
  }
});
</script>

<style scoped>

.audio-visualizer {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: fixed;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
  z-index: -1;
  pointer-events: none;
  
  height: 100vh;

  img {
    object-fit: cover;
  }
}

.visualization-container {
  position: relative;
  height: 100%;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.visualization-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: filter 0.1s ease, transform 0.1s ease;
}

.svg-overlay,
.particle-overlay,
.waveform-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1000;
}

.controls {
  padding: 2rem;
  position: fixed;
  left: 0;
  bottom: 0;
  background: black;
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
  color: white;
}

.effect-toggles {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
}

.effect-toggles label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  color: white;
  cursor: pointer;
}

/* Rest of the styles from previous version... */
</style>
