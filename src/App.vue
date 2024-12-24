<template>
  <div class="scope-visualizer" :style="controlStyles">
    <TransitionGroup name="fade" tag="div" class="bg-container">
      <img :class="`img-${currentImageIndex}`" :key="currentImageIndex" :src="currentImage" class="bg" alt="" />
    </TransitionGroup>

    <canvas ref="canvas" class="scope-canvas"></canvas>

    <div class="fixed bottom-8 flex justify-center inset-x-0">
      <VisualizerControls
        ref="controlsRef"
        :isPlaying="isPlaying"
        @audioUpload="handleAudioUpload"
        @togglePlay="togglePlay"
        @toggleSettings="abstractionControlsVisible = !abstractionControlsVisible"
      />

      <VisualizerSettings
        :visible="abstractionControlsVisible"
        :settings="settings"
        v-model:visible="abstractionControlsVisible"
      />

    </div>
    <nav class="caps fixed top-0 text-sm flex gap-4 ms-[7%] sm:ms-7">
      <span class="text-white/50">2024</span>
      <span>Claude</span>
      <span>/</span>
      <span>MJ</span>
      <span>/</span>
      <span>Algrus</span>
    </nav>
  </div>
</template>

<script setup>
import {ref, computed, onMounted, provide, onBeforeUnmount} from 'vue';
import VisualizerControls from './components/VisualizerControls.vue';
import VisualizerSettings from './components/VisualizerSettings.vue';

// State management
const settings = ref({
  waveModulation: 30,
  curveSmoothing: 70,
  timeDistortion: 20,
  colorHue: 180,
  blendMode: 'hard-light'
});

const canvas = ref(null);
const ctx = ref(null);
const audioElement = ref(null);
const audioContext = ref(null);
const analyser = ref(null);
const isPlaying = ref(false);
const hasAudio = ref(false);
const animationId = ref(null);
const abstractionControlsVisible = ref(false);
const currentImageIndex = ref(1);
const imageInterval = ref(null);
const prevWaveform = ref(new Float32Array(0));
const controlsRef = ref(null);

// Provide
provide('controlsRef', controlsRef);

// Computed properties
const currentImage = computed(() => `/viz-${currentImageIndex.value}.png`);
const currentColor = computed(() => `hsl(${settings.value.colorHue}, 100%, 50%)`);
const currentColorDim = computed(() => `hsla(${settings.value.colorHue}, 100%, 50%, 0.2)`);
const controlStyles = computed(() => ({
  '--current-color': currentColor.value,
  '--current-color-dim': currentColorDim.value,
  '--current-blend-mode': settings.value.blendMode
}));

// Canvas initialization
const initCanvas = () => {
  if (!canvas.value) return;

  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;
  ctx.value = canvas.value.getContext('2d');

  ctx.value.shadowBlur = 10;
  ctx.value.shadowColor = '#0f0';
  ctx.value.lineWidth = 2;

  ctx.value.fillStyle = `hsla(${settings.value.colorHue}, 100%, 20%, .8)`;
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height);
};

// Audio handling
const handleAudioUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  stopAudioPlayback();

  try {
    audioElement.value = new Audio();
    const audioUrl = URL.createObjectURL(file);
    audioElement.value.src = audioUrl;

    audioElement.value.onloadeddata = () => {
      setupAudioChain();
      hasAudio.value = true;
    };
  } catch (error) {
    console.error('Error loading audio:', error);
    hasAudio.value = false;
  }
};

const setupAudioChain = () => {
  if (!audioContext.value) {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
    analyser.value = audioContext.value.createAnalyser();
    analyser.value.fftSize = 2048;
  }

  const source = audioContext.value.createMediaElementSource(audioElement.value);
  source.connect(analyser.value);
  analyser.value.connect(audioContext.value.destination);

  prevWaveform.value = new Float32Array(analyser.value.frequencyBinCount);
};

const togglePlay = () => {
  if (!audioElement.value) {
    audioElement.value = new Audio('/triba.mp3');
    audioElement.value.onloadeddata = () => {
      setupAudioChain();
      hasAudio.value = true;
      startAudioPlayback();
    };
    return;
  }

  abstractionControlsVisible.value = false;

  if (isPlaying.value) {
    stopAudioPlayback();
  } else {
    startAudioPlayback();
  }
};

const startAudioPlayback = () => {
  audioElement.value.play();
  animate();
  isPlaying.value = true;

  imageInterval.value = setInterval(() => {
    currentImageIndex.value = currentImageIndex.value === 3 ? 1 : currentImageIndex.value + 1;
  }, 30000);
};

