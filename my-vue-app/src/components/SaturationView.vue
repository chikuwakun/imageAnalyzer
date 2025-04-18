<template>
  <div class="quadrant-window">
    <div class="window-titlebar bg-red-500">
      <span class="window-title">彩度</span>
    </div>
    <div class="window-content bg-white flex items-center justify-center">
      <h2 v-if="!imageUrl" class="text-lg text-gray-500">画像をドラッグ＆ドロップ</h2>
      <div v-else class="image-container">
        <canvas ref="canvas" class="quadrant-image"></canvas>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted } from 'vue';

const props = defineProps<{
  imageUrl: string | null;
}>();

const canvas = ref<HTMLCanvasElement | null>(null);
let resizeObserver: ResizeObserver | null = null;

// 画像の彩度成分を抽出する関数
const extractSaturation = (imageUrl: string) => {
  if (!canvas.value) return;
  
  const ctx = canvas.value.getContext('2d');
  if (!ctx) return;
  
  const img = new Image();
  img.crossOrigin = 'Anonymous';
  
  img.onload = () => {
    console.log('Image loaded for saturation component, dimensions:', img.width, 'x', img.height);
    
    // キャンバスのサイズを画像に合わせる
    const container = canvas.value?.parentElement?.parentElement;
    const containerWidth = container ? container.clientWidth - 16 : img.width; // パディングを考慮
    const containerHeight = container ? container.clientHeight - 16 : img.height;
    
    // アスペクト比を維持しながらキャンバスサイズを設定
    const scale = Math.min(
      containerWidth / img.width,
      containerHeight / img.height
    );
    
    const width = Math.max(1, Math.floor(img.width * scale));
    const height = Math.max(1, Math.floor(img.height * scale));
    
    if (canvas.value) {
      canvas.value.width = width;
      canvas.value.height = height;
    }
    
    // 画像を描画
    ctx.drawImage(img, 0, 0, width, height);
    
    try {
      // ピクセルデータを取得
      const imageData = ctx.getImageData(0, 0, width, height);
      const data = imageData.data;
      
      // 各ピクセルの彩度成分を抽出して表示
      for (let i = 0; i < data.length; i += 4) {
        const r = data[i];
        const g = data[i + 1];
        const b = data[i + 2];
        
        // RGB -> HSV変換
        const max = Math.max(r, g, b);
        const min = Math.min(r, g, b);
        const delta = max - min;
        
        // 彩度 (0-1)
        const s = max === 0 ? 0 : delta / max;
        
        // 彩度をグレースケールとして表示（高彩度=白、低彩度=黒）
        const saturationValue = Math.round(s * 255);
        
        data[i] = saturationValue;     // R
        data[i + 1] = saturationValue; // G
        data[i + 2] = saturationValue; // B
        // Alpha値はそのまま
      }
      
      // 変換したデータをキャンバスに戻す
      ctx.putImageData(imageData, 0, 0);
    } catch (e) {
      console.error('Error processing image for saturation:', e);
    }
  };
  
  img.onerror = (e) => {
    console.error('Error loading image for saturation:', e);
  };
  
  img.src = imageUrl;
};

// 画像URLが変更されたら処理を実行
watch(() => props.imageUrl, (newUrl) => {
  console.log('Saturation component: Image URL changed:', newUrl ? 'URL exists' : 'URL is null');
  if (newUrl) {
    setTimeout(() => {
      extractSaturation(newUrl);
    }, 100);
  }
}, { immediate: true });

// ウィンドウサイズが変わった時に再描画
onMounted(() => {
  console.log('Saturation component mounted');
  if (canvas.value && canvas.value.parentElement) {
    resizeObserver = new ResizeObserver(() => {
      if (props.imageUrl) {
        extractSaturation(props.imageUrl);
      }
    });
    resizeObserver.observe(canvas.value.parentElement);
  }
});

onUnmounted(() => {
  if (resizeObserver) {
    resizeObserver.disconnect();
  }
});
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

.image-container {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.quadrant-image {
  max-width: 100%;
  max-height: 100%;
  display: block;
  object-fit: contain;
}
</style>