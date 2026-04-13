# 間取りデザインメモ — 仕様書

**バージョン:** v0.01  
**リポジトリ:** https://github.com/yuttan/HouseDesigner  
**公開URL:** https://yuttan.github.io/HouseDesigner/  
**最終更新:** 2026-04-14

---

## 概要

間取り図（PNG / JPG / WEBP / PDF）を読み込み、図面上にピンを立てて素材・照明・寸法などのメモを管理するシングルページWebアプリ。ビルドステップなし・依存ライブラリ最小（PDF.jsのみ）の単一HTMLファイル構成。

---

## 技術スタック

| 項目 | 内容 |
|------|------|
| 実装 | HTML5 / CSS3 / Vanilla JavaScript（単一ファイル `index.html`） |
| Canvas | 2層構成（`#floorplan-canvas` + `#draw-canvas`） |
| PDF | PDF.js 3.11.174（CDN） |
| ストレージ | localStorage（自動保存） / GitHub Gist（クラウド同期） |
| ホスティング | GitHub Pages（`main` ブランチ） |
| 開発サーバー | `python -m http.server 3000`（LAN確認用） |

---

## ファイル構成

```
HouseDesigner/
├── index.html          # アプリ本体（全コード）
├── SPEC.md             # 本仕様書
└── .claude/
    └── launch.json     # Claude Code 開発サーバー設定
```

---

## 状態管理（state オブジェクト）

```js
state = {
  // アノテーション
  annotations: [],      // ピン＋メモの配列
  nextId: 1,            // 次に割り当てる内部ID
  catCounters: {        // カテゴリ別連番カウンター
    wall, floor, ceiling, light, door, window, other
  },
  selectedId: null,
  editingId: null,
  filterCat: 'all',
  pendingX: 0, pendingY: 0,  // ピン追加予定座標（画像座標）

  // 画像・表示
  image: null,          // HTMLImageElement
  imageScale: 1,
  offsetX: 0, offsetY: 0,

  // PDF
  pdfDoc: null,
  pdfPage: 1,
  pdfTotal: 0,

  // クロップ
  crop: null,           // { x, y, w, h }（画像座標）
  cropSelecting: false,
  cropStart: null,
  cropRect: null,

  // 描画
  strokes: [],
  currentStroke: null,
  drawColor: '#c8956c',
  drawWidth: 2,
  drawMode: 'pen',

  // テーマ
  accentColor: '#c8956c',

  // UI
  mode: 'pin',
}
```

---

## カテゴリ定義（CAT_INFO）

| key | 表示名 | アイコン | カラー | ピンIDプレフィックス |
|-----|--------|----------|--------|---------------------|
| wall | 壁 | 🧱 | #4a90e2 | 壁 |
| floor | 床 | 🪵 | #7ed321 | 床 |
| ceiling | 天井 | ⬆ | #bd10e0 | 天 |
| light | 照明 | 💡 | #f5a623 | 灯 |
| door | ドア | 🚪 | #d0021b | 扉 |
| window | 窓 | 🪟 | #50e3c2 | 窓 |
| other | その他 | 📌 | #9b9b9b | 他 |

---

## 機能一覧

### 1. 図面読み込み

- 対応形式：PNG / JPG / WEBP / PDF
- 入力方法：ファイル選択ダイアログ、ドラッグ＆ドロップ
- PDF：PDF.js でレンダリング（scale=2 で高解像度）。複数ページ対応（◀▶ボタン）
- 読み込み後に `fitImage()` で画面に収まるスケールに自動調整
- **注意:** 画像データはlocalStorageに保存されないため、再読み込み時は再度ファイルを開く必要がある

### 2. ズーム・パン

| 操作 | 動作 |
|------|------|
| マウスホイール | ズーム（カーソル中心） |
| `+` / `-` ボタン | ズーム |
| `⊙` ボタン | 全体表示リセット |
| ✋ パンモード + ドラッグ | パン |
| Space + ドラッグ | 一時パン |
| 中ボタンドラッグ | パン |
| 2本指ピンチ（タッチ） | ズーム |
| パンモード + 1本指ドラッグ（タッチ） | パン |

- ズーム範囲：5% 〜 1000%
- `clampOffset()` により図面が画面外に完全に出ないよう制限（80px マージン）
- クロップ中はクロップ範囲外のパンは制限なし

### 3. 高DPI対応

`devicePixelRatio` に基づきcanvas物理サイズを拡大。  
全描画は `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` でCSS座標系に統一。  
→ Retinaディスプレイ・モバイルで鮮明表示。

### 4. ピン追加・管理

