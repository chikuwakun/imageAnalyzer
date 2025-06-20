<template>
  <div class="fullsize-container">
    <img 
      v-if="imageUrl"
      :src="imageUrl" 
      class="fullsize-image" 
      alt="Original image"
    />
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';

const props = defineProps<{
  imageUrl: string | null;
}>();
</script>

<style scoped>
.fullsize-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  background-color: transparent;
}

.fullsize-image {
  max-width: 100%;
  max-height: 100%;
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}
</style>