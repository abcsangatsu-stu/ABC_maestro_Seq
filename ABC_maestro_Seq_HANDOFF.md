# ABC Maestro SEQ — プロジェクト引き継ぎドキュメント

**最終更新:** 2026-03-20
**ステータス:** v7.5（インターリーブSong / ステップ数表示 / S2Sコピペ / ステップベースLoop / パステル28色 / RT小文字命名）
**最新ファイル:** `ABC_maestro_Seq_v7_5.html`（単一HTML、約8040行）
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
song = [{ patIdx, repeat, span, sectionColor?, marker?, groupId?, groupRepeat?, lineBreak? }]  // v7.5
globalAudio = [{ freq, volume, pan } × 3]
```

### RT（1ch）
```javascript
rtPatterns = [{ name, channels:[rtCh0], baseName, varySuffix, patternLength,
                coverBackFrom:-1, coverBackTo:-1 }]
rtState = rtPatterns[currentRTPatternIdx].channels
rtSong = [{ patIdx, repeat, span, sectionColor?, marker?, groupId?, groupRepeat?, lineBreak? }]  // v7.5
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

### Song entry 構造（v7.5〜）
```javascript
{ patIdx: number, repeat: number, span: number,
  sectionColor?: string,      // 背景色（28色、SEC_COLORS参照）
  marker?: number,            // マーカー番号 1〜9（省略可）
  groupId?: string,           // グループID（同一IDのセルがグループ）
  groupRepeat?: number,       // グループリピート回数（デフォルト1）
  lineBreak?: boolean }       // v7.5: この後で手動改行
// span: 1〜8。セル幅とパターン長の倍率。
// repeat: span適用後のパターンを何回繰り返すか。
// _groupRepCount: ランタイム専用。Save時に除外。
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
songPlayhead = 0           // 再生開始位置（Back cell index）
lastSongFocus = 'back'     // 'back' or 'rt'
poolMultiSelect = []       // v7.0: Pool複数選択 [{type,idx}]
songMultiSelect = []       // v7.1: Back Song cell複数選択（Shift+click）
rtSongMultiSelect = []     // v7.1: RT Song cell複数選択
loopPoints = []            // v7.5: ステップベースのループポイント（最大2個、累積ステップ値）
```

---

## 4. チャンネル色（固定）

CH1: `#e11d48`(赤), CH2: `#3b82f6`(青), CH3: `#22c55e`(緑), RT: `#06b6d4`(水色)

---

## 5. UI構成

### SONG欄（v7.5〜 インターリーブ配置）

```
┌─ SONG ─────────────────────────────────────────────────────────────┐
│  SONG    B:2784  R:2784 ✓     [G] [■][■][■]...(28色パレット)       │
│  ┌─ Row Group 1 ──────────────────────────────────────────────┐    │
│  │ B [A x2 ][B x2 ][C x2 ][D x2 ][E x1][C1][C2][C8][+]  1440│    │
│  │   192    192    192    192    96  96  96  96              │    │
│  │ R [_ x2 ][_ x2 ][_ x2 ][_ x2 ][_x1][a ][a1][a3][+]  1440│    │
│  │   192    192    192    192    96  96  96  96              │    │
│  └────────────────────────────────────────────────────────────┘    │
│  ┌─ Row Group 2 ──────────────────────────────────────────────┐    │
│  │ B [D5][D_x3  ][C10]...                                 1344│    │
│  │ R [_ ][b_x3  ][ _ ]...                                 1344│    │
│  └────────────────────────────────────────────────────────────┘    │
│  POOL: A(bold) B(dim) │ RT │ a(bold) b(dim)                       │
└────────────────────────────────────────────────────────────────────┘
```

**インターリーブ配置の詳細:**
- Back行とRT行がペアで表示される（Row Group）
- 各行左端に `B` / `R` ラベル
- 各行右端に行内ステップ合計
- ヘッダーに `B:合計 R:合計 ✓` （不一致時は `▲差分` を赤表示）
- 行の折り返しはコンテナ幅に基づいて自動計算（`computeSongRowGroups()`）
- ウィンドウリサイズ時に自動再計算

**手動改行（v7.5〜）:**
- セル選択状態で **Enter** キー → そのセルの後に改行を挿入/解除（トグル）
- `lineBreak: true` がセットされるとセル下部にアクセント色のバー表示
- Aメロ/Bメロ等のセクション区切りに使用
- Save/Load互換（JSON自動保存）

**ステップ数表示（v7.5〜）:**
- 各セル内にパターンの実効ステップ数（`patternLength × repeat`）を小さく表示
- repeat変更時にリアルタイム更新（行合計・ヘッダー合計も連動）

