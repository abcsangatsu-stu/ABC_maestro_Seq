# ABC Maestro SEQ — プロジェクト引き継ぎドキュメント

**最終更新:** 2026-03-08
**ステータス:** v6.3（btn-row 3段化 + RTステップ別周波数）
**最新ファイル:** `ABC_maestro_Seq_v6_3.html`（単一HTML、約6720行）
**用途:** Oxi One MK2 用 LFOトリガーパターン作成ツール
**関連ツール:** Shape→Sound v2.1（パターン生成 → Maestroへインポート）

---

## 1. プロジェクト概要

バック3ch + RT 1ch の計4チャンネル・トリガーシーケンサー + Maestro LFOシミュレーター。
全チャンネルは同一BPMで並走し、ミックス済みのMIDI / WAV / MP3出力が可能。
デフォルトテーマ: sakura。チャンネル色はテーマ非依存で固定。

Shape→Sound で生成したパターンを S2S IMPORT 機能でインポート可能（v6.1〜）。

---

## 2. タイミング体系

| 定数 | 値 | 意味 |
|------|-----|------|
| `getQuant()` | 96 | 1小節 = 96 tick |
| `getMax()` | 128 | 最大ステップ数 |
| Normal lane | 1セル = 3 tick | 32セル/小節 |
| Triplet lane | 1セル = 2 tick | 48セル/小節 |

**音価:** NORMAL_DURS = [32,16,8,4,2,1], TRIPLET_DURS = [48,24,12,6,3]
`durToSteps(dur) = 96 / dur`

---

## 3. データ構造

### 波形タイプ（WAVEFORMS — 7種）

```javascript
var WAVEFORMS = [
  { id: 'ramp_up',      symbol: '↗', name: 'Ramp Up' },
  { id: 'ramp_down',    symbol: '↘', name: 'Ramp Down' },
  { id: 'triangle',     symbol: '△', name: 'Triangle' },
  { id: 'inv_triangle', symbol: '▽', name: 'Inv Tri' },
  { id: 'square',       symbol: '□', name: 'Square' },
  { id: 'high',         symbol: '—', name: 'High' },
  { id: 'low',          symbol: '_', name: 'Low' },
];
```

**休符の表現:** `on: true` + `wave: 'low'`。`on: false` は「セルが空」であり休符ではない。

### バック（3ch）
```javascript
patterns = [{ name, channels:[ch0,ch1,ch2], baseName, varySuffix, patternLength }]
state = patterns[currentPatternIdx].channels
song = [{ patIdx, repeat, sectionColor? }]
globalAudio = [{ freq, volume, pan } × 3]
```

### RT（1ch）
```javascript
rtPatterns = [{ name, channels:[rtCh0], baseName, varySuffix, patternLength,
                coverBackFrom:-1, coverBackTo:-1 }]
rtState = rtPatterns[currentRTPatternIdx].channels
rtSong = [{ patIdx, repeat, sectionColor? }]
rtGlobalAudio = { freq:440, volume:0.50, pan:0 }
```

### 各chステップ構造
```javascript
// バック3ch（freq は ch/global レベルのみ、step.freq は未使用）
{ on: boolean, wave: string, dur: number }

// RT ch（v6.3〜 step.freq 追加）
{ on: boolean, wave: string, dur: number,
  freq: number|null }   // null = ch/global値を使用
```

**チャンネルオブジェクト共通フィールド:**
```javascript
{ steps: [], count: number, note: number, mich: number,
  brushWave: string, brushDur: number,
  smooth: boolean, bipolar: boolean,
  selectedStep: number,   // -1 = 未選択
  volume: number, freq: number, pan: number }
```

### オーディオモード・ミュート・周波数クオンタイズ
```javascript
globalAudioMode = [true, true, true, true]    // per-channel G/L
chMute = [false, false, false, false]         // per-channel mute
freqQuantize = [false, false, false, false]   // per-channel ♪ ON/OFF
```

