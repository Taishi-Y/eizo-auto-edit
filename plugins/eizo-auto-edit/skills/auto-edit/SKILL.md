# eizo-auto-edit: プロ品質 AI 自動動画編集スキル

「動画を自動編集して」の一言で、生素材をプロ品質の完成動画に仕上げる。
無音カット → 字幕生成＋AI補正 → 内容分析 → カメラワーク → オーディオ演出 → ビジュアル仕上げ → Geminiレビュー → 自動改善ループ の 8 Phase パイプラインを自律的に実行する。

## 前提条件

- eizo が起動中（`pnpm dev` で web + api が稼働）
- ブラウザで eizo が開かれ、タイムラインに動画クリップが配置されている
- eizo MCP Server が Claude Code から利用可能
- `apps/api/.env` に `GEMINI_API_KEY` が設定済み

## 全体フロー

```
Phase 1: 素材分析
  │
Phase 2: 無音カット
  │
Phase 3: 字幕生成 → AI補正 → スタイル適用
  │
Phase 4: 内容分析 → 強調ポイント特定
  │
Phase 5: カメラワーク配置（字幕連動）
  │
Phase 6: オーディオ演出
  │     ├── 6a: BGM + ダッキング
  │     ├── 6b: 効果音（SE）
  │     └── 6c: オーディオエフェクト
  │
Phase 7: ビジュアル仕上げ
  │     ├── 7a: カラーグレーディング
  │     ├── 7b: ビネット
  │     ├── 7c: テキストオーバーレイ
  │     ├── 7d: トランジション
  │     └── 7e: フェード
  │
Phase 8: Gemini レビュー → 自動改善ループ
        │
        ├── エクスポート → Gemini 評価
        ├── スコア < 8.5 → 問題箇所を自動修正 → 再エクスポート
        └── スコア >= 8.5 → 完了
```

---

## Phase 1: 素材分析

セッションに接続し、素材を分析する。

### 手順

1. `eizo_list_sessions` でアクティブセッションを取得。1つなら自動選択、複数ならユーザーに確認。
2. `get_project_info` でプロジェクト情報を取得。
3. `get_timeline_state` でタイムライン状態を取得。
4. 以下を判定:
   - **素材種別**: トーキングヘッド / Vlog / スクリーンキャスト / プレゼン / その他
   - **解像度・FPS**: プロジェクト設定から取得
   - **総尺**: クリップの合計時間
   - **音声の有無**: オーディオトラックの存在
   - **既存編集**: 字幕・エフェクトが既にあるか

### 呼び出し例

```
mcp__eizo__eizo_list_sessions()

mcp__eizo__get_project_info({
  sessionId: "..."
})

mcp__eizo__get_timeline_state({
  sessionId: "..."
})
```

### 素材種別による Phase 設定一覧

| 素材種別 | silence-cut | subtitle | camera | audio | visual |
|----------|:-----------:|:--------:|:------:|:-----:|:------:|
| トーキングヘッド | ON | ON | ON | ON | ON |
| Vlog | ON | ON | 控えめ | ON | ON |
| スクリーンキャスト | ON | ON | 最小限 | ON | ON |
| プレゼン | ON | ON | 控えめ | ON | ON |
| その他 | ON | ON | OFF | ON | ON |

判定が難しい場合はユーザーに確認する。

---

## Phase 2: 無音カット

音声トラックが存在する場合のみ実行。

### 素材種別別パラメータ

| パラメータ | トーキングヘッド | Vlog | スクリーンキャスト | プレゼン |
|---|---|---|---|---|
| sensitivity | 0.5 | 0.6 | 0.5 | 0.45 |
| minSilenceDuration | 0.5 | 0.8 | 0.5 | 0.6 |
| paddingBefore | 0.1 | 0.15 | 0.1 | 0.1 |
| paddingAfter | 0.1 | 0.15 | 0.1 | 0.1 |

### 呼び出し例

```
mcp__eizo__silence_cut({
  sessionId: "...",
  sensitivity: 0.5,
  minSilenceDuration: 0.5,
  paddingBefore: 0.1,
  paddingAfter: 0.1
})
```

実行後、`get_timeline_state` でカット結果を確認する。

---

## Phase 3: 字幕生成 & AI 補正

音声トラックが存在する場合のみ実行。Phase 2（無音カット）の後に実行すること。

### Step 1: 自動文字起こし

```
mcp__eizo__auto_subtitle({
  sessionId: "...",
  language: "ja",
  refine: true
})
```

### Step 2: 字幕テキスト取得

```
mcp__eizo__export_srt({
  sessionId: "..."
})
```

レスポンスから全字幕のテキストとタイムスタンプを取得する。

### Step 3: Claude による字幕補正