const stopAudioPlayback = () => {
  if (audioElement.value) {
    audioElement.value.pause();
  }
  if (animationId.value) {
    cancelAnimationFrame(animationId.value);
  }
  if (imageInterval.value) {
    clearInterval(imageInterval.value);
  }
  isPlaying.value = false;
};

// Visualization
const drawWaveform = (timeData) => {
  const width = canvas.value.width;
  const height = canvas.value.height;
  const centerY = height / 2;
  const scale = height / 3;

  ctx.value.fillStyle = 'rgba(0, 0, 0, 0.1)';
  ctx.value.fillRect(0, 0, width, height);

  const modScale = settings.value.waveModulation / 100;
  const smoothScale = settings.value.curveSmoothing / 100;
  const timeScale = settings.value.timeDistortion / 100;

  const waveColor = `hsl(${settings.value.colorHue}, 100%, 50%)`;
  ctx.value.shadowColor = waveColor;

  drawWaveformLayer(timeData, centerY, scale, 1.0, waveColor, 1.0, modScale, smoothScale, timeScale);
  drawWaveformLayer(timeData, centerY, scale * 0.8, 0.7, waveColor, 0.4, modScale * 0.8, smoothScale, timeScale);
  drawWaveformLayer(timeData, centerY, scale * 1.2, 1.3, waveColor, 0.3, modScale * 1.2, smoothScale, timeScale);
};

const drawWaveformLayer = (
  timeData,
  centerY,
  scale,
  timeScale,
  color,
  alpha,
  modScale,
  smoothScale,
  timeDistortionScale
) => {
  const width = canvas.value.width;

  ctx.value.beginPath();
  ctx.value.strokeStyle = color;
  ctx.value.globalAlpha = alpha;

  const sliceWidth = width / timeData.length;
  let x = 0;

  for (let i = 0; i < timeData.length; i++) {
    const time = Date.now() / 1000;
    const modulation = Math.sin(time * 2 + i * 0.1) * 0.15 * modScale;
    const frequencyMod = Math.sin(i * 0.1) * 0.1 * modScale;
    const timeDistortion = Math.sin(time * timeDistortionScale + i * 0.01) * 0.1;
    const v = timeData[i] * timeScale + modulation + frequencyMod + timeDistortion;
    const smoothingFactor = 0.3 + smoothScale * 0.6;
    const smoothV = v * (1 - smoothingFactor) + (prevWaveform.value[i] || 0) * smoothingFactor;
    const y = centerY + smoothV * scale;

    if (i === 0) {
      ctx.value.moveTo(x, y);
    } else {
      const prevX = x - sliceWidth;
      const prevY = centerY + prevWaveform.value[i - 1] * scale;
      const cpx = (x + prevX) / 2;
      const curveIntensity = smoothScale * 1.5;
      ctx.value.quadraticCurveTo(
        cpx + Math.sin(time + i) * curveIntensity,
        prevY + Math.cos(time + i) * curveIntensity,
        x,
        y
      );
    }

    x += sliceWidth;
    prevWaveform.value[i] = smoothV;
  }

  ctx.value.stroke();
  ctx.value.globalAlpha = 1.0;
};

const animate = () => {
  if (!ctx.value || !analyser.value) return;

  const timeData = new Float32Array(analyser.value.frequencyBinCount);
  analyser.value.getFloatTimeDomainData(timeData);

  const processedData = new Float32Array(timeData.length);
  for (let i = 0; i < timeData.length; i++) {
    const phaseShift = Math.sin(Date.now() / 1000 + i * 0.01) * 0.1;
    processedData[i] = timeData[i] + phaseShift;
  }

  drawWaveform(processedData);
  animationId.value = requestAnimationFrame(animate);
};

onMounted(() => {
  initCanvas();
  window.addEventListener('resize', initCanvas);
});

onBeforeUnmount(() => {
  stopAudioPlayback();
  if (audioContext.value) {
    audioContext.value.close();
  }
  window.removeEventListener('resize', initCanvas);
});
</script>

<style scoped lang="postcss">
.scope-visualizer {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: var(--bg-color);
}

.scope-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  mix-blend-mode: var(--current-blend-mode);
}

nav {
  transform: rotate(90deg);
  transform-origin: left bottom;
  margin-block-start: 10px;
}

.bg-container {
  height: 100%;
}

.bg {
  height: 100%;
  width: 100%;
  object-fit: cover;

  &.img-1 {
    object-position: 62% center;

    @screen sm {
      object-position: center;
    }
  }

  &.img-2 {
    object-position: 16% center;

    @screen sm {
      object-position: center;
    }
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 1s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
