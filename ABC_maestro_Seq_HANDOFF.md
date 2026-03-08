# ABC Maestro SEQ — プロジェクト引き継ぎドキュメント

**最終更新:** 2026-03-08
**ステータス:** v7.0（Song cell span + rtCurStep独立進行 + Pool フィルタ）
**最新ファイル:** `ABC_maestro_Seq_v7_0.html`（単一HTML、約7230行）
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
| `getMax()` | 1024 | 最大ステップ数（v7.0〜: 128→1024, span対応） |
| `getEditMax()` | 動的 | 現在のBackパターンの patternLength |
| `getRTEditMax()` | 動的 | 現在のRTパターンの patternLength |
| Normal lane | 1セル = 3 tick | `getEditMax()/3` セル/パターン |
| Triplet lane | 1セル = 2 tick | `getEditMax()/2` セル/パターン |

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
song = [{ patIdx, repeat, span, sectionColor? }]  // v7.0: span追加
globalAudio = [{ freq, volume, pan } × 3]
```

### RT（1ch）
```javascript
rtPatterns = [{ name, channels:[rtCh0], baseName, varySuffix, patternLength,
                coverBackFrom:-1, coverBackTo:-1 }]
rtState = rtPatterns[currentRTPatternIdx].channels
rtSong = [{ patIdx, repeat, span, sectionColor? }]  // v7.0: span追加
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

### Song entry 構造（v7.0〜）
```javascript
{ patIdx: number, repeat: number, span: number, sectionColor?: string }
// span: 1〜8。セル幅とパターン長の倍率。
// repeat: span適用後のパターンを何回繰り返すか。
// 例: span=2, repeat=3 → パターン長2倍のパターンを3回再生
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
poolMultiSelect = []       // v7.0: Pool複数選択 [{type,idx}]
```

---

## 4. チャンネル色（固定）

CH1: `#e11d48`(赤), CH2: `#3b82f6`(青), CH3: `#22c55e`(緑), RT: `#06b6d4`(水色)

---

## 5. UI構成

```
┌─ SONG ─────────────────────────────────────────────────────────┐
│  [A x1] [— x2] [B_x2 x1 ═] [+]  ← ═はspan拡張セル(幅2倍)  │
│  RT: [RA x1] [— x1] [+]                                       │
│  POOL: A(bold) B(dim) │ RT │ RA(bold)                          │
│  ※ ブランク/コピー派生/同名重複はPOOLに非表示                  │
└────────────────────────────────────────────────────────────────┘
┌─ BTN ROW（v6.3〜 3段構成）────────────────────────────────────┐
│ 上段(大): ▶(青) ▶(赤) ■  ↻  ↓  ↑   ← アイコンのみ          │
│ 中段: PAT CLR | SONG CLR | DENSITY [slider] | RANDOM | VARY   │
│ 下段: SONG WAV | SONG MP3 | WAV | MP3 | EXPORT MIDI            │
│       SCORE | 16 SCORE | SETTINGS                              │
└────────────────────────────────────────────────────────────────┘
┌─ CH BLOCKS ────────────────────────────────────────────────────┐
│  CH1 [M][G] Hz♪▼▲(note) Vol Pan [chain] FILL GAP|CLR|↑S2S   │
│  CH2/CH3/RT 同様                                               │
└────────────────────────────────────────────────────────────────┘
```

### Song cell span（v7.0〜）

- セル右端にドラッグハンドル `┃` が表示（hover時のみ）
- 右ドラッグで整数倍（1〜8）にリサイズ
- **Back**: span変更時にステップを**複製充填**（A→AA for span=2）
- **RT**: span変更時に空ステップで拡張（新メロディ用）
- パターンは自動複製（同じpatIdxの他セルに影響しない）
- span適用後のパターンは自由に編集可能

### Pool 表示ルール（v7.0〜）

- ブランク（`name === ''`）→ 非表示
- 同名パターン → 最初の1つだけ表示
- コピー/Vary/Span由来（`varySuffix !== ''`）→ 非表示
- リネーム済み（`varySuffix` がクリアされる）→ 再表示
- **Shift+クリック** で複数選択 → Delete で一括削除

### Step grid 操作（v7.0改善）

