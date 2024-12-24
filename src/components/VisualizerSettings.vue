<template>
  <div
    ref="settingsRef"
    v-show="visible"
    class="settings absolute bottom-16 text-white p-6 w-[85%] sm:w-[300px] flex flex-col gap-4"
  >
    <div class="control-group">
      <div class="flex justify-between items-center title">
        <span>Wave Modulation</span>
        <span class="value">{{ settings.waveModulation }}%</span>
      </div>
      <input type="range" v-model="settings.waveModulation" min="0" max="100" class="slider" />
    </div>

    <div class="control-group">
      <div class="flex justify-between items-center title">
        <span>Curve Smoothness</span>
        <span class="value">{{ settings.curveSmoothing }}%</span>
      </div>
      <input type="range" v-model="settings.curveSmoothing" min="0" max="100" class="slider" />
    </div>

    <div class="control-group">
      <div class="flex justify-between items-center title">
        <span>Time Distortion</span>
        <span class="value">{{ settings.timeDistortion }}%</span>
      </div>
      <input type="range" v-model="settings.timeDistortion" min="0" max="100" class="slider" />
    </div>

    <div class="control-group">
      <div class="flex justify-between items-center title">
        <span>Blend Mode</span>
      </div>
      <select v-model="settings.blendMode" class="bg-transparent border px-2 py-1 w-full mt-2">
        <option value="hard-light">Hard Light</option>
        <option value="color-dodge">Color Dodge</option>
        <option value="screen">Screen</option>
      </select>
    </div>

    <div class="control-group">
      <div class="flex justify-between items-center title">
        <span>Color</span>
      </div>
      <input type="range" v-model="settings.colorHue" min="0" max="360" class="slider color-slider" />
    </div>
  </div>
</template>

<script setup>
import {onClickOutside} from '@vueuse/core';
import {ref, inject} from 'vue';

const props = defineProps(['visible', 'settings']);
const emit = defineEmits(['update:visible']);
const settingsRef = ref(null);
const controlsRef = inject('controlsRef');

onClickOutside(
  settingsRef,
  () => {
    console.log('Click outside', controlsRef);
    emit('update:visible', false);
  },
  {ignore: [controlsRef]}
);
</script>

<style scoped lang="postcss">
.settings {
  @apply bg-black/60 border;
}

.title {
  @apply uppercase text-xs tracking-wider font-light;
}

.value {
  @apply text-right;
}

.slider {
  @apply bg-white w-full;
  width: 100%;
  height: 1px;
  border-radius: 2px;
  outline: none;
  -webkit-appearance: none;
}

.slider::-webkit-slider-thumb {
  @apply bg-white;
  -webkit-appearance: none;
  width: 0.7rem;
  height: 0.7rem;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.2s ease;
}

.slider::-webkit-slider-thumb:hover {
  transform: scale(1.3);
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
</style>
