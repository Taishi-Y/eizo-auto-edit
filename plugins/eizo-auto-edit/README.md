# eizo-auto-edit Plugin

動画を自動編集し、Gemini の映像レビューで品質を自動改善する Claude Code Plugin。

## セットアップ

### 前提条件

- [eizo](https://github.com/taishiyamasaki/eizo) がクローン済み
- Node.js 20+, pnpm がインストール済み
- Claude Code CLI がインストール済み

### 1. 依存のインストール

```bash
cd eizo
pnpm install
```

### 2. 環境変数の設定

```bash
cp apps/api/.env.example apps/api/.env
```

`apps/api/.env` に以下を設定:

```
GEMINI_API_KEY=your-gemini-api-key
```

### 3. eizo の起動

```bash
pnpm dev
```

ブラウザで http://localhost:5173 を開き、動画をインポートしてタイムラインに配置する。

### 4. Claude Code で使用

```bash
claude
```

Claude Code のプロンプトで:

```
動画を自動編集して
```

## 動作フロー

1. **分析**: セッション接続、素材種別の判定
2. **編集**: 無音カット → 字幕生成 → ズーム演出
3. **エクスポート**: MP4で書き出し
4. **レビュー**: Gemini が映像を評価し、問題点をフィードバック
5. **改善**: フィードバックに基づき自動修正（最大3ループ）

## 使用技術

- **Claude Code**: オーケストレーション、編集判断
- **eizo MCP Tools**: タイムライン操作（無音カット、字幕、ズーム等）
- **Gemini 3.1 Pro**: エクスポート動画の映像レビュー、構造化フィードバック生成
- **eizo API**: `/video-review` エンドポイント経由で Gemini と通信

## アーキテクチャ

```
Claude Code (SKILL.md)
  ├── eizo MCP Tools … タイムライン操作
  ├── eizo API Server … /video-review エンドポイント
  └── Gemini 3.1 Pro  … 映像評価（API経由）
```

## ライセンス

MIT