### 選択状態
```javascript
selectedSongCell = 0       // バック Song cell (-1=未選択)
selectedRTSongCell = 0     // RT Song cell (-1=未選択)
songPlayhead = 0           // 再生開始位置（back/RT共通）
lastSongFocus = 'back'     // 'back' or 'rt'
```

---

## 4. チャンネル色（固定）

CH1: `#e11d48`(赤), CH2: `#3b82f6`(青), CH3: `#22c55e`(緑), RT: `#06b6d4`(水色)

---

## 5. UI構成

```
┌─ SONG ─────────────────────────────────────────────────────────┐
│  [A x1] [— x2] [B x1] [+]    ← —=ブランク(グレー)            │
│  RT: [RA x1] [— x1] [+]                                       │
│  POOL: A(bold) B(dim) │ RT │ RA(bold)                          │
└────────────────────────────────────────────────────────────────┘
┌─ BTN ROW（v6.3〜 3段構成）────────────────────────────────────┐
│ 上段(大): ▶(青) ▶(赤) ■  ↻  ↓  ↑   ← アイコンのみ          │
│ 中段: PAT CLR | SONG CLR | DENSITY [slider] | RANDOM | VARY   │
│ 下段: SONG WAV | SONG MP3 | WAV | MP3 | EXPORT MIDI            │
│       SCORE | 16 SCORE | SETTINGS                              │
│ ↑S2S IMPORT は各 CH Block 内に配置                             │
└────────────────────────────────────────────────────────────────┘
┌─ CH BLOCKS ────────────────────────────────────────────────────┐
│  CH1 [M][G] Hz♪▼▲(note) Vol Pan [chain] FILL GAP|CLR|↑S2S   │
│  CH2/CH3/RT 同様                                               │
└────────────────────────────────────────────────────────────────┘
```

### 上段ボタン詳細（v6.3）
| ボタンID | アイコン | 色 | title |
|---|---|---|---|
| `#btnPlay` | ▶ / ⏸ | 青 `#3b82f6` | PAT PLAY |
| `#btnSongPlay` | ▶ / ⏸ | 赤 `#e11d48` | SONG PLAY |
| `#btnStop` | ■ | テーマ | STOP |
| `#btnLoop` | ↻ | テーマ | LOOP: ON/OFF |
| `#btnSave` | ↓ | テーマ | SAVE |
| `#btnLoad` | ↑ | テーマ | LOAD |

- 再生中は ▶ → ⏸（innerHTML 更新、span カラー要素なし）
- LOOP の ON/OFF 状態は `title` 属性と `.active` クラスで管理

### Song cell Cmd+C/V/D（v6.2〜）

| 操作 | ステップ選択あり | Song cell選択あり | どちらもなし |
|------|----------------|------------------|-------------|
| Cmd+C | セルコピー | Song cellコピー | パターン全体コピー |
| Cmd+V | セルペースト | Song cell上書き | Song末尾に追加 |
| Cmd+D | セル複製 | Song cell複製 | — |

### ブランクセル
- +ボタン → `name:''` の空パターン → Song上で `—` + グレー表示
- Save/Loadで名前保持（`refreshPatternNames` で空文字を保護）

---

## 6. 周波数関連（v6.2〜 / v6.3で拡張）

### ♪▼▲ ボタン動作

| ボタン | RTステップ未選択時 | RTステップ選択時（v6.3〜） |
|--------|------------------|--------------------------|
| ♪ | ch/global freqをクオンタイズON/OFF | 同左（ステップには影響しない） |
| ▼ | ch/global freqを半音下げ | **選択ステップの freq を半音下げ** |
| ▲ | ch/global freqを半音上げ | **選択ステップの freq を半音上げ** |

- per-channel独立。`freqQuantize = [false,false,false,false]`
- **音名は常時表示**（v6.3〜 freqQuantize ON/OFF に関係なく表示）
- バックCH: `fqnote0`〜`fqnote2`、RT: `fqnoter`

### RTステップ別 freq（v6.3〜）