### BTN ROW（v6.3〜 3段構成）
```
┌─ BTN ROW ────────────────────────────────────────────────────────┐
│ 上段(大): ▶(青) ▶(赤) ■  ↻  ↓  ↑   ← アイコンのみ              │
│ 中段: PAT CLR | SONG CLR | DENSITY [slider] | RANDOM | VARY     │
│ 下段: SONG WAV | SONG MP3 | WAV | MP3 | EXPORT MIDI              │
│       SCORE | 16 SCORE | SETTINGS                                │
└──────────────────────────────────────────────────────────────────┘
```

### CH BLOCKS
```
┌─ CH BLOCKS ──────────────────────────────────────────────────────┐
│  CH1 [M][G] Hz♪▼▲(note) Vol Pan [chain] FILL GAP|CLR|↓COPY|↑S2S│
│  CH2/CH3/RT 同様                                                 │
└──────────────────────────────────────────────────────────────────┘
```

### Song cell span（v7.0〜）

- セル右端にドラッグハンドル `┃` が表示（hover時のみ）
- 右ドラッグで整数倍（1〜8）にリサイズ
- **Back**: span変更時にステップを**複製充填**（A→AA for span=2）
- **RT**: span変更時に空ステップで拡張（新メロディ用）
- パターンは自動複製（同じpatIdxの他セルに影響しない）
- span適用後のパターンは自由に編集可能
- **v7.5**: span幅計算に gap 幅（8px）を含む → Back/RTのセル幅が正確に一致

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
- **v7.5**: Cmd+C で RT ステップコピー時に `lastSongFocus` チェックを除去 → パターン切替後もコピペ可能

### Song cell Cmd+C/V/D

| 操作 | ステップ範囲選択 | ステップ単体選択 | Song cell選択 | どちらもなし |
|------|----------------|----------------|--------------|-------------|
| Cmd+C | **グループコピー** | セルコピー | Song cellコピー | パターン全体コピー |
| Cmd+V | **グループペースト** | 選択位置にペースト | Song cell上書き | Song末尾に追加 |
| Cmd+D | — | セル複製 | Song cell複製(span保持) | — |

**clipboard排他制御（v7.0〜）**: cellClipboard設定時にpatternClipboard=null、逆も同様。

### セル背景色（v7.5〜 28色パステルパレット）

- SONG欄右上にカラーパレット（28色、13px ボタン）
- セル選択状態でパレットの色をクリック → `sectionColor` に設定 → 背景色変更
- Shift+クリック複数選択時は一括変更
- `background: color-mix()` 方式（CSSは動的生成）
- Back / RT 両方対応（`lastSongFocus` で適用先を判定）
- **v7.5**: ブランクセルにも色設定可能（色付きブランクは色表示、色なしブランクはグレー半透明）

**SEC_COLORS（28色）:**
```
none, red, red-lt, coral, orange, orange-lt, peach,
yellow, yellow-lt, lime, green, green-lt, mint,
teal, cyan, blue, blue-lt, sky, indigo,
purple, purple-lt, lavender, pink, pink-lt, rose,
brown, sand, gray
```

### マーカー（v7.1〜）

- Song entry に `marker: number|null`（1〜9）
- セル右上に**丸型**バッジ（半透明背景）
- **設定**: セルを選択した状態で数字キー（1〜9）を押す
- **解除**: 同じ数字キーをもう一度押す
- **ジャンプ**: セル未選択状態で数字キー → そのマーカーのセルに playhead ジャンプ
- 同一番号は1つのセルにのみ

### ループポイント（v7.5〜 ステップベース）

- Back/RT **共通**の `loopPoints` 配列（最大2個、**累積ステップ数**で格納）
- セル間の隙間（`.song-gap` 要素）をダブルクリック → ループポイント設置
- gap要素に `data-gap-pos`（ステップ位置）を付与
- 2つのポイント間がループ範囲。3つ目を置くと最も近いポイントを置換
- ループポイントクリックで削除、ドラッグ移動可能
- **再生時**: ステップ位置 → セルインデックスに変換（`getLoopRangeAsCells()`）してBack/RT独立にループ判定
- **v7.5改善**: セル数が異なるBack/RTでも同じ時間位置でループが切り替わる

### パターングループ（v7.1〜）

