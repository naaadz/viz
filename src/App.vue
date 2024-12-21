<template>
  <div class="scope-visualizer" :style="controlStyles">
    <TransitionGroup name="fade" tag="div" class="bg-container">
      <img :key="currentImageIndex" :src="currentImage" class="bg" alt="" />
    </TransitionGroup>
    <canvas ref="canvas" class="scope-canvas"></canvas>

    <div class="controls">
      <label class="button upload-btn disabled">
        <span>Upload Audio</span>
        <input type="file" accept="audio/*" @change="handleAudioUpload" class="file-input" />
      </label>

      <button @click="togglePlay" class="button play-btn">
        {{ isPlaying ? 'Pause' : 'Play' }}
      </button>

      <button
        class="toggle-abstraction-controls button"
        :disabled="!isPlaying"
        @click="abstractionControlsVisible = !abstractionControlsVisible"
      >
        ⚙
      </button>
    </div>

    <div v-show="abstractionControlsVisible" class="abstraction-controls">
      <div class="control-group">
        <label>
          <span>Wave Modulation</span>
          <input type="range" v-model="waveModulation" min="0" max="100" class="slider" />
        </label>
        <span class="value">{{ waveModulation }}%</span>
      </div>

      <div class="control-group">
        <label>
          <span>Curve Smoothness</span>
          <input type="range" v-model="curveSmoothing" min="0" max="100" class="slider" />
        </label>
        <span class="value">{{ curveSmoothing }}%</span>
      </div>

      <div class="control-group">
        <label>
          <span>Time Distortion</span>
          <input type="range" v-model="timeDistortion" min="0" max="100" class="slider" />
        </label>
        <span class="value">{{ timeDistortion }}%</span>
      </div>

      <div class="color-controls">
        <div class="control-group">
          <label>
            <span>Color</span>
            <input type="range" v-model="colorHue" min="0" max="360" class="slider color-slider" />
          </label>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import {ref, onMounted, computed, watch, onBeforeUnmount} from 'vue';

// Canvas and audio state
const canvas = ref(null);
const ctx = ref(null);
const audioElement = ref(null);
const audioContext = ref(null);
const analyser = ref(null);
const isPlaying = ref(false);
const hasAudio = ref(false);
const animationId = ref(null);
const abstractionControlsVisible = ref(false);

// Image state
const currentImageIndex = ref(1);
const imageInterval = ref(null);

// Abstraction control states
const waveModulation = ref(30); // Default modulation intensity
const curveSmoothing = ref(70); // Default curve smoothness
const timeDistortion = ref(20); // Default time distortion

// Color control state (0-360 for hue)
const colorHue = ref(180);

const currentImage = computed(() => `/viz-${currentImageIndex.value}.png`);

// Computed color values for UI elements
const currentColor = computed(() => `hsl(${colorHue.value}, 100%, 50%)`);
const currentColorDim = computed(() => `hsla(${colorHue.value}, 100%, 50%, 0.2)`);
const currentColorBright = computed(() => `hsl(${colorHue.value}, 100%, 70%)`);

// Reactive styles object for dynamic styling
const controlStyles = computed(() => ({
  '--current-color': currentColor.value,
  '--current-color-dim': currentColorDim.value,
  '--current-color-bright': currentColorBright.value
}));

// Previous waveform data for smooth transitions
const prevWaveform = ref(new Float32Array(0));

// Initialize canvas
const initCanvas = () => {
  if (!canvas.value) return;

  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;
  ctx.value = canvas.value.getContext('2d');

  // Set up shadow for glow effect
  ctx.value.shadowBlur = 10;
  ctx.value.shadowColor = '#0f0';
  ctx.value.lineWidth = 2;

  // Draw initial frame
  const width = canvas.value.width;
  const height = canvas.value.height;

  // Clear canvas
  ctx.value.fillStyle = `hsla(${colorHue.value}, 100%, 20%, .8)`;
  ctx.value.fillRect(0, 0, width, height);
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
    analyser.value.fftSize = 2048; // Higher FFT size for smoother waveform
  }

  const source = audioContext.value.createMediaElementSource(audioElement.value);
  source.connect(analyser.value);
  analyser.value.connect(audioContext.value.destination);

  // Initialize previous waveform array
  prevWaveform.value = new Float32Array(analyser.value.frequencyBinCount);
};