- **Shift+クリック**: グリッド上で範囲選択（Back/RT両方）
- **グループドラッグ**: 範囲選択状態でドラッグ → 位置関係保持のまま移動
- **グループコピー**: Cmd+C で範囲選択をグループコピー（相対位置を保持）
- **グループペースト**: Cmd+V で選択位置を基準に相対位置を保ったままペースト
- **Cmd+V 単体**: 選択ステップ位置にペースト（未選択時は次の空き）
- **Delete**: 範囲選択状態で一括削除
- **RT新規ステップ**: 直前のステップの freq を自動継承
- **repackEvents**: 位置保持型（ギャップを詰めずに重複分だけ後方シフト）

### Song cell Cmd+C/V/D

| 操作 | ステップ範囲選択 | ステップ単体選択 | Song cell選択 | どちらもなし |
|------|----------------|----------------|--------------|-------------|
| Cmd+C | **グループコピー** | セルコピー | Song cellコピー | パターン全体コピー |
| Cmd+V | **グループペースト** | 選択位置にペースト | Song cell上書き | Song末尾に追加 |
| Cmd+D | — | セル複製 | Song cell複製(span保持) | — |

**clipboard排他制御（v7.0〜）**: cellClipboard設定時にpatternClipboard=null、逆も同様。

---

## 6. 周波数関連（v6.2〜 / v6.3で拡張 / v7.0で改善）

### ♪▼▲ ボタン動作

| ボタン | RTステップ未選択時 | RTステップ選択時（v6.3〜） |
|--------|------------------|--------------------------|
| ♪ | ch/global freqをクオンタイズON/OFF | 同左 |
| ▼ | ch/global freqを半音下げ | 選択ステップの freq を半音下げ |
| ▲ | ch/global freqを半音上げ | 選択ステップの freq を半音上げ |

### RTステップ別 freq

- `step.freq` が `null` → ch/global の freq を使用
- `step.freq` が数値 → その値で発音（優先）
- **v7.0**: 新規ステップ作成時に直前ステップの freq を自動継承
- **v7.0**: グリッドドラッグ移動時に freq を保持

---

## 7. S2S IMPORT（v6.1〜）

各CH Block に `↑ S2S` ボタン。フォーマット: `S2S|BPM:120|/4 triangle, ...`
REPLACE / APPEND モード。BPM不一致時confirm。
インポートされたステップの `freq` は `null`。

---

## 8. 出力

- MIDI: バック ch1-3, RT ch4
- WAV/MP3: パターン単体 or SONG WAV/SONG MP3
- Score: テキスト形式（RT部分にノート名付記）/ 16ステップ表記
- `renderPatternOffline()` / `renderSongOffline()` — RT部分で `step.freq` 優先使用

---

## 9. Save/Load

現行バージョン: **7**。後方互換あり。

- v6以前のデータ: `span` がなければ `1` を補完
- `song` / `rtSong` エントリの `span` は 1〜8 にクランプ
- `freqQuantize` はSave/Loadに含まれない（再起動時OFF）

---

## 10. Span ヘルパ関数（v7.0〜）

```javascript
getEntrySpan(ent)              // span取得（1〜8にclamp）
makeSongEntry(patIdx, repeat, span)  // span付きエントリ生成
expandPatternSteps(pat, newLen, isRT) // ステップ配列を拡張
cloneAndExpandPattern(pat, newSpan, isRT) // deep clone + 拡張
applySpanChange(entry, songIdx, newSpan, isRT)
  // Back: ステップ複製充填（A→AA）
  // RT: 空ステップ拡張
startSpanDrag(entry, songIdx, isRT, startEvt)
  // 右端ドラッグハンドルのmousemove/mouseup管理
getEditMax() / getRTEditMax()  // 現在パターンの実効max長
getLaneNCells() / getRTLaneNCells() // パターン長ベースのグリッドセル数
```

---

## 11. repackEvents 動作（v7.0〜位置保持型）

```
v6.x: 全イベントを位置0から隙間なく再配置（ギャップ消失）
v7.0: 各イベントの元位置を保持し、重複分のみ後方シフト
      → fill gap なしでもステップ位置が維持される
```

---

## 12. 再生ロジック（v7.0〜 独立カウンター方式）

### ステップカウンター