```javascript
// step.freq が null → ch/global の freq を使用（後方互換）
// step.freq が数値 → その値で発音（優先）

playRTNote(duration, waveType, optChannels, stepFreq)
// stepFreq = step.freq != null ? step.freq : null
```

**表示:** RTの event chain セル内にノート名をミニ表示（`step.freq != null` の場合のみ）

**Save/Load:** `step.freq != null` の場合のみ保存（省略でファイルサイズ削減）。
ロード時は `ss.freq != null ? ss.freq : null` で復元。
`freqQuantize` は Save/Load に含まれない（再起動時 OFF）。

**操作一覧:**
- `setRTFreq(val)` / `setRTFreqInput(val)` — ステップ選択中はそのステップの freq を変更
- `stepRTFreq(dir)` — ステップ選択中はそのステップの freq を半音シフト
- `selectRTStep(s)` — 選択時に freq input を step.freq（null なら ch/global）で同期
- Cmd+C/D — RT ステップコピー・複製時に freq も含める
- `repackRTEvents()` — freq を保持したままリパック
- `fillRTGaps()` — Low 埋め時は `freq: null`
- `applyS2SToChannel()` — インポート時は `freq: null`

---

## 7. S2S IMPORT（v6.1〜）

各CH Block に `↑ S2S` ボタン。フォーマット: `S2S|BPM:120|/4 triangle, ...`
REPLACE / APPEND モード。BPM不一致時confirm。
インポートされたステップの `freq` は `null`（ch/global値を使用）。

---

## 8. 出力

- MIDI: バック ch1-3, RT ch4
- WAV/MP3: パターン単体 or **SONG WAV/SONG MP3**（v6.2〜専用ボタン）
- Score: テキスト形式（RT部分にノート名付記 v6.3〜） / 16ステップ表記
- `renderPatternOffline()` / `renderSongOffline()` — RT部分で `step.freq` 優先使用

---

## 9. Save/Load

現行バージョン: **6**（v6.3でバージョン番号変更なし）。後方互換あり。

**v6.2 falsy安全性修正:** `name`, `baseName`, `coverBackFrom/To`, `volume` の 0/空文字値を `!== undefined` 判定で保持。

**v6.3 RT step.freq:** null は省略して保存。ロード時に未定義なら null 復元。

---

## 10. バージョン履歴

### v6.0（2026-03-01）
Phase 3+完了。全基本機能実装。

### v6.1（2026-03-01）
S2S IMPORT追加。

### v6.2（2026-03-08）
- Song cell Cmd+D/C/V
- ブランクセル（+ボタン→名前なし→グレー表示）
- SONG WAV / SONG MP3 専用ボタン
- 周波数クオンタイズ（per-channel ♪▼▲ + ノート名表示 + 小数1桁）
- バグ修正: refreshPatternNames空文字保護、coverBackFrom 0値保持、volume 0保持、p.name欠落時の例外防止、parseInt→parseFloat

### v6.3（2026-03-08）
- **btn-row 3段化**
  - 上段（`.btn-row-top`）: ▶(青)/▶(赤)/■/↻/↓/↑ アイコンのみ、`font-size:1.4〜1.6em`
  - 中段（`.btn-row-mid`）: PAT CLR / SONG CLR / DENSITY / RANDOM / VARY
  - 下段（`.btn-row-btm`）: SONG WAV / SONG MP3 / WAV / MP3 / EXPORT MIDI / SCORE / 16 SCORE / SETTINGS
  - 既存 button id は全て変更なし
- **音名ラベル常時表示**: freqQuantize ON/OFF に関係なくノート名を常に表示（CH1〜3 + RT）
- **RTステップ別 freq（per-step frequency）**
  - `makeStep()` に `freq: null` 追加
  - RTステップ選択中に ▼▲ で選択ステップの freq を変更
  - event chain セル内にノート名ミニ表示
  - 再生・オフラインレンダリングで step.freq を優先使用
  - Save/Load 対応（null は省略）
  - Cmd+C/D でのコピー・複製時に freq を保持
  - Score 出力にノート名付記
  - バック3ch は一切変更なし