- Song entry に `groupId: string|null` と `groupRepeat: number`（デフォルト1）
- **設定**: Shift+クリック複数選択 → Gボタン or G/Cmd+G キー
- **表示**: 緑枠線 + リピート回数バッジ
- **解除**: Uキー
- **再生**: グループ末尾で自動ループ（`_groupRepCount`）

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
- 新規ステップ作成時に直前ステップの freq を自動継承
- グリッドドラッグ移動時に freq を保持

---

## 7. S2S IMPORT / EXPORT（v6.1〜 / v7.5拡張）

### ↑ S2S（インポート）
各CH Block に `↑ S2S` ボタン。フォーマット: `S2S|BPM:120|/4 triangle, /8 ramp_up A3, ...`
REPLACE / APPEND モード。BPM不一致時confirm。
**v7.5**: 3番目のトークンでノート名（`A3`, `F#4` 等）を認識 → `noteNameToFreq()` で周波数に変換。
APPENDモードで既存ステップの freq も保持。

### ↓ COPY（エクスポート / v7.5〜）
各CH Block に `↓ COPY` ボタンを追加。
- 範囲選択がある場合 → 選択ステップのみをS2S形式でクリップボードにコピー
- 範囲選択がない場合 → チャンネル全ステップをコピー
- RTの場合は周波数情報（ノート名）も含む
- `navigator.clipboard.writeText()` でシステムクリップボードに直接コピー

**パターン間コピペのワークフロー:**
1. パターンAのRTで `↓ COPY` → S2Sテキストがクリップボードに
2. パターンBに切り替え
3. `↑ S2S` → APPEND を選択 → ペースト → 既存波形の後ろに追加

---

## 8. 出力

- MIDI: バック ch1-3, RT ch4
- WAV/MP3: パターン単体 or SONG WAV/SONG MP3
- Score: テキスト形式（RT部分にノート名付記）/ 16ステップ表記
- `renderPatternOffline()` / `renderSongOffline()` — RT部分で `step.freq` 優先使用
- **v7.5 bugfix**: `renderSongOffline()` で `flattenSongWithGroups()` を使い、`groupRepeat` をBack/RT両方で展開。ライブ再生とWAV/MP3出力が一致するように修正

---

## 9. Save/Load

現行Save format version: **7**（v7.1〜v7.5共通）。後方互換あり。

- v6以前のデータ: `span` がなければ `1` を補完
- `song` / `rtSong` エントリの `span` は 1〜8 にクランプ
- `freqQuantize` をSave/Loadに含む（なければ `[false,false,false,false]` で初期化）
- `marker`, `groupId`, `groupRepeat`, `lineBreak` は Song entry に含まれ自動保存
- `_groupRepCount` はSave時に除外（ランタイム専用）
- `loopPoints` はSave/Loadに含まれる（v7.5〜）。旧プロジェクトにはフィールドがないため `[]` で初期化
- `sectionColor` は28色全てSave/Load互換

---

## 10. ヘルパ関数

### Span ヘルパ（v7.0〜）

```javascript
getEntrySpan(ent)              // span取得（1〜8にclamp）
makeSongEntry(patIdx, repeat, span)  // span付きエントリ生成
expandPatternSteps(pat, newLen, isRT) // ステップ配列を拡張
cloneAndExpandPattern(pat, newSpan, isRT) // deep clone + 拡張
applySpanChange(entry, songIdx, newSpan, isRT)
startSpanDrag(entry, songIdx, isRT, startEvt)  // gap=8px
getEditMax() / getRTEditMax()
getLaneNCells() / getRTLaneNCells()
```

### Song描画ヘルパ（v7.5〜）

```javascript
// インターリーブ描画
getEffectiveSteps(entry, isRT)       // patternLength × repeat
updateSongTotals()                   // ヘッダーのB:/R:合計更新
createSongGap(stepPos)               // ループポイント対応gap生成（ステップ位置を受け取る）
createSongCell(entry, idx, isRT)     // 統一セル生成（Back/RT共通）
computeSongRowGroups()               // 行グループ計算（幅+lineBreak考慮）
renderInterleavedSong()              // メイン描画（ロック付き）
_renderInterleavedSongInner()        // 描画本体

// ステップ⇔セル変換
getSongCumulativeSteps(songArr, pats, isRT)  // 累積ステップ配列
// ステップ→セル / セル→ステップは getLoopRangeAsCells() 内および
// doSongPlay() 内でインラインforループにより計算（関数化はされていない）
```

### ループポイントヘルパ（v7.5〜）

```javascript
addLoopPoint(pos)                    // ステップ位置で追加
removeLoopPoint(pos)                 // 削除
getLoopRange()                       // {start, end} ステップ位置 or null
getLoopRangeAsCells(songArr, pats, isRT)  // セルインデックスに変換
startLoopPointDrag(origPos, startEvt)
```

