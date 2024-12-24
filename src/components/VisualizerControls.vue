<template>
  <div class="controls flex gap-2">
    <button @click="$emit('togglePlay')" class="button play-btn">
      <icon v-if="isPlaying" src="/icons/pause.svg" />
      <icon v-else src="/icons/play.svg" />
    </button>

    <button class="toggle-abstraction-controls button" :disabled="!isPlaying" @click="$emit('toggleSettings')">
      <icon src="/icons/settings.svg" />
    </button>
  </div>
</template>

<script setup>
import Icon from '@/components/Icon.vue';
defineProps(['isPlaying']);
defineEmits(['audioUpload', 'togglePlay', 'toggleSettings']);
</script>

<style scoped lang="postcss">
:deep(.icon) {
  @apply h-12;
  filter: drop-shadow(0 0 6px var(--current-color));
}
</style>