const togglePlay = () => {
  //if (!hasAudio.value) return;

  if (!audioElement.value) {
    audioElement.value = new Audio('/triba.mp3');
    audioElement.value.onloadeddata = () => {
      setupAudioChain();
      hasAudio.value = true;
      // Start playing once loaded
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

  // Start image cycling when playing
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
  // Clear the interval when paused
  if (imageInterval.value) {
    clearInterval(imageInterval.value);
  }
  isPlaying.value = false;
};

// Utility function for smooth transitions
const lerp = (start, end, amt) => {
  return (1 - amt) * start + amt * end;
};

// Enhanced waveform visualization with controls
const drawWaveform = (timeData) => {
  const width = canvas.value.width;
  const height = canvas.value.height;
  const centerY = height / 2;
  const scale = height / 3;

  // Apply slight fade effect
  ctx.value.fillStyle = 'rgba(0, 0, 0, 0.1)';
  ctx.value.fillRect(0, 0, width, height);

  // Scale factors based on slider values
  const modScale = waveModulation.value / 100;
  const smoothScale = curveSmoothing.value / 100;
  const timeScale = timeDistortion.value / 100;

  // Generate color from hue (using HSL with fixed saturation and lightness)
  const waveColor = `hsl(${colorHue.value}, 100%, 50%)`;

  // Update shadow color to match wave color
  ctx.value.shadowColor = waveColor;

  // Draw multiple layers with different characteristics
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

    // Modulation intensity controlled by slider
    const modulation = Math.sin(time * 2 + i * 0.1) * 0.15 * modScale;
    const frequencyMod = Math.sin(i * 0.1) * 0.1 * modScale;

    // Time distortion controlled by slider
    const timeDistortion = Math.sin(time * timeDistortionScale + i * 0.01) * 0.1;

    // Combine modulations with control scaling
    const v = timeData[i] * timeScale + modulation + frequencyMod + timeDistortion;

    // Smoothing controlled by slider
    const smoothingFactor = 0.3 + smoothScale * 0.6; // Range from 0.3 to 0.9
    const smoothV = v * (1 - smoothingFactor) + (prevWaveform.value[i] || 0) * smoothingFactor;
    const y = centerY + smoothV * scale;

    if (i === 0) {
      ctx.value.moveTo(x, y);
    } else {
      const prevX = x - sliceWidth;
      const prevY = centerY + prevWaveform.value[i - 1] * scale;
      const cpx = (x + prevX) / 2;

      // Curve intensity affected by smoothing
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

  // Apply some preprocessing to the time data
  const processedData = new Float32Array(timeData.length);
  for (let i = 0; i < timeData.length; i++) {
    // Add subtle phase shifting
    const phaseShift = Math.sin(Date.now() / 1000 + i * 0.01) * 0.1;
    processedData[i] = timeData[i] + phaseShift;
  }

  drawWaveform(processedData);
  animationId.value = requestAnimationFrame(animate);
};

// Lifecycle hooks
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

<style scoped>
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
}

.controls {
  position: fixed;
  bottom: 2rem;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 1rem;
  z-index: 1000;
}

.abstraction-controls {
  position: fixed;
  top: 2rem;
  right: 2rem;
  background: rgba(0, 0, 0, 0.7);
  padding: 1.5rem;
  border-radius: 0.5rem;
  border: 1px solid var(--current-color-dim);
  color: var(--current-color);
  font-family: monospace;
  z-index: 1000;
}

.control-group {
  margin-bottom: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;

  label {
    display: flex;
    gap: 1rem;
    align-items: center;

    .slider {
      flex: 1;
    }
  }
}

.control-group:last-child {
  margin-bottom: 0;
}

.slider {
  width: 200px;
  height: 4px;
  background: var(--current-color-dim);
  border-radius: 2px;
  outline: none;
  -webkit-appearance: none;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 16px;
  background: var(--current-color);
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.2s ease;
}

.slider::-webkit-slider-thumb:hover {
  transform: scale(1.1);
}

.value {
  font-size: 0.8rem;
  opacity: 0.8;
}

.color-controls {
  margin-top: 2rem;
  padding-top: 1rem;
  border-top: 1px solid var(--current-color-dim);
}

.color-slider {
  background: linear-gradient(
    to right,
    hsl(0, 100%, 50%),
    hsl(60, 100%, 50%),
    hsl(120, 100%, 50%),
    hsl(180, 100%, 50%),
    hsl(240, 100%, 50%),
    hsl(300, 100%, 50%),
    hsl(360, 100%, 50%)
  );
}

.color-slider::-webkit-slider-thumb {
  background: white;
  border: 2px solid rgba(0, 0, 0, 0.3);
}

button:disabled,
.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.button {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 0 1.5rem;
  background: var(--current-color-dim);
  border: 1px solid var(--current-color);
  border-radius: 0.5rem;
  color: var(--current-color);
  cursor: pointer;
  font-family: monospace;
  text-transform: uppercase;
  letter-spacing: 1px;
  transition: all 0.2s ease;
}

.toggle-abstraction-controls {
  font-size: 2rem;
}

.upload-btn:hover,
.play-btn:hover {
  background: var(--current-color-dim);
}

.file-input {
  display: none;
}

.play-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

canvas {
  mix-blend-mode: hard-light;
}

.bg-container {
  height: 100%;
}

.bg {
  height: 100%;
  width: 100%;
  object-fit: cover;
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