```javascript
curStep    // Back用。patLen到達で0リセット
rtCurStep  // RT用（v7.0〜）。rtPatLen到達で0リセット
// 両方ともsongTick()内で毎tick +1される
```

### 進行判定

```
Back: curStep >= patLen → curStep=0, songRepeatCount++
RT:   rtCurStep >= rtPatLen → rtCurStep=0, rtSongRepeatCount++
```

BackとRTは**独立して進行**する。例：
- Back 96/96/96 + RT 192/96
- Back側が96消化×2回する間に、RT側は192を1回消化
- Song欄のハイライトもそれぞれ独立して進む

### オフラインレンダリング

`renderSongOffline()` は元々 `rtOffset` でRT長を独立計算しているため変更不要。

---

## 13. バージョン履歴

### v6.0（2026-03-01）
Phase 3+完了。全基本機能実装。

### v6.1（2026-03-01）
S2S IMPORT追加。

### v6.2（2026-03-08）
Song cell Cmd+D/C/V、ブランクセル、SONG WAV/MP3、周波数クオンタイズ、バグ修正多数。

### v6.3（2026-03-08）
btn-row 3段化、音名ラベル常時表示、RTステップ別freq。

### v7.0（2026-03-08）
- **Song cell span（可変長セグメント）**
  - 右端ドラッグでセル幅を整数倍にリサイズ（1〜8）
  - Back: ステップ複製充填（A→AA）、RT: 空ステップ拡張
  - パターン自動複製方式（再生ロジック変更不要）
  - `version: 7`、`getMax()` 1024
- **repackEvents 位置保持型に変更**
  - fill gap なしでもステップ位置が維持される
- **Pool フィルタリング**
  - ブランク非表示、同名重複非表示、コピー派生（varySuffix）非表示
  - Shift+クリック複数選択 + Delete一括削除
- **Step grid 改善**
  - Shift+クリックでグリッド上範囲選択（Back/RT）
  - グループドラッグ移動（位置関係保持）
  - Cmd+V: 選択位置にペースト
  - RTドラッグ移動時にfreq保持
  - RT新規ステップでfreq自動継承
- **Clipboard排他制御**: cellClipboard/patternClipboard相互クリア
- **タブ名**: v5 → v7.0 に更新
- **セル高さ**: 68px → 56px
- **[P1修正] RT独立ステップカウンター `rtCurStep`**: BackとRTで異なるpatternLengthの場合にRT進行がズレる問題を修正。`rtCurStep` を新設し、RT進行を `rtCurStep >= rtPatLen` で独立判定
- **[P2修正] span=1縮小時**: confirmでパターン長も元に戻すか選択可能に
- **グループコピペ**: Shift+クリック範囲選択 → Cmd+C/V で位置関係保持のままコピペ

---

## 14. 既知の注意点

1. `state`/`rtState` はエイリアス。パターン切替時に再バインド
2. バック3ch既存コード（c<3ループ30箇所以上）は変更なし
3. 休符: `on:true` + `wave:'low'`（`on:false` は空セル）
4. ブランクパターン: `name===''`, `baseName===''`。Song欄でダブルクリックリネーム可能
5. freqQuantize はSave/Loadに含まれない
6. Cmd+V Song cell上書きは `lastSongFocus` + `selectedSongCell >= 0` の両方が必要
7. RT step.freq は `null` がデフォルト。バックch の step には freq フィールドなし
8. span変更時にパターンを自動複製するため、同じpatIdxを参照する他セルに影響しない
9. `repackEvents` は位置保持型。隙間を詰めたい場合は明示的に fill gap を使用
10. Pool表示は `shouldShowInPool()` でフィルタ（ブランク/同名/varySuffix）
11. `curStep` と `rtCurStep` は独立カウンター。Back/RTで異なるpatternLengthをサポート
12. `doStop(reset)` で `rtCurStep = 0` もリセットされる

---

## 15. ロードマップ

```
v6.0〜v7.0     ✅ 完了
Phase 4        📋 RT Vary + パターングループ
Phase 5        📋 Undo / クリックノイズ除去 / Mode 2 / 3連混在
```

---

## 16. 外部依存

lamejs (MP3エンコード) のみ。Vanilla JS、単一HTML。

---

## 17. テーマ一覧

dark, teal, ocean, light, **sakura**(デフォルト), mint, sky, cream, lavender, peach
