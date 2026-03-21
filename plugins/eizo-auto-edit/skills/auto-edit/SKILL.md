# eizo-auto-edit: 動画自動編集スキル

「動画を自動編集して」の一言で、生素材を公開可能な状態に仕上げる。
無音カット → 字幕生成 → ズーム演出 → エクスポート → Gemini映像レビュー → 自動改善ループを自律的に実行する。

## 前提条件

- eizo が起動中（`pnpm dev` で web + api が稼働）
- ブラウザで eizo が開かれ、タイムラインに動画クリップが配置されている
- eizo MCP Server が Claude Code から利用可能
- `apps/api/.env` に `GEMINI_API_KEY` が設定済み

## 全体フロー

```
Phase 1: 分析 → Phase 2: 編集 → Phase 3: エクスポート → Phase 4: レビュー & 改善
                                                              ↓
                                                         スコア不足なら
                                                              ↓
                                                    Phase 2 に戻って改善
                                                        (最大3ループ)
```

---

## Phase 1: 分析

セッションに接続し、素材を分析する。

### 手順

1. `eizo_list_sessions` でアクティブセッションを取得。1つなら自動選択、複数ならユーザーに確認。
2. `get_project_info` でプロジェクト情報を取得。
3. `get_timeline_state` でタイムライン状態を取得。
4. 以下を判定:
   - **素材種別**: トーキングヘッド / Vlog / スクリーンキャスト / その他
   - **総尺**: クリップの合計時間
   - **音声の有無**: オーディオトラックの存在
   - **既存編集**: 字幕・エフェクトが既にあるか

### 素材種別による設定

| 素材種別 | silence-cut | subtitle | zoom |
|----------|:-----------:|:--------:|:----:|
| トーキングヘッド | ON (threshold: -40dB, minSilence: 0.5s) | ON | ON |
| Vlog | ON (threshold: -45dB, minSilence: 0.8s) | ON | OFF |
| スクリーンキャスト | ON (threshold: -40dB, minSilence: 0.5s) | ON | OFF |
| その他 | ON (デフォルト) | ON | OFF |

判定が難しい場合はユーザーに確認する。

---

## Phase 2: 編集パイプライン

以下のモジュールを順番に実行する。各モジュールは前のモジュールの結果に依存する。

### モジュール 1: silence-cut（無音カット）

音声トラックが存在する場合のみ実行。

```
使用ツール: silence_cut
パラメータ:
  sessionId: <Phase 1で取得>
  threshold: -40  (素材種別により調整)
  minSilenceDuration: 0.5  (素材種別により調整)
  padding: 0.1
```

呼び出し例:
```
mcp__eizo__silence_cut({
  sessionId: "...",
  threshold: -40,
  minSilenceDuration: 0.5,
  padding: 0.1
})
```

実行後、`get_timeline_state` でカット結果を確認する。

### モジュール 2: subtitle（字幕生成 & スタイリング）

音声トラックが存在する場合のみ実行。silence-cut の後に実行すること。

```
Step 1: 自動字幕生成
使用ツール: auto_subtitle
パラメータ:
  sessionId: <Phase 1で取得>
  language: "ja" (素材から推定、デフォルト日本語)
  refine: true

Step 2: 字幕スタイル適用
使用ツール: apply_subtitle_style_preset
パラメータ:
  sessionId: <Phase 1で取得>
  preset: "youtube-default"
```

呼び出し例:
```
mcp__eizo__auto_subtitle({
  sessionId: "...",
  language: "ja",
  refine: true
})

mcp__eizo__apply_subtitle_style_preset({
  sessionId: "...",
  preset: "youtube-default"
})
```

### モジュール 3: zoom（強調ズーム）

トーキングヘッド素材の場合のみ実行。subtitle の後に実行すること。