#### ピンID（カテゴリ別連番）

ピン作成時、カテゴリのプレフィックス＋連番を `pinId` として付与。  
例：壁1, 壁2, 床1, 天1, 灯1, 扉1, 窓1, 他1  
- 削除しても連番は詰めない（付与時固定）
- JSON/Gist読み込み時は `recomputeCatCounters()` で既存pinIdから自動復元

#### ピンの操作

| 操作 | 動作 |
|------|------|
| 📍ピン追加モード + クリック/タップ | メモ追加ダイアログを開く |
| ピンをクリック | 選択（再クリックで解除） |
| ピンをダブルクリック | 編集ダイアログ |
| ピンをドラッグ（マウス/タッチ） | 位置変更 |
| Delete / Backspace キー | 選択中ピンを削除 |

#### アノテーションデータ構造

```js
{
  id: Number,          // 内部一意ID（削除後も再利用しない）
  pinId: String,       // 表示用ID（例: "壁1"）
  x: Number,           // 画像座標 X
  y: Number,           // 画像座標 Y
  cat: String,         // カテゴリキー
  room: String,        // 場所・部屋名
  material: String,    // 素材・商品名
  maker: String,       // メーカー・品番
  color: String,       // 色・仕上げ
  note: String,        // 備考
  lightType: String,   // 照明種別（cat=light のみ）
}
```

### 5. メモ一覧（サイドバー）

- カテゴリフィルター（すべて / 壁 / 床 / 天井 / 照明 / ドア / 窓 / その他）
- カード表示：pinID バッジ、カテゴリ色、場所名、素材・品番等
- 「📍 移動」ボタン：該当ピンを画面中央に移動＋選択
- 「編集」ボタン：編集ダイアログ
- 「✕」ボタン：削除（確認ダイアログあり）
- 折りたたみ可能（右下FABボタン ▶/📋 で開閉）
- 画面幅 < 768px の場合は初期状態で折りたたまれる

### 6. クロップ機能

- ✂ クロップモード でドラッグ → 矩形選択
- 「クロップ適用」で表示範囲を固定（ピンもクロップ外は非表示）
- 「📷 画像を保存」でクロップ範囲を PNG としてダウンロード
- 「✂ クロップ中 ✕ 解除」で解除
- クロップ外へのピン追加は不可

### 7. お絵かき機能（✏️ 描画モード）

#### 描画ツール

| ツール | 操作 |
|--------|------|
| ✏️ ペン | フリーハンド（逐次描画） |
| 🧹 消しゴム | `destination-out` で消去 |
| ╱ 直線 | ドラッグで始点→終点 |
| ▭ 四角 | ドラッグで矩形 |
| ◯ 丸 | ドラッグで楕円 |
| ● 点 | クリックで円点 |

- 線の太さ：細(2px) / 中(5px) / 太(10px)
- 色：カラーピッカー（初期値：アクセントカラー）
- ↩ 戻す：最後のストローク取り消し
- 🗑 全消去：全ストローク削除（確認ダイアログ）
- ストロークは**画像座標**で保存 → ズーム・パンに追従
- `drawStroke(ctx, stroke, previewEnd)` ヘルパーで一元描画
- シェイプツール（直線/四角/丸）はドラッグ中にリアルタイムプレビュー
- タッチ対応

#### ストロークデータ構造

```js
{
  tool: 'pen' | 'eraser' | 'line' | 'rect' | 'circle' | 'dot',
  color: String,
  width: Number,
  eraser: Boolean,
  points: [{ x: Number, y: Number }],  // 画像座標
}
```

### 8. データ保存・共有

#### localStorage（自動保存）

- キー：`housedesigner_v1`
- 保存タイミング：ピン追加/編集/削除/移動、ストローク確定、クロップ変更、JSON読み込み
- 保存内容：annotations / nextId / strokes / crop / imageScale / offsetX / offsetY
- 保存時に「✓ 自動保存済み」インジケーターを1.5秒表示
- **画像データは保存しない**

#### JSON書き出し / 読み込み

- 書き出し：全データをJSONファイルにダウンロード（バージョン番号 `version: 3` 付き）
- 読み込み：JSONファイルを選択して状態を復元

```js
// JSON構造
{
  version: 3,
  annotations: [...],
  nextId: Number,
  imageScale: Number,
  offsetX: Number,
  offsetY: Number,
  crop: Object | null,
  strokes: [...],
}
```

#### GitHub Gist 同期

