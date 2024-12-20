# AudioVisualizer.vue
<template>
  <div class="audio-visualizer">
    <div class="image-container">
      <img
        :src="imageUrl || placeholderUrl"
        :style="imageStyle"
        alt="Visualization"
        ref="visualizerImage"
      />
    </div>

    <div class="controls">
      <div class="upload-controls">
        <label class="upload-btn">
          Upload Audio
          <input
            type="file"
            accept="audio/*"
            @change="handleAudioUpload"
            class="file-input"
          />
        </label>

        <label class="upload-btn">
          Upload Image
          <input
            type="file"
            accept="image/*"
            @change="handleImageUpload"
            class="file-input"
          />
        </label>
      </div>

      <button 
        @click="togglePlay"
        :disabled="!hasAudio"
        class="play-btn"
      >
        {{ isPlaying ? 'Pause' : 'Play' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onBeforeUnmount } from 'vue'

// Reactive state
const audioContext = ref(null)
const analyser = ref(null)
const audioSource = ref(null)
const audioBuffer = ref(null)
const isPlaying = ref(false)
const hasAudio = ref(false)
const imageUrl = ref(null)
const placeholderUrl = ref('/api/placeholder/500/500')
const animationId = ref(null)

const filters = ref({
  hueRotate: 0,
  brightness: 100,
  contrast: 100
})

// Computed styles
const imageStyle = computed(() => ({
  filter: `
    hue-rotate(${filters.value.hueRotate}deg)
    brightness(${filters.value.brightness}%)
    contrast(${filters.value.contrast}%)
  `
}))

// Methods
const handleAudioUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  // Stop any existing playback
  stopPlayback()

  // Initialize audio context if needed
  if (!audioContext.value) {
    audioContext.value = new (window.AudioContext || window.webkitAudioContext)()
    analyser.value = audioContext.value.createAnalyser()
    analyser.value.fftSize = 256
  }

  try {
    const arrayBuffer = await file.arrayBuffer()
    audioBuffer.value = await audioContext.value.decodeAudioData(arrayBuffer)
    hasAudio.value = true
  } catch (error) {
    console.error('Error loading audio:', error)
    hasAudio.value = false
  }
}

const handleImageUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    imageUrl.value = e.target.result
  }
  reader.readAsDataURL(file)
}

const setupAudioSource = () => {
  audioSource.value = audioContext.value.createBufferSource()
  audioSource.value.buffer = audioBuffer.value
  audioSource.value.connect(analyser.value)
  analyser.value.connect(audioContext.value.destination)
}

const startPlayback = () => {
  if (!hasAudio.value) return
  
  setupAudioSource()
  audioSource.value.start(0)
  animate()
  isPlaying.value = true
}

const stopPlayback = () => {
  if (audioSource.value) {
    audioSource.value.stop()
  }
  if (animationId.value) {
    cancelAnimationFrame(animationId.value)
  }
  isPlaying.value = false
}

const togglePlay = () => {
  if (isPlaying.value) {
    stopPlayback()
  } else {
    startPlayback()
  }
}

const animate = () => {
  const dataArray = new Uint8Array(analyser.value.frequencyBinCount)
  analyser.value.getByteFrequencyData(dataArray)
  
  // Calculate average frequency
  const average = dataArray.reduce((a, b) => a + b) / dataArray.length
  
  // Update filters based on audio data
  filters.value = {
    hueRotate: (average * 1.5) % 360,
    brightness: 100 + (average / 2),
    contrast: 100 + (average / 2)
  }
  
  animationId.value = requestAnimationFrame(animate)
}

// Cleanup
onBeforeUnmount(() => {
  stopPlayback()
  if (audioContext.value) {
    audioContext.value.close()
  }
})
</script>

<style scoped>
.audio-visualizer {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
}

.image-container {
  margin-bottom: 2rem;
  max-width: 500px;
  width: 100%;
}

.image-container img {
  width: 100%;
  height: auto;
  transition: filter 0.1s ease;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
}

.upload-controls {
  display: flex;
  gap: 1rem;
}

.upload-btn {
  padding: 0.5rem 1rem;
  background: #4a5568;
  color: white;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.upload-btn:hover {
  background: #2d3748;
}

.file-input {
  display: none;
}

.play-btn {
  padding: 0.5rem 2rem;
  background: #48bb78;
  color: white;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: background-color 0.2s;
}

.play-btn:hover {
  background: #38a169;
}

.play-btn:disabled {
  background: #a0aec0;
  cursor: not-allowed;
}
</style>