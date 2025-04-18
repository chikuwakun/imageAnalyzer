<template>
  <div class="quadrant-window">
    <div class="window-titlebar bg-green-500">
      <span class="window-title">明るさ</span>
    </div>
    <div class="window-content bg-white flex items-center justify-center">
      <h2 v-if="!imageUrl" class="text-lg text-gray-500">画像をドラッグ＆ドロップ</h2>
      <div v-else class="image-container">
        <canvas ref="canvas" class="quadrant-image"></canvas>
        <canvas 
          ref="histogramCanvas" 
          class="histogram-canvas"
          @mousedown="startSelection"
          @mousemove="updateSelection"
          @mouseup="endSelection"
          @mouseleave="endSelection"
        ></canvas>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted, reactive } from 'vue';

const props = defineProps<{
  imageUrl: string | null;
}>();

const canvas = ref<HTMLCanvasElement | null>(null);
const histogramCanvas = ref<HTMLCanvasElement | null>(null);
let resizeObserver: ResizeObserver | null = null;

// 選択範囲の状態を管理
const selection = reactive({
  active: false,
  startX: 0,
  currentX: 0,
  minBrightness: -1,
  maxBrightness: -1
});

// 元の画像データを保存
const originalImageData = ref<ImageData | null>(null);
// 現在のヒストグラムデータ
const histogramData = ref<number[]>([]);

// 選択開始
const startSelection = (event: MouseEvent) => {
  if (!histogramCanvas.value) return;
  
  selection.active = true;
  
  // キャンバス上の座標を取得
  const rect = histogramCanvas.value.getBoundingClientRect();
  const x = event.clientX - rect.left;
  
  selection.startX = x;
  selection.currentX = x;
  
  // 輝度値に変換
  const width = histogramCanvas.value.width;
  const minBrightness = Math.floor((Math.min(selection.startX, selection.currentX) / width) * 256);
  const maxBrightness = Math.floor((Math.max(selection.startX, selection.currentX) / width) * 256);
  
  selection.minBrightness = Math.max(0, Math.min(255, minBrightness));
  selection.maxBrightness = Math.max(0, Math.min(255, maxBrightness));
  
  // ヒストグラムを再描画して選択範囲を表示
  drawHistogramWithSelection(histogramData.value);
  
  // 選択した明るさ範囲の画素を強調表示
  highlightSelectedBrightnessRange();
};

// 選択更新
const updateSelection = (event: MouseEvent) => {
  if (!selection.active || !histogramCanvas.value) return;
  
  // キャンバス上の座標を取得
  const rect = histogramCanvas.value.getBoundingClientRect();
  const x = event.clientX - rect.left;
  
  selection.currentX = x;
  
  // 輝度値に変換
  const width = histogramCanvas.value.width;
  const minBrightness = Math.floor((Math.min(selection.startX, selection.currentX) / width) * 256);
  const maxBrightness = Math.floor((Math.max(selection.startX, selection.currentX) / width) * 256);
  
  selection.minBrightness = Math.max(0, Math.min(255, minBrightness));
  selection.maxBrightness = Math.max(0, Math.min(255, maxBrightness));
  
  // ヒストグラムを再描画して選択範囲を表示
  drawHistogramWithSelection(histogramData.value);
  
  // 選択した明るさ範囲の画素を強調表示
  highlightSelectedBrightnessRange();
};

// 選択終了
const endSelection = () => {
  if (!selection.active) return;
  selection.active = false;
  
  // 選択範囲が無効になった場合は画像を元に戻す
  if (selection.minBrightness === selection.maxBrightness) {
    restoreOriginalImage();
    selection.minBrightness = -1;
    selection.maxBrightness = -1;
  }
};

// 選択した明るさ範囲に対応する画素を赤く表示
const highlightSelectedBrightnessRange = () => {
  if (!canvas.value || !originalImageData.value || 
      selection.minBrightness < 0 || selection.maxBrightness < 0 ||
      selection.minBrightness >= selection.maxBrightness) return;
  
  const ctx = canvas.value.getContext('2d');
  if (!ctx) return;
  
  // 元の画像データをコピー
  const imageData = new ImageData(
    new Uint8ClampedArray(originalImageData.value.data),
    originalImageData.value.width,
    originalImageData.value.height
  );
  const data = imageData.data;
  
  // 各ピクセルをチェック
  for (let i = 0; i < data.length; i += 4) {
    // グレースケール値
    const grayValue = data[i]; // どのチャンネルも同じ値のはず
    
    // 選択した明るさ範囲内かチェック
    if (grayValue >= selection.minBrightness && grayValue <= selection.maxBrightness) {
      // 赤く表示（赤チャンネルはそのまま、緑と青は0）
      data[i] = 255;     // R（強制的に255に設定）
      data[i + 1] = 0;   // G
      data[i + 2] = 0;   // B
      // Alpha値はそのまま
    }
  }
  
  // 変更したデータをキャンバスに戻す
  ctx.putImageData(imageData, 0, 0);
};

