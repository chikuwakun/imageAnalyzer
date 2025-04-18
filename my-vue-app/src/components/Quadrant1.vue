<template>
  <div class="quadrant-window">
    <div class="window-titlebar bg-blue-500">
      <span class="window-title">元画像</span>
    </div>
    <div class="window-content bg-white flex items-center justify-center">
      <h2 v-if="!imageUrl" class="text-lg text-gray-500">画像をドラッグ＆ドロップ</h2>
      <img v-else :src="imageUrl" class="quadrant-image" alt="Original image" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { watch } from 'vue';

const props = defineProps<{
  imageUrl: string | null;
}>();

// 画像URLが変更されたときにコンソールに表示（デバッグ用）
watch(() => props.imageUrl, (newUrl) => {
  console.log('元画像コンポーネント: 画像URL変更:', newUrl ? 'URLあり' : 'URLなし');
}, { immediate: true });
</script>

<style scoped>
.quadrant-window {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  border-radius: 8px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  overflow: hidden;
  border: 1px solid #e2e8f0;
}

.window-titlebar {
  height: 24px;
  display: flex;
  align-items: center;
  padding: 0 0.5rem;
}

.window-title {
  color: white;
  font-size: 12px;
  font-weight: 500;
}

.window-content {
  flex-grow: 1;
  padding: 1rem;
  overflow: hidden;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.quadrant-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  display: block;
}
</style>