### S2Sヘルパ（v7.5〜）

```javascript
copyChToClipboard(chIdx, isRT)       // ↓COPYボタン。クリップボードにS2Sテキスト出力
noteNameToFreq(noteName)             // "A3" → 220Hz 等
freqToNoteName(freq)                 // 220Hz → "A3" 等
```

### その他

```javascript
flattenSongWithGroups(songArr)       // groupRepeatを展開した平坦配列を返す（エクスポート用）
renderSongColorDropdown()            // 28色パレット描画
generateGroupId()                    // ユニークなgroupId生成
groupSelectedCells(isRT)             // 選択セルをグループ化
ungroupCell(isRT, idx)               // グループ解除
setMarker(num, isRT)                 // マーカー設定
jumpToMarker(num)                    // マーカージャンプ
```

---

## 11. repackEvents 動作（v7.0〜位置保持型）

```
v6.x: 全イベントを位置0から隙間なく再配置（ギャップ消失）
v7.0: 各イベントの元位置を保持し、重複分のみ後方シフト
```

---

## 12. 再生ロジック（v7.0〜 独立カウンター方式 / v7.5 ステップベース同期）

### ステップカウンター

```javascript
curStep    // Back用。patLen到達で0リセット
rtCurStep  // RT用（v7.0〜）。rtPatLen到達で0リセット
```

### 途中再生（v7.5〜 ステップベース同期）

```javascript
// doSongPlay()
songPos = songPlayhead（Back cell index）
// Backの累積ステップ数をインラインforループで計算
backStepPos = Σ(patterns[song[i].patIdx].patternLength × repeat) for i < songPos
// RTの対応セルをインラインforループで探索
rtStepAccum を積み上げ、backStepPos を超えるセルを rtSongPos に設定
rtCurStep = 0（セル先頭から再生）
```

BackとRTは**独立して進行**するが、途中再生時はBackの開始ステップ位置からRTの対応セルを計算して同期する。

### ループ判定（v7.5〜）

```javascript
var backLoop = getLoopRangeAsCells(song, patterns, false);
if (backLoop && songPos > backLoop.end) songPos = backLoop.start;

var rtLoop = getLoopRangeAsCells(rtSong, rtPatterns, true);
if (rtLoop && rtSongPos > rtLoop.end) rtSongPos = rtLoop.start;
```

ステップベースの `loopPoints` → セルインデックスに変換してBack/RT独立にループ判定。

---

## 13. RT命名規則（v7.5〜）

- **新規RTパターン名**: 小文字 `a, b, c, ... z, a2, b2, ...`（v7.1以前は `RA, RB, ...`）
- **Backパターン名**: 大文字 `A, B, C, ...`（変更なし）
- 既存パターンの名前は自動変更されない

---

## 14. バージョン履歴

### v6.0（2026-03-01）
Phase 3+完了。全基本機能実装。

### v6.1（2026-03-01）
S2S IMPORT追加。

### v6.2（2026-03-08）
Song cell Cmd+D/C/V、ブランクセル、SONG WAV/MP3、周波数クオンタイズ、バグ修正多数。

### v6.3（2026-03-08）
btn-row 3段化、音名ラベル常時表示、RTステップ別freq。

### v7.0（2026-03-08）
Song cell span、repackEvents位置保持、Poolフィルタリング、Step grid改善、RT独立カウンター、グループコピペ。

### v7.1（2026-03-08）
セル背景色、マーカー、ループポイント、パターングループ、freqQuantize Save/Load、Song cell複数選択。

### v7.5（2026-03-20）
- **インターリーブSong欄**
  - Back/RTをペアで表示（Row Group方式）
  - 各セルにステップ数表示、行合計、ヘッダーに総合計と不一致警告
  - `renderInterleavedSong()` でBack/RT統一描画
  - `renderSongLane()` / `renderRTSongLane()` はラッパーに変更
  - 再生ハイライトは `data-back-song-idx` / `data-rt-song-idx` 属性で特定
  - ウィンドウリサイズ時に行グループ自動再計算
- **手動改行**
  - Song entry に `lineBreak: boolean`（Enterキーでトグル）
  - セル下部にアクセント色バーで視覚表示
- **パステル28色パレット**
  - SEC_COLORS を10色→28色に拡大（パステル系・濃淡バリエーション）
  - CSSクラスを動的生成方式に変更
  - カラーボタンサイズ縮小（16px→13px）
  - ブランクセルにも色設定可能
