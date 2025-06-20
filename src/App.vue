<script setup lang="ts">
import { ref } from 'vue';
import OriginalImageView from './components/OriginalImageView.vue'
import BrightnessView from './components/BrightnessView.vue'
import PCCS from './components/PCCS.vue'
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

// 画像をクリアする
const clearImage = () => {
  imageUrl.value = null;
};

// ファイル選択ハンドラ
const handleFileSelect = (e: Event) => {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (event: ProgressEvent<FileReader>) => {
      if (event.target?.result) {
        imageUrl.value = event.target.result as string;
      }
    };
    reader.readAsDataURL(file);
  }
};
</script>

<template>
  <div class="app-container">
    <!-- メインコンテンツ -->
    <main 
      class="main-content" 
      @dragover="onDragOver" 
      @drop="onDrop"
    >
      <!-- 画像がない場合のドロップゾーン -->
      <div v-if="!imageUrl" class="drop-zone">
        <div class="drop-message">
          <div class="drop-icon">📁</div>
          <h2>画像をドラッグ＆ドロップしてください</h2>
          <p>または</p>
          <label class="file-input-label">
            画像を選択
            <input 
              type="file" 
              accept="image/*" 
              class="file-input" 
              @change="handleFileSelect"
            />
          </label>
          
        </div>
      </div>

      <!-- 4つのコンポーネント表示 -->
      <div v-else class="four-grid-layout">
        <div class="grid-item">
          <OriginalImageView :image-url="imageUrl" />
        </div>
        <div class="grid-item">
          <BrightnessView :image-url="imageUrl" />
        </div>
        <div class="grid-item">
          <PCCS :image-url="imageUrl" />
        </div>
        <div class="grid-item">
          <SaturationView :image-url="imageUrl" />
        </div>

        <!-- 画像クリアボタン -->
        <button class="clear-image-button" @click="clearImage">
          画像クリア
        </button>
      </div>
    </main>
  </div>
</template>

<style scoped>
.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  width: 100%;
  background-color: #121212;
  color: #f5f5f5;
}

/* メインコンテンツスタイル */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100vh;
  width: 100%;
  padding: 0;
  overflow: hidden;
}

.drop-zone {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 80%;
  height: 80%;
  border: 2px dashed #444;
  border-radius: 16px;
  background-color: rgba(255, 255, 255, 0.03);
}

.drop-zone:hover {
  border-color: #818cf8;
  background-color: rgba(129, 140, 248, 0.1);
}

.drop-message {
  text-align: center;
  padding: 2rem;
}

.drop-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.file-input-label {
  display: inline-block;
  margin-top: 1rem;
  padding: 0.75rem 1.5rem;
  background-color: #6366f1;
  color: white;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 500;
}

.file-input-label:hover {
  background-color: #4f46e5;
  box-shadow: 0 4px 12px rgba(79, 70, 229, 0.3);
}

.file-input {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* 4分割グリッドレイアウト */
.four-grid-layout {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-template-rows: 1fr 1fr;
  gap: 0;
  width: 100%;
  height: 100vh;
  position: relative;
  background-color: #121212;
}

.grid-item {
  background-color: transparent;
  overflow: hidden;
  width: 100%;
  height: 100%;
  border: none;
  padding: 0;
  margin: 0;
}

/* 画像クリアボタン */
.clear-image-button {
  position: fixed;
  bottom: 20px;
  right: 20px;
  padding: 10px 16px;
  background-color: rgba(255, 255, 255, 0.2);
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
  z-index: 100;
  transition: background-color 0.3s;
}

.clear-image-button:hover {
  background-color: rgba(255, 255, 255, 0.3);
}

/* レスポンシブ対応 */
@media (max-width: 768px) {
  .four-grid-layout {
    grid-template-columns: 1fr;
    grid-template-rows: repeat(4, 1fr);
  }
}
</style>