ロジック:
1. `get_timeline_state` でタイムライン上の字幕データを取得
2. 字幕の間隔が長い箇所（発話の切り替わりポイント）を検出
3. 各ポイントで以下のズームアニメーションを追加:
   - ズームイン: scale 1.0 → 1.15（0.3秒, ease-in-out）
   - 保持: 2-5秒
   - ズームアウト: 1.15 → 1.0（0.3秒, ease-in-out）
4. 全体の20-30%程度にズームを配置（多すぎると逆効果）

```
各ズームポイントで:

Step 1: ズームイン開始キーフレーム
mcp__eizo__add_keyframe({
  sessionId: "...",
  clipId: "<対象クリップ>",
  property: "scale",
  time: <ズーム開始時間>,
  value: { x: 1.0, y: 1.0 },
  easing: "ease-in-out"
})

Step 2: ズームイン完了キーフレーム
mcp__eizo__add_keyframe({
  sessionId: "...",
  clipId: "<対象クリップ>",
  property: "scale",
  time: <ズーム開始時間 + 0.3>,
  value: { x: 1.15, y: 1.15 },
  easing: "ease-in-out"
})

Step 3: ズームアウト開始キーフレーム
mcp__eizo__add_keyframe({
  sessionId: "...",
  clipId: "<対象クリップ>",
  property: "scale",
  time: <ズーム終了時間 - 0.3>,
  value: { x: 1.15, y: 1.15 },
  easing: "ease-in-out"
})

Step 4: ズームアウト完了キーフレーム
mcp__eizo__add_keyframe({
  sessionId: "...",
  clipId: "<対象クリップ>",
  property: "scale",
  time: <ズーム終了時間>,
  value: { x: 1.0, y: 1.0 },
  easing: "ease-in-out"
})
```

---

## Phase 3: エクスポート

編集完了後、動画をエクスポートする。

```
mcp__eizo__export_video({
  sessionId: "...",
  format: "mp4",
  quality: "high"
})
```

エクスポート完了後、返されるファイルパスを Phase 4 で使用する。

---

## Phase 4: Gemini レビュー & 改善ループ

### レビューリクエスト

eizo API サーバーの `/video-review` エンドポイントにリクエストを送信する。

```bash
curl -X POST http://localhost:3001/video-review \
  -H "Content-Type: application/json" \
  -d '{
    "videoPath": "<Phase 3のエクスポートファイルパス>",
    "reviewCategories": [
      {
        "category": "pacing",
        "criteria": "不自然な間やテンポの悪い箇所がないか。カットが急すぎる・遅すぎる箇所。"
      },
      {
        "category": "subtitle",
        "criteria": "字幕の表示タイミングは適切か。読む時間は十分か。内容と音声が一致しているか。"
      },
      {
        "category": "visual_emphasis",
        "criteria": "ズームの配置は適切か。重要な発言にズームがあるか。ズームが多すぎ/少なすぎないか。単調になっていないか。"
      },
      {
        "category": "transition",
        "criteria": "カット間の繋がりは自然か。ジャンプカットが気になる箇所はないか。"
      },
      {
        "category": "overall_quality",
        "criteria": "映像全体として視聴者が最後まで見たいと思える仕上がりか。改善すべき最も重要な1点は何か。"
      }
    ],
    "context": {
      "sourceType": "<Phase 1で判定した素材種別>",
      "targetPlatform": "youtube",
      "language": "ja"
    }
  }'
```

### レスポンスの解釈

レスポンスは以下の形式:
```json
{
  "overallScore": 7.5,
  "categoryResults": {
    "pacing": { "score": 8, "issues": [...] },
    "subtitle": { "score": 6, "issues": [...] },
    ...
  },
  "done": false,
  "summary": "..."
}
```

### ループ終了条件

以下のいずれかを満たしたら終了:
1. `done === true`（overallScore >= 8.5 かつ全カテゴリ score >= 7）
2. ループ3回完了（改善が収束しない場合の打ち切り）
3. 前回と今回の overallScore の差が 0.3 未満（改善効果が頭打ち）