- **↓ COPY（S2Sエクスポート）**
  - 各CH Block に `↓ COPY` ボタン追加
  - 範囲選択時は選択部分のみ、なければ全ステップをS2Sテキストでクリップボードにコピー
  - RT周波数情報（ノート名）も含む
  - `↑ S2S` の APPEND モードと組み合わせてパターン間コピペ
- **S2S IMPORT拡張**
  - 3番目トークンでノート名認識（`/8 triangle A3`）
  - `noteNameToFreq()` 追加
  - APPENDモードで既存ステップのfreqも保持
- **ステップベースループポイント**
  - `loopPoints` がセルインデックス → 累積ステップ位置に変更
  - `getSongCumulativeSteps()` / `getLoopRangeAsCells()` 追加
  - `getLoopRangeAsCells()` でBack/RT独立にセルインデックス変換
  - gap要素に `data-gap-pos`（累積ステップ位置）を付与
- **途中再生のBack/RT同期**
  - `doSongPlay()` でBackの開始ステップ位置からRTの対応セル・オフセットを計算
  - Back/RTセル数やステップ長が異なっても正しい位置から再生
- **span幅修正**: gap幅を4px→8px。Back/RTのセル幅が正確に一致
- **Cmd+Cコピー修正**: RT単一ステップコピー時の `lastSongFocus` チェック除去
- **RT命名小文字化**: 新規RTパターンは `a, b, c, ...`（既存は変更なし）
- **bugfix**: `renderSongOffline()` で `groupRepeat` が無視されていた問題を修正（`flattenSongWithGroups()` 追加）
- **bugfix**: `loopPoints` が `saveProject()` / `loadProject()` に含まれていなかった問題を修正
- **bugfix**: セル削除時に `loopPoints` をセルインデックスではなくステップ数で正しくシフトするように修正
- **bugfix**: スコア生成にも `groupRepeat` 情報を表示するように修正

---

## 15. 既知の注意点

1. `state`/`rtState` はエイリアス。パターン切替時に再バインド
2. バック3ch既存コード（c<3ループ30箇所以上）は変更なし
3. 休符: `on:true` + `wave:'low'`（`on:false` は空セル）
4. ブランクパターン: `name===''`, `baseName===''`。Song欄でダブルクリックリネーム可能。色設定可能（v7.5）
5. freqQuantize はv7.1でSave/Load対応済み
6. Cmd+V Song cell上書きは `lastSongFocus` + `selectedSongCell >= 0` の両方が必要
7. RT step.freq は `null` がデフォルト。バックch の step には freq フィールドなし
8. span変更時にパターンを自動複製するため、同じpatIdxを参照する他セルに影響しない
9. `repackEvents` は位置保持型。隙間を詰めたい場合は明示的に fill gap を使用
10. Pool表示は `shouldShowInPool()` でフィルタ（ブランク/同名/varySuffix）
11. `curStep` と `rtCurStep` は独立カウンター。Back/RTで異なるpatternLengthをサポート
12. `doStop(reset)` で `rtCurStep = 0` もリセットされる
13. `loopPoints` はBack/RT共通。**ステップ位置**で格納（v7.5〜）。Save/Load対応済み
14. `_groupRepCount` はランタイム専用。Save時に `song.map()` で除外
15. Song cell複数選択（`songMultiSelect`/`rtSongMultiSelect`）は Shift+クリックで操作
16. LOOPボタンはアイコンのみ（↻）。`active` クラスで色が変わることでON/OFFを明示
17. `renderSongLane()` / `renderRTSongLane()` は `renderInterleavedSong()` のラッパー（v7.5〜）
18. セル背景色CSSは動的生成。`SEC_COLOR_MAP` に色を追加するだけで拡張可能（v7.5〜）
19. ↓ COPY でクリップボードAPI不可時は `prompt()` フォールバック

---

## 16. ロードマップ

```
v6.0〜v7.0     ✅ 完了
v7.1           ✅ 完了
v7.5           ✅ 完了（インターリーブSong / ステップ表示 / S2Sコピペ / ステップLoop / パステル28色 / RT小文字 / 途中再生同期 / 手動改行）
Phase 4+       📋 Undo / span変更時ビジュアルフィードバック
Phase 5        📋 Mode 2 / 3連混在（ペンディング）
ペンディング    🔒 クリックノイズ除去 / RT Vary / パターン一括transpose
```

---

## 17. 外部依存

lamejs (MP3エンコード) のみ。Vanilla JS、単一HTML。

---

## 18. テーマ一覧

dark, teal, ocean, light, **sakura**(デフォルト), mint, sky, cream, lavender, peach