- キー：`housedesigner_gist`（token / gistId / desc を保存）
- ⚙ Gist設定でPAT（Personal Access Token / gistスコープ）とGist IDを設定
- ☁ Gist保存：初回POST → 以降PATCH（同一Gist IDを更新）
- ☁ Gist読込：Gist IDを指定してGETで取得・復元
- Gistファイル名：`housedesigner.json`
- トークンはこの端末のlocalStorageにのみ保存（他端末には共有されない）

### 9. アクセントカラー設定

- ☰ メニュー → 🎨 アプリ設定
- カラーピッカー + 6色プリセット
- `applyAccent(color)` が `<style id="accent-style">` を動的書き換え
- canvas上の描画（クロップ枠等）は `state.accentColor` を参照
- localStorageキー：`housedesigner_accent`

**デフォルト色:** `#c8956c`（ウォームベージュ）  
**プリセット:**
- `#c8956c` デフォルト（ウォームベージュ）
- `#4f454f` ダークパープル
- `#4a90e2` ブルー
- `#7ed321` グリーン
- `#e94560` レッド
- `#50e3c2` ティール

### 10. ハンバーガーメニュー（☰）

| 項目 | 機能 |
|------|------|
| 📤 書き出し (JSON) | JSONダウンロード |
| 📥 読み込み (JSON) | JSONファイル読み込み |
| ☁ Gist保存 | GitHub Gistに保存 |
| ☁ Gist読込 | GitHub Gistから読み込み |
| ⚙ Gist設定 | PAT・Gist ID設定 |
| 🖨 印刷/PDF | ブラウザ印刷（サイドバー等を非表示） |
| 🎨 アプリ設定 | アクセントカラー変更 |
| 📖 使い方 | 操作説明モーダル |

### 11. モバイル対応

- `user-scalable=no` でブラウザズームを無効化（ピンチは内部で処理）
- 2本指ピンチ：ズーム
- パンモード + 1本指ドラッグ：パン
- ピン追加モード + タップ：ピン追加
- ピンタッチドラッグ：ピン位置変更
- 描画モード：touchstart/touchmove/touchend 完全対応
- クロップモード：タッチ対応
- 画面幅 < 768px：サイドバーを初期折りたたみ

### 12. キーボードショートカット

| キー | 動作 |
|------|------|
| P | 📍 ピン追加モード |
| H | ✋ パンモード |
| S | ↖ 選択モード |
| C | ✂ クロップモード |
| D | ✏️ 描画モード |
| + / = | ズームイン |
| - | ズームアウト |
| 0 | 全体表示リセット |
| Space（ホールド） | 一時パンモード |
| Delete / Backspace | 選択中ピンを削除 |
| Escape | 選択解除 / クロップキャンセル / モーダルを閉じる |

---

## 座標系

| 系 | 説明 |
|----|------|
| 画像座標 | 元画像のピクセル座標。ズーム・パン不変。ピン・ストロークはこれで保存 |
| スクリーン座標 | canvas上の表示座標（CSS px）。`imageToScreen()` / `screenToImage()` で変換 |
| 物理ピクセル座標 | canvas内部座標（DPR倍）。描画APIはこれを使用 |

```js
function screenToImage(sx, sy) {
  return { x: (sx - state.offsetX) / state.imageScale,
           y: (sy - state.offsetY) / state.imageScale };
}
function imageToScreen(ix, iy) {
  return { x: ix * state.imageScale + state.offsetX,
           y: iy * state.imageScale + state.offsetY };
}
```

---

## Canvas描画構成

```
canvasArea（div）
├── #floorplan-canvas  ← 図面画像・クロップオーバーレイ・ピン（DOM要素）
│   └── ctx.setTransform(dpr,0,0,dpr,0,0) → redraw()
└── #draw-canvas       ← 描画ストローク（pointer-events:none）
    └── drawCtx.setTransform(dpr,0,0,dpr,0,0) → redrawStrokes()
```

`redraw()` フロー：
1. `clampOffset()` でパン範囲制限
2. `ctx.setTransform(dpr,0,0,dpr,0,0)`
3. 画像描画
4. クロップオーバーレイ描画
5. `updatePins()` でピンDOM更新
6. `redrawStrokes()` で描画レイヤー更新

---

## 既知の制限・注意事項

- 画像データはlocalStorageに保存されない（容量制限のため）
- ページ更新・再読み込み後は同じ画像ファイルを手動で再読み込みが必要
- Gist APIはCORS対応のためブラウザから直接アクセス（サーバーレス）
- PDF の複数ページ読み込み時、ページ変更するとストローク・クロップはリセットされる
- `localStorage` の容量上限（通常5MB）を超えるとストロークが保存できない場合がある
