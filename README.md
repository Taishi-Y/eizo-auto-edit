# eizo-auto-edit

「動画を自動編集して」の一言で、生素材を公開可能な状態に仕上げる Claude Code Plugin。

Claude Code がオーケストレーター、eizo MCP Tools がタイムライン操作、Gemini 3.1 Pro が映像レビューを担当するマルチモデルエージェントループ。

## 何ができるか

1. **無音カット** — 音声を分析し、無音区間を自動検出・カット
2. **字幕生成** — 音声認識で字幕を自動生成し、YouTube向けスタイルを適用
3. **ズーム演出** — トーキングヘッド動画で、発話の切り替わりに自動ズーム
4. **エクスポート** — MP4で書き出し
5. **Gemini 映像レビュー** — エクスポート動画を Gemini 3.1 Pro が視聴・評価
6. **自動改善ループ** — スコアが基準未満なら自動修正して再レビュー（最大3回）

## アーキテクチャ

```
Claude Code (SKILL.md)
  ├── eizo MCP Tools … タイムライン操作（無音カット、字幕、ズーム等）
  ├── eizo API Server … /video-review エンドポイント
  └── Gemini 3.1 Pro  … 映像評価（構造化JSON出力）
```

## インストール

```bash
/plugin marketplace add Taishi-Y/eizo-auto-edit
/plugin install eizo-auto-edit@eizo-auto-edit-marketplace
```

## 使い方

### 前提条件

- [eizo](https://eizo.ai) が起動中（`pnpm dev` で web + api が稼働）
- ブラウザで eizo を開き、タイムラインに動画クリップを配置
- eizo MCP Server が Claude Code から利用可能
- `apps/api/.env` に `GEMINI_API_KEY` を設定

### 実行

Claude Code で以下を入力:

```
動画を自動編集して
```

自動で分析 → 編集 → エクスポート → レビュー → 改善のフルパイプラインが実行されます。

## 入出力

| | 内容 |
|---|---|
| **入力** | eizo タイムライン上の動画クリップ |
| **出力** | 無音カット・字幕・ズーム済みの MP4 + Gemini レビューレポート |

## デモ動画

（準備中）

## 使用技術

- **Claude Code** — オーケストレーション、編集判断
- **eizo MCP Tools** — タイムライン操作
- **Gemini 3.1 Pro** — 映像レビュー、構造化フィードバック生成
- **Hono API** — `/video-review` エンドポイント

## ライセンス

MIT