---

## 11. 既知の注意点

1. `state`/`rtState` はエイリアス。パターン切替時に再バインド
2. バック3ch既存コード（c<3ループ30箇所以上）は変更なし（v6.3でも維持）
3. 休符: `on:true` + `wave:'low'`（`on:false` は空セル）
4. ブランクパターン: `name===''`, `baseName===''`。refreshPatternNames で undefined/null のみ書き換え
5. freqQuantize はSave/Loadに含まれない（再起動時OFF）
6. Cmd+V Song cell上書きは `lastSongFocus` + `selectedSongCell >= 0` の両方が必要
7. Song外クリックで selectedSongCell が -1 にリセットされる
8. RT step.freq は `null` がデフォルト。バックch の step には freq フィールドなし（互換性維持のため追加禁止）
9. `updateRTFreqNoteLabel(freq)` は常に `freqToNoteName(freq)` を表示（v6.3〜、ON/OFF判定なし）

---

## 12. ロードマップ

```
v6.0〜v6.3     ✅ 完了
v7.0候補       📋 Song cell span（可変長セグメント）
Phase 4        📋 RT Vary + パターングループ
Phase 5        📋 Undo / クリックノイズ除去 / Mode 2 / 3連混在
```

---

## 13. v7.0候補: Song cell span（可変長セグメント）設計メモ

> **ステータス:** 未実装。設計検討済み。

### 要件
- Song欄の1セルが複数パターン分の長さにまたがれるようにする（`span` プロパティ）
- `span=2` → セル表示幅2倍 + stepGrid列数2倍（実際にステップ数が増える）
- BackとRTのセル幅を見かけ上揃えられる（異なるspanでも手動で幅を合わせる）

### データモデル案

```javascript
// song / rtSong エントリに span を追加
song = [{ patIdx, repeat, span: 1, sectionColor? }]
// span: 1〜8 の整数。既定 1。
// version を 7 に上げる。旧データは span=1 で移行。
```

### 実装方針（推奨A: パターン自動複製方式）

`span` 変更時に**パターンを複製**して `patternLength *= span` した新パターンを作成し、
そのSong cellの `patIdx` を新パターンに差し替える。

```
span=2 に変更:
  現パターン A (patternLength=128) → 複製 A_x2 (patternLength=256)
  song[i].patIdx → A_x2 のインデックスに差し替え
  song[i].span = 2
```

**利点:** 同じ patIdx を参照する他のSong cellに影響しない。再生・レンダリングがシンプル。
**欠点:** POOLにパターンが増える。

### 実装ステップ（差分設計書より）

1. データモデル追加: `span` フィールド + `version=7` + save/load
2. ヘルパ関数: `getEntrySpan()`, `expandPatternForSpan()`, `getLaneUnits()` 等
3. Song/RT UI可変幅化: `cell.style.width = (78 * span + gap*(span-1)) + 'px'`
4. Back/RT見かけ幅同期: 「幅同期ボタン」で手動実行（自動上書きは危険）
5. 再生ロジック: `songPos` をindex基準からunit基準（累積ユニット位置）へ変更
6. Grid長反映: 選択セルの span に応じて editableなpatternLengthを決定
7. D&D整合: 移動時に span を保持、ドロップ位置をunit境界で決定
8. エクスポート追従: renderSongOffline / generateScore / MIDI

### 難所
- `songPos` が「配列index前提」→「累積ユニット位置」への変更は再生ロジック全体に影響
- D&D + unit境界計算
- BackとRTが異なるspan構成のとき再生同期をどう保つか

---

## 14. 外部依存

lamejs (MP3エンコード) のみ。Vanilla JS、単一HTML。

---

## 15. テーマ一覧

dark, teal, ocean, light, **sakura**(デフォルト), mint, sky, cream, lavender, peach
