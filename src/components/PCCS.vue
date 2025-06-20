<template>
  <div class="fullsize-container">
    <canvas ref="canvas" class="fullsize-canvas"></canvas>
    <div v-if="isLoading" class="loading-overlay">
      <div class="loading-spinner"></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import pccsImage from '../assets/pccs.png';

const props = defineProps<{
  imageUrl: string | null;
}>();

const canvas = ref<HTMLCanvasElement | null>(null);
const isLoading = ref(false);
const pccsChartLoaded = ref(false);

interface ImageDataType {
  originalImage: HTMLImageElement | null;
  originalCanvas: HTMLCanvasElement | null;
  pixelTones: Record<number, string>;
  tonePixels: Record<string, number[]>;
  lastOffsetX: number;
  lastOffsetY: number;
  lastScale: number;
}

const imageData = ref<ImageDataType>({
  originalImage: null,
  originalCanvas: null,
  pixelTones: {},
  tonePixels: {},
  lastOffsetX: 0,
  lastOffsetY: 0,
  lastScale: 0
});

// トーンの種類と位置情報 (元の位置と新しい位置の間で1:4の比率に調整)
const toneInfo = {
  v: { x: 614, y: 350, radius: 48 },  // vivid
  b: { x: 448, y: 194, radius: 48 },  // bright
  s: { x: 448, y: 350, radius: 48 },  // strong
  dp: { x: 448, y: 505, radius: 48 }, // deep
  lt: { x: 284, y: 115, radius: 48 }, // light
  sf: { x: 284, y: 271, radius: 48 }, // soft
  d: { x: 284, y: 426, radius: 48 },  // dull
  dk: { x: 284, y: 580, radius: 48 }, // dark
  p: { x: 120, y: 115, radius: 48 },  // pale
  ltg: { x: 120, y: 271, radius: 48 }, // light grayish
  g: { x: 120, y: 426, radius: 48 },   // grayish
  dkg: { x: 120, y: 580, radius: 48 }, // dark grayish
};

// RGB値をLab色空間へ変換する関数
const rgbToLab = (r: number, g: number, b: number) => {
  // RGB → XYZ変換
  r = r / 255;
  g = g / 255;
  b = b / 255;
  
  // sRGBのガンマ補正を解除
  r = r > 0.04045 ? Math.pow((r + 0.055) / 1.055, 2.4) : r / 12.92;
  g = g > 0.04045 ? Math.pow((g + 0.055) / 1.055, 2.4) : g / 12.92;
  b = b > 0.04045 ? Math.pow((b + 0.055) / 1.055, 2.4) : b / 12.92;
  
  // XYZ色空間への変換
  // sRGB D65参照白色
  const x = r * 0.4124 + g * 0.3576 + b * 0.1805;
  const y = r * 0.2126 + g * 0.7152 + b * 0.0722;
  const z = r * 0.0193 + g * 0.1192 + b * 0.9505;
  
  // XYZ → Lab 変換
  // D65参照白色の正規化値
  const xn = 0.95047;
  const yn = 1.0;
  const zn = 1.08883;
  
  // 正規化
  const xyz = [x / xn, y / yn, z / zn].map(v => {
    return v > 0.008856 ? Math.pow(v, 1/3) : (7.787 * v) + 16/116;
  });
  
  // Lab値の計算
  const l = (116 * xyz[1]) - 16;  // L: 明度 (0-100)
  const a = 500 * (xyz[0] - xyz[1]); // a: 赤-緑 軸
  const bb = 200 * (xyz[1] - xyz[2]); // b: 黄-青 軸
  
  return { l, a, b: bb };
};

// LabからHCL色空間（色相、彩度、明度）へ変換する関数
const labToHcl = (lab: { l: number, a: number, b: number }) => {
  // HCL（色相、彩度、明度）に変換
  const h = Math.atan2(lab.b, lab.a) * (180 / Math.PI); // 色相 (0-360)
  const c = Math.sqrt(lab.a * lab.a + lab.b * lab.b);   // 彩度
  const l = lab.l;                                      // 明度 (0-100)
  
  return {
    h: h < 0 ? h + 360 : h, // 角度を0-360の範囲に正規化
    c,
    l
  };
};

