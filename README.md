# Image Analyzer

画像の分析を簡単に行うためのアプリケーションです。
ドラッグ&ドロップで画像を分析することができます。


## 機能

- **元画像表示**: アップロードした画像をそのまま表示
- **明るさ分析**: 画像をグレースケールに変換し、明度ヒストグラムを表示
    - グレースケールにはxyz表色系のYを採用している。そのため、実際の見え方に近いグレースケールとなる。
- **PCCS色空間分析（開発中）**: PCCS（Practical Color Coordinate System）に基づく色調分析
- **彩度分析（開発中）**: HSV色空間での彩度成分を視覚化






## 使用方法

1. 画像をドラッグ&ドロップまたはファイル選択でアップロード
2. 4つのビューで同時に画像を分析
3. 右下の「画像クリア」ボタンで画像をリセット

また、以下のURLから試すことが可能です。
https://chikuwakun.github.io/imageAnalyzer/

## 開発用

```bash
# 依存関係のインストール
npm install

# 開発サーバーの起動
npm run dev

# ビルド
npm run build
```