### フィードバック → アクション マッピング

Gemini のレビュー結果に含まれる issue を、対応する eizo MCP tool で修正する。

#### pacing（テンポ）

| issue | アクション |
|-------|----------|
| 不自然な間がある | `trim_clip` で該当タイムスタンプ付近のクリップを短縮 |
| カットが急すぎる | `silence_cut` を padding 増やして該当区間を再調整。または `undo` して結合 |

```
mcp__eizo__trim_clip({
  sessionId: "...",
  clipId: "<該当クリップ>",
  trimEnd: <短縮する秒数>
})
```

#### subtitle（字幕）

| issue | アクション |
|-------|----------|
| 表示時間が短い | `update_subtitle` で endTime を延長 |
| タイミングがずれている | `update_subtitle` で startTime/endTime を調整 |
| 誤字・誤認識 | `update_subtitle` で text を修正 |

```
mcp__eizo__update_subtitle({
  sessionId: "...",
  subtitleId: "<該当字幕ID>",
  endTime: <延長後の終了時間>
})

mcp__eizo__update_subtitle({
  sessionId: "...",
  subtitleId: "<該当字幕ID>",
  text: "<修正後テキスト>"
})
```

#### visual_emphasis（ズーム）

| issue | アクション |
|-------|----------|
| ズームがない箇所がある | `add_keyframe` で該当箇所にズーム追加（Phase 2 zoom と同じ手順） |
| ズームが多すぎる | severity: low のズームキーフレームを `remove_keyframe` で削除 |
| ズームが急すぎる | `update_keyframe` で duration を延長 |

```
mcp__eizo__remove_keyframe({
  sessionId: "...",
  clipId: "<該当クリップ>",
  keyframeId: "<削除するキーフレームID>"
})

mcp__eizo__update_keyframe({
  sessionId: "...",
  clipId: "<該当クリップ>",
  keyframeId: "<更新するキーフレームID>",
  time: <新しい時間>,
  value: { x: 1.15, y: 1.15 }
})
```

#### transition（トランジション）

| issue | アクション |
|-------|----------|
| 繋がりが不自然 | `add_transition` で該当クリップ間にクロスフェード追加 |

```
mcp__eizo__add_transition({
  sessionId: "...",
  clipId: "<該当クリップ>",
  type: "crossfade",
  duration: 0.3
})
```

### 改善実行後

改善を適用したら、再度 Phase 3（エクスポート）→ Phase 4（レビュー）を繰り返す。

---

## エラーハンドリング

### セッション接続失敗
- `eizo_list_sessions` が空の場合: 「eizo でプロジェクトを開いてからもう一度お試しください」とユーザーに伝える

### MCP tool 呼び出し失敗
- リトライ1回。それでも失敗したら該当モジュールをスキップし、次のモジュールに進む
- スキップしたモジュールはユーザーに報告する

### Gemini レビュー失敗
- API エラーの場合: リトライ1回
- タイムアウトの場合: レビューなしで完了とし、ユーザーに手動確認を依頼
- GEMINI_API_KEY 未設定: レビューをスキップし、Phase 2 の編集結果のみで完了

### エクスポート失敗
- エラー内容をユーザーに表示し、手動エクスポートを案内

---

## ユーザーへの報告

各フェーズ完了時に進捗を報告する:

```
Phase 1 完了: 「素材を分析しました。トーキングヘッド動画（3分24秒）、音声あり。」
Phase 2 完了: 「編集が完了しました。無音カット（12箇所）、字幕生成（45件）、ズーム演出（8箇所）を適用。」
Phase 3 完了: 「エクスポート中...完了しました。」
Phase 4 完了: 「Geminiレビュー結果: 総合スコア 7.2/10。subtitle(6.5)に改善の余地あり。改善を実行します。」
ループ終了: 「最終スコア: 8.7/10（3回の改善ループ）。全カテゴリが基準を満たしました。」
```