// HCL値からトーンを判定する関数
const determineColorTone = (hcl: { h: number, c: number, l: number }) => {
  const { c, l } = hcl;
  
  // PCCSトーン分類の境界値
  // Lab/HCL空間でのトーン分類境界値
  // L: 明度 (0-100), C: 彩度
  
  // 明度に基づくトーン分類
  if (l >= 85) {
    if (c < 15) return 'w';  // white (白)
    if (c < 30) return 'p';  // pale (淡い)
    return 'v';              // vivid (鮮やかな)
  } else if (l >= 70) {
    if (c < 10) return 'ltg'; // light grayish (明るい灰色)
    if (c < 35) return 'lt';  // light (明るい)
    return 'b';               // bright (明るく鮮やかな)
  } else if (l >= 50) {
    if (c < 10) return 'g';   // grayish (灰色)
    if (c < 35) return 'sf';  // soft (柔らかい)
    return 's';               // strong (鮮やか)
  } else if (l >= 30) {
    if (c < 10) return 'dkg'; // dark grayish (暗い灰色)
    if (c < 30) return 'd';   // dull (くすんだ)
    return 'dp';              // deep (深い)
  } else {
    if (c < 15) return 'Bk';  // black (黒)
    return 'dk';              // dark (暗い)
  }
};

// トーンのカウントを初期化する関数
const initToneCounts = () => {
  return {
    v: 0, b: 0, s: 0, dp: 0,
    lt: 0, sf: 0, d: 0, dk: 0,
    p: 0, ltg: 0, g: 0, dkg: 0,
    w: 0, Bk: 0, total: 0
  };
};

// PCCS色相環とトーンチャートを描画する関数
const drawPccsChart = () => {
  if (!canvas.value) return;

  const ctx = canvas.value.getContext('2d');
  if (!ctx) return;

  // コンテナのサイズを取得（パディングを考慮しない）
  const container = canvas.value.parentElement?.parentElement;
  const containerWidth = container ? container.clientWidth : 300;
  const containerHeight = container ? container.clientHeight : 300;

  canvas.value.width = containerWidth;
  canvas.value.height = containerHeight;

  // 背景を透明にする
  ctx.clearRect(0, 0, containerWidth, containerHeight);

  const img = new Image();
  img.src = pccsImage;

  img.onload = () => {
    console.log('PCCS画像読み込み完了:', img.width, 'x', img.height);

    // 画像がはみ出ないようにするために Math.min を使用
    const scale = Math.min(
      containerWidth / img.width,
      containerHeight / img.height
    );

    const width = img.width * scale;
    const height = img.height * scale;

    // 中央に配置
    const x = (containerWidth - width) / 2;
    const y = (containerHeight - height) / 2;

    // キャンバスをクリアして透明にする
    ctx.clearRect(0, 0, containerWidth, containerHeight);
    
    // 画像を描画
    ctx.drawImage(img, x, y, width, height);
    pccsChartLoaded.value = true;
    console.log('PCCSトーンチャート表示完了');

    // 画像URLがある場合は画像分析結果も表示
    if (props.imageUrl) {
      analyzeAndVisualizeImage(props.imageUrl, ctx, x, y, scale);
    }
  };

  img.onerror = (e) => {
    console.error('PCCSトーン画像の読み込みに失敗:', e);
  };
};

