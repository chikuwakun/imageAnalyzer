<script setup lang="ts">
import { ref } from 'vue';
import OriginalImageView from './components/OriginalImageView.vue'
import BrightnessView from './components/BrightnessView.vue'
import HueView from './components/HueView.vue'
import SaturationView from './components/SaturationView.vue'

// ドラッグ＆ドロップした画像のURL
const imageUrl = ref<string | null>(null);

// ドラッグイベントのハンドラ
const onDragOver = (event: DragEvent) => {
  event.preventDefault();
};

// ドロップイベントのハンドラ
const onDrop = (event: DragEvent) => {
  event.preventDefault();
  
  if (!event.dataTransfer?.files.length) return;

  const file = event.dataTransfer.files[0];
  
  // ファイルが画像かどうかをチェック
  if (!file.type.match('image.*')) return;

  // FileReader APIを使って画像を読み込む
  const reader = new FileReader();
  reader.onload = (e: ProgressEvent<FileReader>) => {
    if (e.target?.result) {
      imageUrl.value = e.target.result as string;
    }
  };
  
  reader.readAsDataURL(file);
};
</script>

<template>
  <div 
    class="grid-container" 
    @dragover="onDragOver" 
    @drop="onDrop"
  >
    <div class="grid-item"><OriginalImageView :image-url="imageUrl" /></div>
    <div class="grid-item"><BrightnessView :image-url="imageUrl" /></div>
    <div class="grid-item"><HueView :image-url="imageUrl" /></div>
    <div class="grid-item"><SaturationView :image-url="imageUrl" /></div>
  </div>
</template>

<style scoped>
.grid-container {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(2, 1fr);
  gap: 1rem;
  height: 100vh;
  width: 100vw;
  position: absolute;
  top: 0;
  left: 0;
  padding: 1rem;
  box-sizing: border-box;
}

.grid-item {
  min-height: 0;
}

@media (max-width: 768px) {
  .grid-container {
    grid-template-columns: 1fr;
    grid-template-rows: repeat(4, 1fr);
  }
}
</style>