export_srt で取得した字幕テキストを Claude が分析し、以下を補正する:

| 補正項目 | 説明 | 例 |
|---|---|---|
| **固有名詞修正** | 音声認識の誤認識を文脈から修正 | 「オープンクロー」→「Open Claude」 |
| **文節区切り修正** | 日本語として自然な位置で改行 | 「今日は天気が」→「今日は / 天気がいいので」 |
| **フィラー除去** | えーと、あのー等を削除 | 「えーとですね」→ 削除 or 前後の字幕に統合 |
| **句読点追加** | 適切な句読点を補完 | 「これはですね」→「これはですね、」 |
| **表記統一** | カタカナ/英語の統一 | 「マックブック」→「MacBook」 |

**補正ロジック:**
1. 全字幕テキストを通読し、動画の文脈・トピックを把握する
2. 固有名詞のリストを作成し、出現箇所を統一する
3. 各字幕を自然な文節で区切り直す
4. フィラーを除去し、前後の字幕と統合する
5. 補正後の全字幕データを配列として構築する

### Step 4: 補正済み字幕を一括置換

```
mcp__eizo__replace_all_subtitles({
  sessionId: "...",
  subtitles: [
    { "startTime": 0.5, "endTime": 2.3, "text": "補正後のテキスト1" },
    { "startTime": 2.5, "endTime": 4.8, "text": "補正後のテキスト2" },
    ...
  ]
})
```

### Step 5: 字幕スタイル適用

```
mcp__eizo__apply_subtitle_style_preset({
  sessionId: "...",
  preset: "youtube-default"
})
```

必要に応じて個別スタイル調整:

```
mcp__eizo__set_subtitle_style({
  sessionId: "...",
  subtitleId: "...",
  style: {
    fontSize: 24,
    fontColor: "#FFFFFF",
    backgroundColor: "rgba(0,0,0,0.6)",
    strokeColor: "#000000",
    strokeWidth: 1
  }
})
```

### 字幕スタイル基準