// 画像を分析し、トーン統計を視覚化する関数
const analyzeAndVisualizeImage = async (url: string, ctx: CanvasRenderingContext2D, offsetX: number, offsetY: number, scale: number) => {
  try {
    // 画像を読み込む
    const img = new Image();
    img.crossOrigin = "Anonymous";
    
    await new Promise<void>((resolve, reject) => {
      img.onload = () => resolve();
      img.onerror = reject;
      img.src = url;
    });
    
    // 一時キャンバスで画像の分析
    const tempCanvas = document.createElement('canvas');
    const tempCtx = tempCanvas.getContext('2d');
    if (!tempCtx) {
      throw new Error('一時キャンバスのコンテキストを取得できません');
    }
    
    tempCanvas.width = img.width;
    tempCanvas.height = img.height;
    tempCtx.drawImage(img, 0, 0);
    
    const imgDataObj = tempCtx.getImageData(0, 0, img.width, img.height);
    const data = imgDataObj.data;
    
    // トーンのカウントを初期化
    const toneCounts = initToneCounts();
    
    // ピクセルごとのトーンマッピングを初期化
    const pixelTones: Record<number, string> = {};
    const tonePixels: Record<string, number[]> = {};
    Object.keys(toneInfo).forEach(tone => {
      tonePixels[tone] = [];
    });
    tonePixels['w'] = [];
    tonePixels['Bk'] = [];
    
    // ピクセルを分析してトーンをカウント
    for (let i = 0; i < data.length; i += 4) {
      const pixelIndex = i / 4;
      const r = data[i];
      const g = data[i + 1];
      const b = data[i + 2];
      const a = data[i + 3];
      
      // 透明ピクセルはスキップ
      if (a === 0) continue;
      
      // RGB値をLab値に変換
      const lab = rgbToLab(r, g, b);
      
      // LabをHCL値に変換
      const hcl = labToHcl(lab);
      
      // HCL値からトーンを判定
      const tone = determineColorTone(hcl);
      
      // トーンをカウント
      if (tone in toneCounts) {
        toneCounts[tone as keyof typeof toneCounts]++;
        toneCounts.total++;
        
        // ピクセルとトーンのマッピングを保存
        pixelTones[pixelIndex] = tone;
        if (tonePixels[tone]) {
          tonePixels[tone].push(pixelIndex);
        }
      }
    }
    
    console.log('トーン分析結果 (RGB→Lab→HCL):', toneCounts);
    
    // 後で使うために元の画像データを保存
    imageData.value.originalImage = img;
    imageData.value.originalCanvas = tempCanvas;
    imageData.value.pixelTones = pixelTones;
    imageData.value.tonePixels = tonePixels;
    imageData.value.lastOffsetX = offsetX;
    imageData.value.lastOffsetY = offsetY;
    imageData.value.lastScale = scale;
    
    // トーン統計を視覚化
    visualizeToneStatistics(ctx, toneCounts, offsetX, offsetY, scale);
    
  } catch (error) {
    console.error('画像分析エラー:', error);
  }
};

// トーン統計を視覚化する関数
const visualizeToneStatistics = (ctx: CanvasRenderingContext2D, toneCounts: Record<string, number>, offsetX: number, offsetY: number, scale: number) => {
  // トーンの総数
  const totalPixels = toneCounts.total;
  if (totalPixels === 0) return;
  
  // PCCSチャートに重ねて円を描画
  Object.entries(toneInfo).forEach(([tone, info]) => {
    const count = toneCounts[tone as keyof typeof toneCounts] || 0;
    const percentage = count / totalPixels;
    
    // 表示サイズを計算（割合に基づく）
    const maxOverlayRadius = info.radius * 0.7; // 最大サイズを制限
    const overlayRadius = Math.sqrt(percentage) * maxOverlayRadius * 3; // 面積が比率に比例するよう平方根をとる
    
    // 実際の描画位置をスケーリング
    const x = offsetX + info.x * scale;
    const y = offsetY + info.y * scale;
    
    // 比率に基づく円を描画
    if (percentage > 0.01) { // 1%以上の場合のみ描画
      ctx.beginPath();
      ctx.arc(x, y, overlayRadius, 0, Math.PI * 2);
      ctx.fillStyle = `rgba(30, 144, 255, 0.3)`; // 青色（ドジャーブルー）の半透明
      ctx.fill();
      ctx.strokeStyle = 'rgba(0, 94, 184, 0.8)'; // 濃い青色
      ctx.lineWidth = 2 * scale;
      ctx.stroke();
    }
  });
};

// 画像処理とPCCS色空間へのマッピング
const processImage = async (url: string) => {
  console.log('画像処理開始:', url.substring(0, 30) + '...');
  isLoading.value = true;

  if (!canvas.value) {
    console.error('キャンバス要素がありません');
    isLoading.value = false;
    return;
  }

  // PCCSチャートを描画（画像分析と視覚化は内部で行われる）
  drawPccsChart();
  isLoading.value = false;
};

// リサイズハンドラ
const onResize = () => {
  drawPccsChart();
};

// 画像URLが変更されたときに処理を実行
watch(() => props.imageUrl, (newUrl) => {
  if (newUrl) {
    setTimeout(() => {
      processImage(newUrl);
    }, 200);
  } else {
    drawPccsChart();
  }
});

// マウント時の処理
onMounted(() => {
  console.log('PCCSビューコンポーネントがマウントされました');
  window.addEventListener('resize', onResize);
  setTimeout(() => {
    drawPccsChart();
    if (props.imageUrl) {
      processImage(props.imageUrl);
    }
  }, 300);
});

// アンマウント時の処理
onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize);
});
</script>

<style scoped>
.fullsize-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  position: relative;
  background-color: transparent;
}

.fullsize-canvas {
  max-width: 100%;
  max-height: 100%;
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top: 4px solid #ffffff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>