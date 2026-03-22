# eizo-auto-edit

「動画を自動編集して」の一言で、生素材をプロ品質の完成動画に仕上げる Claude Code Plugin。

Claude Code がオーケストレーター、eizo MCP Tools（60+）がタイムライン操作、Gemini 3.1 Pro が映像レビューを担当する**マルチモデルエージェントループ**。

## デモ動画

<!-- TODO: デモ動画のリンクを追加 -->
（準備中）

## 何ができるか

8 Phase の自動編集パイプラインをワンコマンドで実行します。

```
Phase 1: 素材分析（種別判定・プロジェクト設定取得）
  │
Phase 2: 無音カット
  │
Phase 3: 字幕生成 → AI補正 → スタイル適用
  │
Phase 4: 内容分析 → 強調ポイント特定
  │
Phase 5: カメラワーク配置（字幕連動）
  │
Phase 6: オーディオ演出（BGM + ダッキング / 効果音 / オーディオエフェクト）
  │
Phase 7: ビジュアル仕上げ（カラグレ / ビネット / テロップ / トランジション / フェード）
  │
Phase 8: Gemini レビュー → 自動改善ループ
        ├── エクスポート → Gemini 3.1 Pro が映像評価
        ├── スコア < 8.5 → 問題箇所を自動修正 → 再エクスポート
        └── スコア >= 8.5 → 完了
```

## アーキテクチャ

```
┌─────────────────────────────────────────────────┐
│  Claude Code (SKILL.md — 8 Phase パイプライン)     │
│  オーケストレーション・編集判断・自動改善            │
└──────┬──────────────┬──────────────┬─────────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────────┐ ┌──────────┐ ┌──────────────────┐
│ eizo MCP     │ │ eizo API │ │ Gemini 3.1 Pro   │
│ Tools (60+)  │ │ Server   │ │                  │
│              │ │          │ │ 映像レビュー       │
│ タイムライン   │ │ /video-  │ │ 構造化フィードバック│
│ 操作全般      │ │ review   │ │ スコアリング       │
└──────────────┘ └──────────┘ └──────────────────┘
```

## クイックスタート（審査員向け）

### Step 1: eizo デスクトップアプリをインストール

[**Releases ページ**](https://github.com/Taishi-Y/eizo-auto-edit/releases) から最新の DMG をダウンロード:

- **macOS (Apple Silicon)**: `eizo-0.2.1-arm64.dmg`

> アプリは Apple 公証済み（Notarized）です。ダブルクリックでそのまま開けます。

### Step 2: API キーを設定

eizo アプリを起動したら、**設定画面**（右上の歯車アイコン）から以下を設定:

| API キー | 用途 | 必須 |
|----------|------|------|
| **Anthropic API Key** | AI エージェント（自動編集の頭脳） | **必須** |
| **Gemini API Key** | Phase 8 の映像レビューループ | 推奨 |

- Anthropic API Key: https://console.anthropic.com で取得
- Gemini API Key: https://aistudio.google.com/apikey で取得

### Step 3: プラグインをインストール

[Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) を開き、以下を実行:

```bash
/plugin marketplace add Taishi-Y/eizo-auto-edit
/plugin install eizo-auto-edit@eizo-auto-edit-marketplace
```

### Step 4: 動画を自動編集

1. eizo アプリで動画ファイルをインポートし、タイムラインに配置
2. Claude Code で以下を入力:

```
動画を自動編集して
```

自動で8 Phase パイプラインが実行され、プロ品質の動画が完成します。

## 必要環境

| 項目 | 要件 |
|------|------|
| OS | macOS 12+（Apple Silicon） |
| CLI | [Claude Code](https://docs.anthropic.com/en/docs/claude-code) |
| API Key | Anthropic API Key（必須）、Gemini API Key（推奨） |

## 入出力

| | 内容 |
|---|---|
| **入力** | eizo タイムライン上の動画クリップ（トーキングヘッド、Vlog、スクリーンキャスト等） |
| **出力** | フル編集済み MP4（無音カット・字幕・カメラワーク・BGM・エフェクト適用済み）+ Gemini レビューレポート |

## 8 Phase パイプライン詳細

| Phase | 内容 | 使用ツール |
|-------|------|-----------|
| 1. 素材分析 | 素材種別判定、解像度・FPS取得 | `get_project_info`, `get_timeline_state` |
| 2. 無音カット | 音声分析、無音区間の自動検出・カット | `silence_cut` |
| 3. 字幕 | 音声認識→AI補正→スタイル適用 | `auto_subtitle`, `apply_subtitle_style_preset` |
| 4. 内容分析 | 字幕テキストから強調ポイントを特定 | 内部分析ロジック |
| 5. カメラワーク | 強調箇所にズーム・パン等を配置 | `add_camera_effect` |
| 6. オーディオ | BGM追加+ダッキング、効果音、エフェクト | `set_audio_ducking`, `add_audio_effect` |
| 7. ビジュアル | カラグレ、ビネット、テロップ、トランジション | `set_color_grading`, `add_transition`, `set_fade` |
| 8. レビュー | Gemini 3.1 Pro 映像評価→自動改善ループ | `export_video`, `/video-review` API |

## 使用技術

- **Claude Code** — オーケストレーション、編集判断の全体制御
- **eizo MCP Tools (60+)** — タイムライン操作（無音カット、字幕生成、カメラワーク、エフェクト等）
- **Gemini 3.1 Pro** — エクスポート動画の映像レビュー、構造化フィードバック生成
- **eizo Desktop App** — Electron ベースの動画編集アプリ（WebCodecs / WebGPU）

## プラグイン構成

```
plugins/eizo-auto-edit/
├── .claude-plugin/
│   └── plugin.json         # プラグインメタデータ
├── skills/
│   └── auto-edit/
│       └── SKILL.md        # 8 Phase パイプライン定義（1000+ 行）
└── README.md               # プラグイン説明
```

## ライセンス

MIT