| スタイル要素 | 通常字幕 | 強調字幕 |
|---|---|---|
| フォントサイズ | 24px | 32-36px |
| 色 | 白 (#FFFFFF) | 黄 (#FFFF00) |
| 背景 | rgba(0,0,0,0.6) | rgba(0,0,0,0.85) |
| ストローク | 黒 1px | 黒 2-3px |
| シャドウ | なし | blur: 6, offset: 2 |

---

## Phase 4: 内容分析 & 強調ポイント特定

Claude が Phase 3 で補正済みの字幕テキスト全体を分析し、各字幕に強調レベルをタグ付けする。

### 判定基準

| ポイント種別 | 判定基準 | 強調レベル |
|---|---|---|
| **メインテーマ発表** | 動画の目的・テーマを述べている箇所 | high |
| **重要キーワード** | 製品名、数値、結論に関わる単語 | high |
| **ユーモア/オチ** | 笑いを狙った箇所、意外な展開 | medium |
| **話題の転換** | 新しいトピックに移る瞬間 | medium |
| **結論・まとめ** | 最後のまとめ、CTA | high |
| **感情的な発言** | 強い意見、驚き、喜び | medium-high |
| **通常の発話** | 上記に該当しない | none or low |

### 出力形式

各字幕に以下のタグを付与する（Claude 内部処理、後続 Phase で参照）:

```json
[
  {
    "subtitleId": "sub-001",
    "startTime": 0.5,
    "endTime": 2.3,
    "text": "今日はOpen Claudeについて紹介します",
    "emphasis_level": "high",
    "emphasis_type": "theme_intro"
  },
  {
    "subtitleId": "sub-002",
    "startTime": 2.5,
    "endTime": 4.8,
    "text": "まず基本的な使い方から見ていきましょう",
    "emphasis_level": "low",
    "emphasis_type": "normal"
  },
  {
    "subtitleId": "sub-003",
    "startTime": 10.0,
    "endTime": 12.5,
    "text": "ここが一番面白いところなんですが",
    "emphasis_level": "medium",
    "emphasis_type": "humor"
  }
]
```

### 制約

- 強調ポイント（medium 以上）は全体の **20-30%** に収める（多すぎると効果が薄れる）
- high は全体の **10%以下** が目安
- 話題転換ポイントは Phase 7d のトランジション配置にも使用する

---

## Phase 5: カメラワーク（字幕連動）

Phase 4 の強調レベルに基づき、`add_camera_effect` でカメラエフェクトを配置する。

### 強調レベル → カメラエフェクト マッピング

| 強調レベル | カメラエフェクト | パラメータ |
|---|---|---|
| **none** | **なし（静止）** | カメラエフェクトを付けない |
| **low** | **なし（静止）** | カメラエフェクトを付けない |
| **medium** | zoom-in | scale: 1.15-1.2, ease-out |
| **high** | impact-zoom または zoom-in | scale: 1.25-1.3, ease-out, 0.3s |
| **話題転換** | pan（方向交互） | 視線の移動を表現 |
| **結論** | slow-zoom-out | scale: 1.2→1.0, ease-in-out |

### 呼び出し例

```
// none / low — カメラエフェクトなし（静止のまま）
// → add_camera_effect を呼ばない。静止がデフォルト。

// medium — ズームイン
mcp__eizo__add_camera_effect({
  sessionId: "...",
  clipId: "...",
  type: "zoom-in",
  startTime: 10.0,
  duration: 2.0,
  params: {
    scale: 1.2,
    easing: "ease-out"
  }
})

// high — インパクトズーム
mcp__eizo__add_camera_effect({
  sessionId: "...",
  clipId: "...",
  type: "impact-zoom",
  startTime: 15.0,
  duration: 0.3,
  params: {
    scale: 1.3,
    easing: "ease-out"
  }
})

// 結論 — スローズームアウト
mcp__eizo__add_camera_effect({
  sessionId: "...",
  clipId: "...",
  type: "slow-zoom-out",
  startTime: 50.0,
  duration: 5.0,
  params: {
    scale: 1.2,
    easing: "ease-in-out"
  }
})
```

### カメラワークのルール

- **none / low レベルにはカメラエフェクトを付けない** — 静止がデフォルト
- カメラモーションは **medium 以上の強調ポイントのみ** に配置する
- カメラモーションを付ける箇所は全体の **15-20% 以下** に抑える
- 同じエフェクトが **連続しない**（交互に変化させる）
- 各エフェクト間に十分な静止区間を挟み、メリハリをつける
- エフェクトの duration は **字幕の duration に合わせる**
- 迷ったら「付けない」を選ぶ — 少ないほど強調が際立つ

---

## Phase 6: オーディオ演出

eizo CDN (`https://eizo-assets.web.app`) のテンプレート素材を使用する。
全ての素材はロイヤリティフリー。`import_media_from_url` で CDN から直接インポートし、MCP ツールのみで操作する。

### 6a: BGM 追加 & ダッキング

#### BGM テンプレート一覧（CDN）

素材種別・動画の雰囲気に応じて選択する:

| BGM | CDN URL | 用途 |
|---|---|---|
| AlgorithmicFlow (Main) | `https://eizo-assets.web.app/audio/bgm/MA_LEXMusic_AlgorithmicFlow_Main.mp3` | 落ち着いたトーク、解説系 |
| AlgorithmicFlow (15s) | `https://eizo-assets.web.app/audio/bgm/MA_LEXMusic_AlgorithmicFlow_15s.mp3` | 短尺動画、イントロ/アウトロ用 |
| QuantumElevation (Main) | `https://eizo-assets.web.app/audio/bgm/MA_LEXMusic_QuantumElevation_Main.mp3` | エネルギッシュ、テンポの速い動画 |
| QuantumElevation (Short) | `https://eizo-assets.web.app/audio/bgm/MA_LEXMusic_QuantumElevation_Short.mp3` | 短尺動画、テンポの速い動画 |
| VioletStones | `https://eizo-assets.web.app/audio/bgm/MA_LexMusic_VioletStones.mp3` | ムーディー、Vlog 向け |
| EchoesOfTheFuture (Main) | `https://eizo-assets.web.app/audio/bgm/MA_RoomanProduction_EchoesOfTheFuture_main.mp3` | シネマティック、ドラマチック |
| EchoesOfTheFuture (Loop) | `https://eizo-assets.web.app/audio/bgm/MA_RoomanProduction_EchoesOfTheFuture_loop.mp3` | 長尺のループBGM |
| StartFresh (15s) | `https://eizo-assets.web.app/audio/bgm/MA_DiegoMartinez_StartFresh_15sec.mp3` | 短尺、明るいイントロ |
| Beyond Neon Stars | `https://eizo-assets.web.app/audio/bgm/AncepScore%20-%20Beyond%20Neon%20Stars.mp3` | エレクトロニック、テック系 |

#### 素材種別 → BGM 選択ガイド

| 素材種別 | 推奨 BGM | 理由 |
|---|---|---|
| トーキングヘッド | AlgorithmicFlow (Main) | 控えめで発話を邪魔しない |
| Vlog | VioletStones | ムーディーで雰囲気が出る |
| スクリーンキャスト | AlgorithmicFlow (Main) | 落ち着いたテンポ |
| プレゼン | QuantumElevation (Main) | エネルギッシュで前向き |
| Shorts/TikTok | StartFresh (15s) or QuantumElevation (Short) | 短尺に最適化 |

#### BGM インポート & 配置

```
// Step 1: CDN から BGM をインポート
mcp__eizo__import_media_from_url({
  sessionId: "...",
  url: "https://eizo-assets.web.app/audio/bgm/MA_LEXMusic_AlgorithmicFlow_Main.mp3",
  fileName: "bgm-algorithmic-flow.mp3"
})
// → レスポンスの mediaId を記録

// Step 2: タイムラインに配置（新しいオーディオトラックに追加）
mcp__eizo__add_clip_from_media({
  sessionId: "...",
  mediaId: "<インポートで取得した mediaId>",
  trackIndex: -1,
  startTime: 0
})
// → レスポンスの clipId を記録
```

#### BGM 音量設定

```
mcp__eizo__set_volume({
  sessionId: "...",
  clipId: "<BGMクリップID>",
  volume: 0.3
})
```

#### ダッキング設定

発話中にBGM音量を自動で下げる:

```
mcp__eizo__set_audio_ducking({
  sessionId: "...",
  clipId: "<BGMクリップID>",
  enabled: true,
  threshold: -30,
  reduction: 12,
  attack: 0.1,
  release: 0.3
})
```

#### BGM 音量基準

| 区間 | BGM音量 | ダッキング |
|---|---|---|
| 通常（話していない） | 40-50% | なし |
| 発話中 | 15-25% | 有効 |
| **強調ポイント発話中** | **8-15%** | **強めのダッキング** |
| イントロ/アウトロ | 60-80% | なし |

### 6b: 効果音（SE）

カメラワーク（Phase 5）に連動して CDN テンプレート SE を配置する。

#### SE テンプレート一覧（CDN）

用途別に分類した推奨 SE:

**Whoosh / Swoosh（ズームイン時）:**

| SE | CDN URL |
|---|---|
| Airy Swoosh (Fast) | `https://eizo-assets.web.app/audio/sfx/MA_Originals_swoosh___airy_fast_and_short.mp3` |
| Airy Swoosh (Thin) | `https://eizo-assets.web.app/audio/sfx/MA_Originals_swoosh___airy_thin_fast_short.mp3` |
| Fire Whoosh | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_FireMotion_Fast_Tourch_Woosh_MP3.mp3` |
| Orchestral Whoosh | `https://eizo-assets.web.app/audio/sfx/MA_Originals_OrchestralRisers_airy_whoosh_MP3.mp3` |
| Golf Whoosh | `https://eizo-assets.web.app/audio/sfx/MA_Originals_GolfClubWhooshes_3_MP3.mp3` |

**Impact / Hit（インパクトズーム時）:**

| SE | CDN URL |
|---|---|
| Bass Drop 01 | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_Bass%20Drop%2001.mp3` |
| Bass Drop 04 | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_Bass_Drop_04_MP3.mp3` |
| Hit Cave | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_Hit_Cave_01_MP3.mp3` |
| Heavy Impact | `https://eizo-assets.web.app/audio/sfx/MA_CliftonScratch_HeavySwordHitsAndImpacts_1_MP3.mp3` |
| Water Impact | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_WaterWhooshImpact_2.mp3` |

**Transition（話題転換時）:**

| SE | CDN URL |
|---|---|
| Swift Transition | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_SwiftIntenseTransitions_001.mp3` |
| Rapid Wind | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_RapidWindTransition2_004_MP3.mp3` |
| Swoosh with Drop | `https://eizo-assets.web.app/audio/sfx/MA_Originals_SwooshesWithDrops_1.mp3` |
| Tape Reverse | `https://eizo-assets.web.app/audio/sfx/MA_Originals_TapeReverse_01.mp3` |

**Pop / Ding（テキスト表示、通知音）:**

| SE | CDN URL |
|---|---|
| UI Swipe 1 | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_UISwipesAndTypes_1_MP3.mp3` |
| UI Swipe 2 | `https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_UISwipesAndTypes_2_MP3.mp3` |
| Notification Beep | `https://eizo-assets.web.app/audio/sfx/MA_CresterStudio_NotificationBeeps_1_MP3.mp3` |
| Twinkle | `https://eizo-assets.web.app/audio/sfx/MA_SFXpecial_Twinkle_Glockenpiel_01_MP3.mp3` |

**Riser（結論・まとめ時）:**

| SE | CDN URL |
|---|---|
| Polar Wind Riser | `https://eizo-assets.web.app/audio/sfx/MA_Paula_Sounds_CinematicFuel_Whooshes_Risers_Polar_Wind_Riser_MP3.mp3` |
| Spaceship Riser | `https://eizo-assets.web.app/audio/sfx/MA_Paula_Sounds_CinematicFuel_Whooshes_Risers_Spaceship_Take_Off_MP3.mp3` |
| Futuristic Rise | `https://eizo-assets.web.app/audio/sfx/MA_Originals_FuutreCyberTech_loading_bleeps_and_stretch_transformation_bot_MP3.mp3` |

#### トリガー → SE マッピング

| トリガー | SE カテゴリ | タイミング | 推奨 SE |
|---|---|---|---|
| 強調 zoom-in | Whoosh | カメラ動き開始時 | Airy Swoosh (Fast) |
| impact-zoom | Impact | ズーム到達時 | Bass Drop 01 or Hit Cave |
| 話題転換 | Transition | 転換ポイント | Swift Transition |
| テキスト表示 | Pop | テキスト出現時 | Twinkle or UI Swipe 1 |
| 結論 | Riser | まとめ開始時 | Polar Wind Riser |

#### SE インポート & 配置の手順

```
// Step 1: 必要な SE を CDN からインポート（動画ごとに初回のみ）
// ※ 同じ種別の SE は 1 回インポートすれば複数箇所で再利用可能
mcp__eizo__import_media_from_url({
  sessionId: "...",
  url: "https://eizo-assets.web.app/audio/sfx/MA_Originals_swoosh___airy_fast_and_short.mp3",
  fileName: "se-whoosh.mp3"
})
// → mediaId を記録

mcp__eizo__import_media_from_url({
  sessionId: "...",
  url: "https://eizo-assets.web.app/audio/sfx/MA_CineTransitions_Bass%20Drop%2001.mp3",
  fileName: "se-impact.mp3"
})
// → mediaId を記録

// Step 2: カメラエフェクトのタイミングに合わせて配置
mcp__eizo__add_clip_from_media({
  sessionId: "...",
  mediaId: "<whoosh SE の mediaId>",
  trackIndex: -1,
  startTime: <カメラエフェクト開始時間>
})

// Step 3: 音量設定
mcp__eizo__set_volume({
  sessionId: "...",
  clipId: "<SEクリップID>",
  volume: 0.2
})
```

#### SE 運用ルール

- SE音量: **15-25%**（BGMより控えめ、邪魔にならない程度）
- 同じ SE を連続使用してもよい（whoosh は繰り返し使うのが自然）
- impact 系は **high レベルの強調ポイントのみ** に限定する
- 1動画あたり SE の種類は **3-5 種類以内** に抑える（多すぎると散漫になる）
- SE は **カメラエフェクトがある箇所のみ** に配置する（Phase 5 で none/low の箇所には不要）

### 6c: オーディオエフェクト

メイン音声トラックに適用:

#### ノイズリダクション

```
mcp__eizo__apply_noise_reduction({
  sessionId: "...",
  clipId: "<メイン動画クリップID>"
})
```

#### コンプレッション（音量均一化）

```
mcp__eizo__add_audio_effect({
  sessionId: "...",
  clipId: "<メイン動画クリップID>",
  type: "compressor",
  params: {
    ratio: 3,
    threshold: -20,
    attack: 0.01,
    release: 0.1
  }
})
```

#### EQ（声をクリアに）

```
mcp__eizo__add_audio_effect({
  sessionId: "...",
  clipId: "<メイン動画クリップID>",
  type: "eq",
  params: {
    highShelf: { frequency: 3000, gain: 2 },
    lowCut: { frequency: 80 }
  }
})
```

---

## Phase 7: ビジュアル仕上げ

### 7a: カラーグレーディング

メイン動画クリップに適用。素材種別とターゲットプラットフォームに応じて調整。

```
mcp__eizo__set_color_grading({
  sessionId: "...",
  clipId: "<メイン動画クリップID>",
  contrast: 8,
  saturation: 12,
  shadows: { r: -5, g: 0, b: 5 },
  highlights: { r: 5, g: 2, b: -3 },
  luminance: 5
})
```

#### プリセット基準

| パラメータ | YouTube向け | Cinematic |
|---|---|---|
| contrast | +5〜10 | +10〜15 |
| saturation | +10〜15 | +5〜10 |
| shadows | ブルー寄せ | ティール寄せ |
| highlights | ウォーム寄せ | オレンジ寄せ |
| luminance | +5 | 0 |

### 7b: ビネット

視線を中央に集中させる効果:

```
mcp__eizo__add_effect({
  sessionId: "...",
  clipId: "<メイン動画クリップID>",
  type: "vignette",
  params: {
    intensity: 0.3,
    radius: 0.8
  }
})
```

- intensity: 0.3-0.5（控えめ）
- radius: 0.7-0.8

### 7c: テキストオーバーレイ（任意）

Phase 4 で特定した強調ポイントや話題転換に基づき、テキストを配置する。

| テキスト種別 | タイミング | スタイル |
|---|---|---|
| タイトル | 0-3秒 | 大きめ、アニメーション付き |
| チャプタータイトル | 話題転換時 | 中サイズ、slide-up |
| ポイント表示 | 強調時 | キーワードをテキスト表示 |
| CTA / エンドカード | 最後 | 大きめ、fade |

```
// タイトル
mcp__eizo__add_text({
  sessionId: "...",
  text: "<動画タイトル>",
  startTime: 0,
  duration: 3,
  style: {
    fontSize: 48,
    fontColor: "#FFFFFF",
    position: { x: 0.5, y: 0.3 }
  }
})

mcp__eizo__apply_text_animation_preset({
  sessionId: "...",
  clipId: "<テキストクリップID>",
  preset: "fade-in"
})

// チャプタータイトル（話題転換ポイント）
mcp__eizo__add_text({
  sessionId: "...",
  text: "<チャプター名>",
  startTime: <話題転換の時間>,
  duration: 2,
  style: {
    fontSize: 32,
    fontColor: "#FFFFFF",
    position: { x: 0.5, y: 0.2 }
  }
})

mcp__eizo__apply_text_animation_preset({
  sessionId: "...",
  clipId: "<テキストクリップID>",
  preset: "slide-up"
})
```

### 7d: トランジション

Phase 4 の話題転換ポイントやカット間にトランジションを配置する。

| シーン間 | トランジション | duration |
|---|---|---|
| カット間（同トピック） | なし or dissolve 0.2s | 短め |
| 話題転換 | dissolve 0.5s | 標準 |
| セクション切り替え | wipe or slide 0.5s | 標準 |
| アウトロ | fade 1.0s | 長め |

```
// 話題転換時の dissolve
mcp__eizo__add_transition({
  sessionId: "...",
  clipId: "<該当クリップID>",
  type: "dissolve",
  duration: 0.5
})

// アウトロの fade
mcp__eizo__add_transition({
  sessionId: "...",
  clipId: "<最後のクリップID>",
  type: "fade",
  duration: 1.0
})
```

### 7e: フェード

動画の冒頭と末尾にフェードを適用:

```
// 冒頭 fade-in
mcp__eizo__set_fade({
  sessionId: "...",
  clipId: "<最初のクリップID>",
  fadeIn: 0.8
})

// 末尾 fade-out
mcp__eizo__set_fade({
  sessionId: "...",
  clipId: "<最後のクリップID>",
  fadeOut: 1.2
})
```

- 動画冒頭: fade-in **0.5-1.0s**
- 動画末尾: fade-out **1.0-1.5s**

---

## Phase 8: Gemini レビュー & 改善ループ

### エクスポート

```
mcp__eizo__export_video({
  sessionId: "...",
  format: "mp4",
  quality: "high"
})
```

### レビューリクエスト

eizo API サーバーの `/video-review` エンドポイントにリクエストを送信する。
**注:** Gemini レビューは現在 MCP ツール化されていないため、Bash ツールで REST API を直接呼び出す（本パイプラインで唯一の非 MCP 操作）。

```bash
curl -X POST http://localhost:3001/video-review \
  -H "Content-Type: application/json" \
  -d '{
    "videoPath": "<エクスポートファイルパス>",
    "reviewCategories": [
      {
        "category": "pacing",
        "criteria": "不自然な間やテンポの悪い箇所がないか。カットが急すぎる・遅すぎる箇所。"
      },
      {
        "category": "subtitle",
        "criteria": "字幕の正確性、タイミング、読みやすさ。文節の切り方は自然か。固有名詞は正しいか。"
      },
      {
        "category": "visual_emphasis",
        "criteria": "カメラワークの配置は適切か。強調の効果は十分か。過剰/不足はないか。同じエフェクトが連続していないか。"
      },
      {
        "category": "audio_mix",
        "criteria": "BGM音量バランスは適切か。ダッキングは効果的に機能しているか。SE配置は自然か。音声はクリアか。"
      },
      {
        "category": "color_grade",
        "criteria": "カラーグレーディングの統一感。映像の見やすさ。ビネットは自然か。全体の色味は意図通りか。"
      },
      {
        "category": "transition",
        "criteria": "シーン間の繋がりは自然か。トランジションの種類と長さは適切か。フェードイン/アウトは効果的か。"
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

```json
{
  "overallScore": 7.5,
  "categoryResults": {
    "pacing": { "score": 8, "issues": [...] },
    "subtitle": { "score": 6, "issues": [...] },
    "visual_emphasis": { "score": 7, "issues": [...] },
    "audio_mix": { "score": 7.5, "issues": [...] },
    "color_grade": { "score": 8, "issues": [...] },
    "transition": { "score": 7, "issues": [...] },
    "overall_quality": { "score": 7.5, "issues": [...] }
  },
  "done": false,
  "summary": "..."
}
```

### ループ終了条件

以下のいずれかを満たしたら終了:
1. `done === true`（overallScore >= 8.5 かつ全カテゴリ score >= 7）
2. ループ **3回** 完了（改善が収束しない場合の打ち切り）
3. 前回と今回の overallScore の差が **0.3 未満**（改善効果が頭打ち）

### フィードバック → アクション マッピング

Gemini のレビュー結果に含まれる issue を、対応する eizo MCP tool で修正する。

#### pacing（テンポ）

| issue | アクション |
|-------|----------|
| 不自然な間がある | `trim_clip` で該当タイムスタンプ付近のクリップを短縮 |
| カットが急すぎる | `silence_cut` を paddingBefore/paddingAfter 増やして再調整、または `undo` して結合 |

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
| 文節区切りが不自然 | `update_subtitle` で text を修正 |

```
mcp__eizo__update_subtitle({
  sessionId: "...",
  subtitleId: "<該当字幕ID>",
  text: "<修正後テキスト>",
  endTime: <延長後の終了時間>
})
```

#### visual_emphasis（カメラワーク）

| issue | アクション |
|-------|----------|
| カメラワークがない箇所 | `add_camera_effect` で該当箇所にエフェクト追加 |
| カメラワークが多すぎる | `remove_camera_effect` で一部削除 |
| エフェクトが単調 | `remove_camera_effect` → `add_camera_effect` で種類を変更 |

```
mcp__eizo__add_camera_effect({
  sessionId: "...",
  clipId: "...",
  type: "zoom-in",
  startTime: <開始時間>,
  duration: 2.0,
  params: { scale: 1.2, easing: "ease-out" }
})

mcp__eizo__remove_camera_effect({
  sessionId: "...",
  clipId: "...",
  effectId: "<削除するエフェクトID>"
})
```

#### audio_mix（オーディオミックス）

| issue | アクション |
|-------|----------|
| BGM音量が大きい | `set_volume` で音量を下げる |
| BGM音量が小さい | `set_volume` で音量を上げる |
| ダッキングが不足 | `set_audio_ducking` で reduction を増やす |
| ダッキングが過剰 | `set_audio_ducking` で reduction を減らす |
| SE音量が不適切 | `set_volume` でSE個別に調整 |

```
mcp__eizo__set_volume({
  sessionId: "...",
  clipId: "<BGMクリップID>",
  volume: 0.2
})

mcp__eizo__set_audio_ducking({
  sessionId: "...",
  clipId: "<BGMクリップID>",
  enabled: true,
  threshold: -30,
  reduction: 15,
  attack: 0.1,
  release: 0.3
})
```

#### color_grade（カラーグレーディング）

| issue | アクション |
|-------|----------|
| 色味が薄い | `set_color_grading` で彩度・コントラストを上げる |
| 色味が過剰 | `set_color_grading` で彩度・コントラストを下げる |
| 統一感がない | `set_color_grading` を全クリップに統一適用 |
| ビネットが強い | `update_effect` で intensity を下げる |

```
mcp__eizo__set_color_grading({
  sessionId: "...",
  clipId: "<メイン動画クリップID>",
  contrast: 12,
  saturation: 15,
  luminance: 5
})
```

#### transition（トランジション）

| issue | アクション |
|-------|----------|
| 繋がりが不自然 | `add_transition` でクロスフェード追加 |
| トランジションが長すぎる | `add_transition` で duration を短縮 |
| トランジションが不足 | 話題転換ポイントに `add_transition` を追加 |

```
mcp__eizo__add_transition({
  sessionId: "...",
  clipId: "<該当クリップ>",
  type: "dissolve",
  duration: 0.5
})
```

### 改善実行後

改善を適用したら、再度エクスポート → レビューを繰り返す。

---

## エラーハンドリング

### セッション接続失敗
- `eizo_list_sessions` が空の場合: 「eizo でプロジェクトを開いてからもう一度お試しください」とユーザーに伝える

### MCP tool 呼び出し失敗
- リトライ1回。それでも失敗したら該当モジュールをスキップし、次のモジュールに進む
- スキップしたモジュールはユーザーに報告する

### Phase 3 字幕補正失敗
- replace_all_subtitles が失敗した場合: 個別の `update_subtitle` にフォールバックする
- それも失敗した場合: 補正なしの自動生成字幕のまま続行し、ユーザーに報告

### Phase 5 カメラエフェクト失敗
- add_camera_effect が失敗した場合: `add_keyframe` を使ったキーフレームベースのズームにフォールバックする

### Phase 6 オーディオ失敗
- BGM素材が見つからない場合: BGM追加をスキップし、ノイズリダクションとコンプレッションのみ適用
- ダッキング設定が失敗した場合: set_volume で手動音量調整にフォールバック

### Phase 7 ビジュアル仕上げ失敗
- 個別のサブフェーズ（7a-7e）が失敗しても他のサブフェーズには影響しない
- 失敗したサブフェーズはスキップし、ユーザーに報告

### Gemini レビュー失敗
- API エラーの場合: リトライ1回
- タイムアウトの場合: レビューなしで完了とし、ユーザーに手動確認を依頼
- GEMINI_API_KEY 未設定: レビューをスキップし、Phase 7 までの編集結果で完了

### エクスポート失敗
- エラー内容をユーザーに表示し、手動エクスポートを案内

---

## ユーザーへの報告

各フェーズ完了時に進捗を報告する:

```
Phase 1 完了: 「素材を分析しました。トーキングヘッド動画（3分24秒）、1920x1080 30fps、音声あり。」
Phase 2 完了: 「無音カットを実行しました。12箇所カット、総尺 3:24 → 2:58。」
Phase 3 完了: 「字幕を生成しAI補正を適用しました。45件生成、8件の固有名詞修正、3件のフィラー除去。」
Phase 4 完了: 「内容を分析しました。強調ポイント: high 4箇所、medium 8箇所、low 12箇所。話題転換 3箇所。」
Phase 5 完了: 「カメラワークを配置しました。zoom-in 6箇所、impact-zoom 2箇所、pan 4箇所、slow-zoom-in 8箇所。」
Phase 6 完了: 「オーディオ演出を適用しました。BGMダッキング設定、SE 5箇所、ノイズリダクション適用済み。」
Phase 7 完了: 「ビジュアル仕上げを適用しました。カラーグレーディング、ビネット、トランジション 3箇所、フェードイン/アウト。」
Phase 8 完了: 「Geminiレビュー結果: 総合スコア 8.7/10。全カテゴリが基準を満たしました。」
```

改善ループ中:
```
改善ループ 1: 「スコア 7.2 → subtitle(6.5)、audio_mix(6.8) に改善の余地あり。修正を実行します。」
改善ループ 2: 「スコア 8.1 → color_grade(6.8) を調整。再レビューします。」
最終結果: 「最終スコア: 8.7/10（2回の改善ループ）。全カテゴリが基準（7以上）を満たしました。」
```

---

## プラットフォーム別プリセット

ユーザーがターゲットプラットフォームを指定した場合、以下の設定を適用する:

| 設定 | YouTube | Shorts/TikTok | Instagram Reels |
|---|---|---|---|
| 解像度 | 1920x1080 | 1080x1920 | 1080x1920 |
| FPS | 30 | 30 | 30 |
| 字幕位置 | 下部中央 | 中央やや下 | 中央やや下 |
| 字幕サイズ | 24-36px | 32-48px | 32-48px |
| カメラワーク | 控えめ | 積極的 | 積極的 |
| テンポ | 標準 | 速め | 速め |
| BGM | 控えめ | やや大きめ | やや大きめ |
| 最大長 | 制限なし | 60秒 | 90秒 |

---

## 使用ツール一覧

### MCP ツール（全操作の 99%）

| Phase | ツール | 用途 |
|---|---|---|
| 1 | eizo_list_sessions, get_project_info, get_timeline_state | 素材分析 |
| 2 | silence_cut | 無音カット |
| 3 | auto_subtitle, export_srt, replace_all_subtitles, apply_subtitle_style_preset, set_subtitle_style | 字幕生成・AI補正 |
| 4 | (Claude内部処理) | 内容分析・強調タグ付け |
| 5 | add_camera_effect, remove_camera_effect | カメラワーク |
| 6 | import_media_from_url, add_clip_from_media, set_volume, set_audio_ducking, apply_noise_reduction, add_audio_effect | オーディオ演出（CDN テンプレート素材使用） |
| 7 | set_color_grading, add_effect, add_text, update_text_style, apply_text_animation_preset, add_transition, set_fade | ビジュアル仕上げ |
| 8 | export_video, trim_clip, update_subtitle, add_camera_effect, remove_camera_effect, set_volume, set_audio_ducking, set_color_grading, add_transition | レビュー後の改善 |

### Bash ツール（Phase 8 のみ）

| Phase | コマンド | 用途 |
|---|---|---|
| 8 | `curl -X POST http://localhost:3001/video-review` | Gemini 3.1 Pro 映像レビュー（未 MCP 化） |
