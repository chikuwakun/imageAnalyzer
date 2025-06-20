<template>
  <div class="quadrant-window">
    <div class="window-titlebar bg-green-500">
      <span class="window-title">明るさ</span>
    </div>
    <div class="window-content bg-white flex items-center justify-center">
      <h2 v-if="!imageUrl" class="text-lg text-gray-500">画像をドラッグ＆ドロップ</h2>
      <canvas v-else ref="canvas" class="quadrant-image"></canvas>
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

// 画像をグレースケールに変換する関数
const convertToGrayscale = (imageUrl: string) => {
  if (!canvas.value) return;
  
  const ctx = canvas.value.getContext('2d');
  if (!ctx) return;
  
  const img = new Image();
  img.crossOrigin = 'Anonymous';
  
  img.onload = () => {
    console.log('Image loaded for brightness component, dimensions:', img.width, 'x', img.height);
    
    // キャンバスのサイズを画像に合わせる
    const container = canvas.value?.parentElement;
    const containerWidth = container ? container.clientWidth - 32 : img.width; // padding考慮
    const containerHeight = container ? container.clientHeight - 32 : img.height;
    
    console.log('Container size:', containerWidth, 'x', containerHeight);
    
    // アスペクト比を維持しながらキャンバスサイズを設定
    const scale = Math.min(
      containerWidth / img.width,
      containerHeight / img.height
    );
    
    const width = Math.max(1, Math.floor(img.width * scale));
    const height = Math.max(1, Math.floor(img.height * scale));
    
    console.log('Canvas size will be:', width, 'x', height);
    
    if (canvas.value) {
      canvas.value.width = width;
      canvas.value.height = height;
      console.log('Canvas size set to:', canvas.value.width, 'x', canvas.value.height);
    }
    
    // 画像を描画
    ctx.drawImage(img, 0, 0, width, height);
    console.log('Image drawn to canvas');
    
    try {
      // ピクセルデータを取得
      const imageData = ctx.getImageData(0, 0, width, height);
      const data = imageData.data;
      
      // 各ピクセルをXYZ表色系のY値を使ってグレースケールに変換
      for (let i = 0; i < data.length; i += 4) {
        const r = data[i] / 255;
        const g = data[i + 1] / 255;
        const b = data[i + 2] / 255;
        
        // sRGBからXYZ表色系のYを計算（明るさ成分）
        // ガンマ補正を戻す（線形RGB空間に変換）
        const r_linear = r <= 0.04045 ? r / 12.92 : Math.pow((r + 0.055) / 1.055, 2.4);
        const g_linear = g <= 0.04045 ? g / 12.92 : Math.pow((g + 0.055) / 1.055, 2.4);
        const b_linear = b <= 0.04045 ? b / 12.92 : Math.pow((b + 0.055) / 1.055, 2.4);
        
        // XYZ表色系のY成分（明るさ）を計算
        // 係数はsRGB->XYZ変換の標準的な値
        const y = 0.2126 * r_linear + 0.7152 * g_linear + 0.0722 * b_linear;
        
        // 線形輝度値にガンマ補正を再適用（sRGB変換）
        // 人間の目に合った明るさに変換
        const y_srgb = y <= 0.0031308 ? y * 12.92 : 1.055 * Math.pow(y, 1/2.4) - 0.055;
        
        // 8ビット値に戻す
        const grayValue = Math.round(y_srgb * 255);
        
        // RGBすべてに同じ値を設定（グレースケール）
        data[i] = grayValue;     // R
        data[i + 1] = grayValue; // G
        data[i + 2] = grayValue; // B
        // Alpha値はそのまま
      }
      
      // 変換したデータをキャンバスに戻す
      ctx.putImageData(imageData, 0, 0);
      console.log('Grayscale conversion complete');
    } catch (e) {
      console.error('Error processing image:', e);
    }
  };
  
  img.onerror = (e) => {
    console.error('Error loading image:', e);
  };
  
  img.src = imageUrl;
  console.log('Image source set to:', imageUrl.substring(0, 50) + '...');
};

// 画像URLが変更されたらグレースケール変換を実行
watch(() => props.imageUrl, (newUrl) => {
  console.log('Image URL changed:', newUrl ? 'URL exists' : 'URL is null');
  if (newUrl) {
    setTimeout(() => {
      convertToGrayscale(newUrl);
    }, 100); // 少し遅延させて確実にDOM要素が生成されるようにする
  }
}, { immediate: true });

// ウィンドウサイズが変わった時に再描画
onMounted(() => {
  console.log('Brightness component mounted');
  if (canvas.value && canvas.value.parentElement) {
    resizeObserver = new ResizeObserver(() => {
      console.log('Resize detected');
      if (props.imageUrl) {
        convertToGrayscale(props.imageUrl);
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

.quadrant-image {
  max-width: 100%;
  max-height: 100%;
  display: block;
}
</style>