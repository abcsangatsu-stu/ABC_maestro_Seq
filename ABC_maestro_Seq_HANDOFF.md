# ABC Maestro SEQ — プロジェクト引き継ぎドキュメント

**最終更新:** 2026-03-08
**ステータス:** v6.2（4機能追加 + バグ修正）
**最新ファイル:** `ABC_maestro_Seq_v6_2.html`（単一HTML、約6661行、163関数）
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
{ on: boolean, wave: string, dur: number,
  freq: number, volume: number, pan: number,
  smooth: boolean, bipolar: boolean,
  brushWave: string, brushDur: number, selectedStep: number }
```

### オーディオモード・ミュート・周波数クオンタイズ
```javascript
globalAudioMode = [true, true, true, true]    // per-channel G/L
chMute = [false, false, false, false]         // per-channel mute
freqQuantize = [false, false, false, false]   // per-channel ♪ ON/OFF (v6.2)
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
┌─ BTN ROW ─────────────────────────────────────────────────────┐
│ PAT PLAY│SONG PLAY│LOOP│STOP│PAT CLR│SONG CLR│EXPORT MIDI    │
│ SAVE│LOAD│DENSITY│RANDOM│SETTINGS│VARY│SCORE│16 SCORE         │
│ WAV│MP3│SONG WAV│SONG MP3│↑S2S IMPORT(各chに配置)             │
└────────────────────────────────────────────────────────────────┘
┌─ CH BLOCKS ────────────────────────────────────────────────────┐
│  CH1 [M][G] Hz♪▼▲(note) Vol Pan [chain] FILL GAP|CLR|↑S2S   │
│  CH2/CH3/RT 同様                                               │
└────────────────────────────────────────────────────────────────┘
```

### Song cell Cmd+C/V/D（v6.2）

| 操作 | ステップ選択あり | Song cell選択あり | どちらもなし |
|------|----------------|------------------|-------------|
| Cmd+C | セルコピー | Song cellコピー | パターン全体コピー |
| Cmd+V | セルペースト | Song cell上書き | Song末尾に追加 |
| Cmd+D | セル複製 | Song cell複製 | — |

### ブランクセル
- +ボタン → `name:''` の空パターン → Song上で `—` + グレー表示
- Save/Loadで名前保持（`refreshPatternNames` で空文字を保護）

---

## 6. 周波数クオンタイズ（v6.2）

| ボタン | 動作 |
|--------|------|
| ♪ | ON/OFF トグル。ONで12平均律にスナップ、小数1桁表示 |
| ▼ | 半音下げ |
| ▲ | 半音上げ |

per-channel独立。`freqQuantize = [false,false,false,false]`。

---

## 7. S2S IMPORT（v6.1〜）

各CH Block に `↑ S2S` ボタン。フォーマット: `S2S|BPM:120|/4 triangle, ...`
REPLACE / APPEND モード。BPM不一致時confirm。

---

## 8. 出力

- MIDI: バック ch1-3, RT ch4
- WAV/MP3: パターン単体 or **SONG WAV/SONG MP3**（v6.2専用ボタン）
- Score: テキスト形式 / 16ステップ表記

---

## 9. Save/Load

バージョン6。後方互換あり。
**v6.2 falsy安全性修正:** `name`, `baseName`, `coverBackFrom/To`, `volume` の 0/空文字値を `!== undefined` 判定で保持。

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

---

## 11. 既知の注意点

1. `state`/`rtState` はエイリアス。パターン切替時に再バインド
2. バック3ch既存コード（c<3ループ30箇所以上）は変更なし
3. 休符: `on:true` + `wave:'low'`（`on:false` は空セル）
4. ブランクパターン: `name===''`, `baseName===''`。refreshPatternNames で undefined/null のみ書き換え
5. freqQuantize はSave/Loadに含まれない（再起動時OFF）
6. Cmd+V Song cell上書きは `lastSongFocus` + `selectedSongCell >= 0` の両方が必要
7. Song外クリックで selectedSongCell が -1 にリセットされる

---

## 12. ロードマップ

```
v6.0〜v6.2     ✅ 完了
Phase 4        📋 RT Vary + パターングループ
Phase 5        📋 Undo / クリックノイズ除去 / Mode 2 / 3連混在
```

---

## 13. 外部依存

lamejs (MP3エンコード) のみ。Vanilla JS、単一HTML。

---

## 14. テーマ一覧

dark, teal, ocean, light, **sakura**(デフォルト), mint, sky, cream, lavender, peach
