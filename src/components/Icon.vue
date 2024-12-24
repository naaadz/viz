<template>
  <div v-html="svgContent" class="icon"></div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
const props = defineProps({
  src: {
    type: String,
    required: true
  }
});

const svgContent = ref('');

onMounted(async () => {
  try {
    const response = await fetch(props.src);
    const svg = await response.text();
    svgContent.value = svg;
  } catch (error) {
    console.error('Error loading the SVG:', error);
  }
});
</script>

<style scoped>
.icon {
  fill: currentColor; 
}

.icon :deep(svg) {
  display: block;
  width: 100%;
  height: 100%;
}

</style>
