# Image Analyzer

画像を4つの異なる視点で分析するWebアプリケーションです。

## 機能

- **元画像表示**: アップロードした画像をそのまま表示
- **明るさ分析**: 画像をグレースケールに変換し、明度ヒストグラムを表示
- **PCCS色空間分析**: PCCS（Practical Color Coordinate System）に基づく色調分析
- **彩度分析**: HSV色空間での彩度成分を視覚化

## 使用方法

1. 画像をドラッグ&ドロップまたはファイル選択でアップロード
2. 4つのビューで同時に画像を分析
3. 右下の「画像クリア」ボタンで画像をリセット

## 技術スタック

- Vue 3 + TypeScript + Vite
- Canvas API for image processing
- CSS Grid Layout
- GitHub Pages for deployment

## 開発

```bash
# 依存関係のインストール
npm install

# 開発サーバーの起動
npm run dev

# プロダクション用ビルド
npm run build
```

## デプロイ

このプロジェクトはGitHub Actionsを使用してGitHub Pagesに自動デプロイされます。`main`ブランチにプッシュすると自動的にデプロイが実行されます。