// 元の画像に戻す
const restoreOriginalImage = () => {
  if (!canvas.value || !originalImageData.value) return;
  
  const ctx = canvas.value.getContext('2d');
  if (!ctx) return;
  
  ctx.putImageData(originalImageData.value, 0, 0);
};

// ヒストグラムを描画する関数（選択範囲表示機能付き）
const drawHistogramWithSelection = (grayValues: number[]) => {
  if (!histogramCanvas.value) return;
  
  const histCtx = histogramCanvas.value.getContext('2d');
  if (!histCtx) return;
  
  // ヒストグラムキャンバスのサイズを設定
  // メインキャンバスのサイズから右40%×下20%のサイズを計算
  if (canvas.value) {
    const width = Math.floor(canvas.value.width * 0.4);
    const height = Math.floor(canvas.value.height * 0.2);
    
    histogramCanvas.value.width = width;
    histogramCanvas.value.height = height;
    
    // 背景を半透明の黒にする
    histCtx.fillStyle = 'rgba(0, 0, 0, 0.6)';
    histCtx.fillRect(0, 0, width, height);
    
    // ヒストグラムの最大値を見つける（描画時のスケーリング用）
    const maxValue = Math.max(...grayValues);
    const scaleFactor = maxValue > 0 ? height / maxValue : 1;
    
    // ヒストグラムを描画（256階調）
    const barWidth = width / 256;
    
    // ヒストグラムのバーを描画
    histCtx.fillStyle = 'rgba(0, 255, 0, 0.8)';
    for (let i = 0; i < 256; i++) {
      const barHeight = grayValues[i] * scaleFactor;
      if (barHeight > 0) {
        histCtx.fillRect(
          i * barWidth,
          height - barHeight,
          barWidth,
          barHeight
        );
      }
    }
    
    // 選択範囲があれば表示
    if (selection.minBrightness >= 0 && selection.maxBrightness >= 0 && 
        selection.minBrightness !== selection.maxBrightness) {
      // 選択範囲を半透明の青色で表示
      histCtx.fillStyle = 'rgba(0, 100, 255, 0.3)';
      histCtx.fillRect(
        selection.minBrightness * barWidth,
        0,
        (selection.maxBrightness - selection.minBrightness + 1) * barWidth,
        height
      );
      
      // 選択範囲の境界線を描画
      histCtx.strokeStyle = 'rgba(0, 150, 255, 0.8)';
      histCtx.lineWidth = 2;
      histCtx.beginPath();
      
      // 左の境界線
      histCtx.moveTo(selection.minBrightness * barWidth, 0);
      histCtx.lineTo(selection.minBrightness * barWidth, height);
      
      // 右の境界線
      histCtx.moveTo((selection.maxBrightness + 1) * barWidth, 0);
      histCtx.lineTo((selection.maxBrightness + 1) * barWidth, height);
      
      histCtx.stroke();
      
      // 選択した明るさ範囲を表示
      histCtx.fillStyle = 'white';
      histCtx.font = '10px Arial';
      histCtx.fillText(
        `${selection.minBrightness}-${selection.maxBrightness}`,
        selection.minBrightness * barWidth + 2,
        12
      );
    }
    
    // ヒストグラムの外枠を描画
    histCtx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
    histCtx.lineWidth = 1;
    histCtx.strokeRect(0, 0, width, height);
  }
};

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
    const container = canvas.value?.parentElement?.parentElement;
    // 親要素のサイズを取得（window-contentクラスの要素）
    const containerWidth = container ? container.clientWidth - 16 : img.width; // 余白を少なく
    const containerHeight = container ? container.clientHeight - 16 : img.height;
    
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
      
      // ヒストグラム用の配列を初期化（0〜255の値の出現回数）
      const histogram = new Array(256).fill(0);
      
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
        
        // ヒストグラムにカウントを追加
        histogram[grayValue]++;
        
        // RGBすべてに同じ値を設定（グレースケール）
        data[i] = grayValue;     // R
        data[i + 1] = grayValue; // G
        data[i + 2] = grayValue; // B
        // Alpha値はそのまま
      }
      
      // 変換したデータをキャンバスに戻す
      ctx.putImageData(imageData, 0, 0);
      console.log('Grayscale conversion complete');
      
      // 元の画像データを保存
      originalImageData.value = imageData;
      
      // ヒストグラムデータを保存
      histogramData.value = histogram;
      
      // ヒストグラムを描画
      drawHistogramWithSelection(histogram);
      
      // 選択範囲がリセットされたことを確認
      selection.minBrightness = -1;
      selection.maxBrightness = -1;
      selection.active = false;
      
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
        // 選択範囲をリセット
        selection.minBrightness = -1;
        selection.maxBrightness = -1;
        selection.active = false;
        
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

.histogram-canvas {
  position: absolute;
  bottom: 0;
  right: 0;
  pointer-events: auto; /* マウスイベントを受け取れるように変更 */
  cursor: crosshair; /* カーソルを十字に変更 */
}
</style>