# edjiki 詳細設計書 — フォント拡張・設定入出力

対応する概要設計: [design-font-and-settings-io.md](design-font-and-settings-io.md)（§8 決定事項確定済み）
作成日: 2026-07-02 / ステータス: 実装可能ドラフト

行番号アンカーはすべて **v0.8.0（commit 69bd7a1）時点の `edjiki.html`** を指す。実装が進むとずれるため、照合には各節に併記した検索キー（`grep` 可能な文字列）を優先すること。

---

## 0. 改修ポイント総覧

| # | 現行位置 | 検索キー | 変更内容 | 対象機能 |
|---|---|---|---|---|
| 1 | L8 | `fonts.googleapis.com/css2` | URL に `:wght@400;700` 追加（7 書体） | F2 |
| 2 | L155–169 | `.entry-body {` | `font-weight` / `-webkit-text-stroke` / `paint-order` の 3 行追加 | F2 F3 |
| 3 | L531 付近 | `.font-option {` | `#font-preview`・`.outline-options`・カスタムフォント行の CSS 追加 | F1 F3 プレビュー |
| 4 | L702–827 | `id="settings-panel"` | プレビュー行・太字/疑似ボールド UI・カスタムグループ・バックアップ行を追加 | 全機能 |
| 5 | L827 直後 | `id="drive-settings-overlay"` の前 | フォント追加ダイアログ（`#font-add-overlay`）を追加 | F1 |
| 6 | L892 | `id="file-input"` | 隣に `#settings-file-input`（JSON 用）と `#font-file-input`（フォント用）を追加 | F1 F4 |
| 7 | L2756 直前 | `// ========== Settings ==========` の前 | 新セクション `// ========== Custom Fonts ==========` を挿入 | F1 |
| 8 | L2758–2771 | `const FONT_FAMILIES` | 参照解決を `resolveFontFamily()` 経由に変更 | F1 |
| 9 | L2781 | `SETTINGS_DEFAULTS` | `fontBold` / `fontOutline` / `customFonts` を追加 | F2 F3 F1 |
| 10 | L2782–2796 | `let settings = (() => {` | `customFonts` の配列ガードを追加 | F1 |
| 11 | L2803–2824 | `function applySettings()` | CSS 変数 `--entry-font-weight` / `--entry-text-stroke` の設定を追加、font-family を `resolveFontFamily()` に | F2 F3 F1 |
| 12 | L2864–2887 | `function openSettings()` | 新設ウィジェットの状態同期を追加 | 全機能 |
| 13 | L3053–3070 | `$fontSizeSlider.addEventListener` 付近 | 太字チェック・疑似ボールド選択・追加/削除・書出/読込のリスナーを追加 | 全機能 |
| 14 | L2134 節 | `// ========== Download / Import ==========` | `exportSettings()` / `importSettings()` を追加 | F4 |
| 15 | L2852 | `applySettings();`（初回呼出） | 直後に `loadCustomFonts()`（非同期・非ブロッキング）を追加 | F1 |

CLAUDE.md（Architecture 表・Settings shape・Key invariants・ショートカット表は変更なし）と README の更新も各リリースに含める。

---

## 1. 共通基盤の変更

### 1.1 Google Fonts URL（L8）

700 ウェイトを持つ 7 書体に `:wght@400;700` を付与。さわらび明朝・ひな明朝は 400 のみのため据え置き。

```html
<link href="https://fonts.googleapis.com/css2?family=BIZ+UDPGothic:wght@400;700&family=M+PLUS+1p:wght@400;700&family=M+PLUS+Rounded+1c:wght@400;700&family=Noto+Sans+JP:wght@400;700&family=Noto+Serif+JP:wght@400;700&family=Zen+Old+Mincho:wght@400;700&family=Shippori+Mincho:wght@400;700&family=Sawarabi+Mincho&family=Hina+Mincho&display=swap" rel="stylesheet">
```

`unicode-range` サブセット配信のため 700 は太字使用時まで実ダウンロードされない（概要 §2.2・決定事項 4）。

### 1.2 SETTINGS_DEFAULTS（L2781）

```js
const SETTINGS_DEFAULTS = { fontSize: 16, lineHeight: 1.6, fontKey: "biz-ud", theme: "auto",
  daysToKeep: 60, lineEnding: "LF", dateFormat: "slash", isoTzMode: "stored", isoTzFixed: "+09:00",
  autoFullscreenOnNew: false, driveAutoSave: true,
  fontBold: false,                // F2
  fontOutline: "none",            // F3: "none"|"weak"|"medium"|"strong"
  customFonts: [] };              // F1: [{ key, label, source, family }]
```

settings ローダー（L2786 `const parsed = JSON.parse(raw);` の直後）にガードを 1 行追加:

```js
if (!Array.isArray(parsed.customFonts)) delete parsed.customFonts;
```

マイグレーション不要（スプレッドで既定値が補完される。既存の dateSep 移行と同居可）。

### 1.3 CSS 変数と `.entry-body`（L155–169）

`.entry-body` に 3 行追加。textarea は通常・全画面とも `entry-body` クラスを持つ（L1762 `ta.className = "entry-body" + (isFS ? " entry-body-fs" : "")`）ため、ここ 1 箇所で両モードをカバーする。

```css
.entry-body {
  /* …既存宣言… */
  font-weight: var(--entry-font-weight, 400);
  -webkit-text-stroke: var(--entry-text-stroke, 0) currentColor;
  paint-order: stroke fill;
}
```

`currentColor` は `.entry-body` の `color: var(--fg)` を継承するため、テーマ切替に自動追従する。`paint-order` 非対応環境（Chrome 122 以前等）では宣言が無視されストロークが字画中心に乗るが、線幅が細く実害小（概要 §2.3）— JS での機能検出・分岐は行わない。

### 1.4 `applySettings()`（L2803）

```js
const OUTLINE_WIDTHS = { none: "0", weak: "0.015em", medium: "0.03em", strong: "0.05em" };

function applySettings() {
  const r = document.documentElement;
  r.style.setProperty("--entry-font-size",   settings.fontSize + "px");
  r.style.setProperty("--entry-font-family", resolveFontFamily(settings.fontKey));   // 変更（§4.2）
  r.style.setProperty("--entry-line-height", String(settings.lineHeight));
  r.style.setProperty("--entry-font-weight", settings.fontBold ? "700" : "400");     // 追加
  r.style.setProperty("--entry-text-stroke", OUTLINE_WIDTHS[settings.fontOutline] || "0"); // 追加
  /* …テーマ処理（既存のまま）… */
}
```

`OUTLINE_WIDTHS` の値は下記プロトタイプで妥当性確認済み。微調整が要ればこの 1 テーブルの変更で完結する。

### 1.4.1 F3 プロトタイプ検証結果（2026-07-02・Chromium 145 headless）

概要設計 §8 で唯一の技術ゲートだった「textarea 内で `-webkit-text-stroke` + `paint-order: stroke fill` が効くか」を、本番相当の textarea・実フォント（Hina/Sawarabi/Shippori Mincho）・4 段階・16/24px で描画検証した。結果は**全項目 PASS**:

- **機能サポート**: `CSS.supports('paint-order','stroke fill')` = true、`-webkit-text-stroke` = true。対象環境（Chrome 123+）を大きく上回る 145 で確認。
- **本命**: textarea 内テキストにストロークが乗り、「強」(0.05em) で全書体が明確に太る。**字画のふところ（物・語・給などの内側の空白）は潰れず開いたまま**均一に太る＝ paint-order が塗りの背面にストロークを回せている。
- **フォールバック**: `paint-order: normal`（非対応ブラウザ相当）でも 0.05em では大きな破綻はなく可読。→ 機能検出による JS 分岐は不要という設計判断を裏付け。
- **キャレット/選択**: テキスト選択の反転ハイライトが正常に描画され、ストローク適用テキストも選択反転下で可読。編集操作への副作用なし。
- **F2 との役割分担の裏取り**: 本物の 700 を持つ書体（Shippori/Noto Serif）では、擬似ボールド（均一に太る）より実 700（縦太・横細のコントラストを保持）の方が美しい。→ 「700 保有書体は F2、非保有書体（さわらび/ひな/取込）は F3」という設計の棲み分けが視覚的に正しいと確認できた。

検証物: `scratchpad/f3-proto.html`（単体で `file://` でも動作）+ `f3-shot.py`（Playwright スクショ）。結論として **F3 は設計どおり実装可能で、設計を無効化するリスクは解消**。

---

## 2. F2: ボールド設定

### 2.1 HTML（フォント欄 `.font-options` 閉じタグ L774 の直後）

```html
<label style="font-size:12px;display:flex;align-items:center;gap:8px;cursor:pointer;margin-top:8px">
  <input type="checkbox" id="setting-font-bold"> 太字で表示
</label>
<div id="font-bold-note" class="settings-note hidden">※ このフォントは太字データを持たないため簡易表示になります。疑似ボールドの利用がおすすめです</div>
```

```css
.settings-note { font-size: 11px; color: var(--muted); margin-top: 2px; }
```

### 2.2 JS（Settings 節）

```js
// 本物の 700 ウェイトを持たない書体（合成ボールドに落ちる）
const BOLD_SIMULATED = new Set(["sawarabi-mincho", "hina-mincho"]);
function updateFontBoldNote() {
  const simulated = BOLD_SIMULATED.has(settings.fontKey) || settings.fontKey.startsWith("cf-");
  document.getElementById("font-bold-note")
    .classList.toggle("hidden", !(settings.fontBold && simulated));
}
```

- リスナー（L3063 の radio リスナー群の近くに追加）: `change` → `settings.fontBold = checked` → `applySettings(); saveSettings(); resizeAll(); updateFontBoldNote();`
- `openSettings()`（L2864）に `document.getElementById("setting-font-bold").checked = !!settings.fontBold; updateFontBoldNote();` を追加。
- フォント radio 変更時（既存 L3063–3070 のハンドラ内）にも `updateFontBoldNote()` を呼ぶ（書体を替えた瞬間に注記が追従）。
- 汎用 serif / sans-serif / monospace は OS フォント依存だが通常 700 を持つため `BOLD_SIMULATED` に含めない。
- `resizeAll()` 必須: font-weight はグリフ幅を変え折返し行数が変わるため（概要 §2.3 補足）。

## 3. F3: 疑似ボールド

### 3.1 HTML（太字チェックの直後）

```html
<label class="settings-label" style="margin-top:8px">疑似ボールド</label>
<div class="outline-options" id="outline-options"></div>
```

```css
.outline-options { display: flex; gap: 6px; }
.outline-options button {
  flex: 1; padding: 6px 0; font-size: 12px; font-family: inherit;
  background: var(--card); color: var(--fg);
  border: 1px solid var(--border); border-radius: 6px; cursor: pointer;
}
.outline-options button.selected { border-color: var(--accent); box-shadow: 0 0 0 1px var(--accent); }
```

### 3.2 JS — `renderOutlineOptions()`（`renderThemeOptions` L2825 と同型）

```js
const OUTLINE_LEVELS = [
  { key: "none",   label: "なし" },
  { key: "weak",   label: "弱"   },
  { key: "medium", label: "中"   },
  { key: "strong", label: "強"   },
];
function renderOutlineOptions() {
  const container = document.getElementById("outline-options");
  container.innerHTML = "";
  OUTLINE_LEVELS.forEach(l => {
    const btn = document.createElement("button");
    btn.className = settings.fontOutline === l.key ? "selected" : "";
    btn.textContent = l.label;
    btn.addEventListener("click", () => {
      settings.fontOutline = l.key;
      applySettings(); saveSettings(); renderOutlineOptions();
    });
    container.appendChild(btn);
  });
}
```

- `openSettings()` に `renderOutlineOptions();` を追加。
- `resizeAll()` は不要（ストロークはメトリクス不変）だが、F2 と共通ハンドラにしても害はない。

## 4. プレビュー行と F1: カスタムフォント

### 4.1 ライブプレビュー行（フォント欄 settings-row L720 の直前に挿入）

```html
<div class="settings-row">
  <label class="settings-label">プレビュー</label>
  <div id="font-preview"></div>
</div>
```

```css
#font-preview {
  background: var(--card); border: 1px solid var(--border); border-radius: 6px;
  padding: 8px 10px; color: var(--fg);
  font-family: var(--entry-font-family, 'BIZ UDPGothic', sans-serif);
  font-size: var(--entry-font-size, 16px);
  line-height: var(--entry-line-height, 1.6);
  font-weight: var(--entry-font-weight, 400);
  -webkit-text-stroke: var(--entry-text-stroke, 0) currentColor;
  paint-order: stroke fill;
}
```

日付部は**当日日付を動的表示**（プレースホルダ `yyyy/mm/dd` ではなく実際の本文エントリと同じ見た目）。フォント/サイズ/行間/太字/疑似ボールド/テーマは全設定が CSS 変数経由のため、各ウィジェットが呼ぶ `applySettings()` だけで即時反映される。日付文字列だけは JS で 1 回設定する:

```js
function updateFontPreview() {
  // fmtDisplay は画面上のエントリ一覧と同じ表示（常に "YYYY/MM/DD HH:MM:SS" スラッシュ形式）
  const stamp = fmtDisplay(nowIso());             // Time utils（L1238）の既存関数
  document.getElementById("font-preview").textContent =
    stamp + " 京にとくあげ給て、物語の多く候ふなる、あるかぎり見せ給へ";
}
```

- `openSettings()`（L2864）に `updateFontPreview();` を追加（パネルを開くたびに現在時刻で更新。日付跨ぎにも自然に対応）。
- **`fmtDisplay` は `dateFormat` 設定（slash/dash/iso）に追従しない**（確認済み・L1238）。その設定はエクスポート（`serializeTxt`）にのみ効き、画面表示は常にスラッシュ形式。プレビューは「エクスポート後の .txt」ではなく「画面上のエントリ一覧の見た目」を再現するのが目的なので、これで正しい。`#setting-date-format` の change ハンドラにフックする必要はない。
- 出力例: `2026/07/02 14:23:05 京にとくあげ給て…`。時分秒までエントリ表示と一致する（本文エントリの time 表示と同一関数）。

### 4.2 フォント参照の解決 — `resolveFontFamily()`

`FONT_FAMILIES[settings.fontKey]`（L2806）の直接参照を置き換える:

```js
function resolveFontFamily(key) {
  if (key && key.startsWith("cf-")) {
    const cf = (settings.customFonts || []).find(f => f.key === key);
    if (cf && cf.family) return `'${cf.family}', ${FONT_FAMILIES["biz-ud"]}`;
    return FONT_FAMILIES["biz-ud"];   // メタ欠落時はフォールバック
  }
  return FONT_FAMILIES[key] || FONT_FAMILIES["biz-ud"];
}
```

- `customFonts` 要素: `{ key: "cf-xxxxxxxx", label: 表示名, source: "file"|"named", family: CSS参照名 }`
  - `source: "named"` — `family` = ユーザー入力のフォント名。保存時に `family.replace(/['"\\;{}<>]/g, "")` でサニタイズ（style 値へのインジェクション防御）。
  - `source: "file"` — `family` = `"edjiki-" + key`（自動生成。インストール済みフォントとの名前衝突を構造的に回避）。`label` = ファイル名から拡張子を除いた文字列。
- `key` は `"cf-" + newId().slice(0, 8)`（既存 `newId()` L1529 節を流用）。

### 4.3 IndexedDB ヘルパー（新セクション `// ========== Custom Fonts ==========`、Settings 節 L2756 の直前に配置）

```js
const FONT_DB_NAME = "edjiki-fonts", FONT_STORE = "fonts";
function openFontDB() {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(FONT_DB_NAME, 1);
    req.onupgradeneeded = () => req.result.createObjectStore(FONT_STORE);
    req.onsuccess = () => resolve(req.result);
    req.onerror   = () => reject(req.error);
  });
}
async function fontDBPut(key, buf)  { /* readwrite put(buf, key) を Promise 化 */ }
async function fontDBGet(key)       { /* readonly get(key) → ArrayBuffer | undefined */ }
async function fontDBDelete(key)    { /* readwrite delete(key) */ }
```

- 値は ArrayBuffer のみ（MIME は FontFace が自動判別するため保存不要）。
- DB は初回 `fontDBPut` まで作られない（＝カスタムフォント未使用ユーザーには IndexedDB が一切出現しない。概要 §5.5）。

### 4.4 起動時の再登録 — `loadCustomFonts()`

初回 `applySettings();`（L2852）の直後に追加:

```js
const _cfMissing = new Set();   // バイナリ欠落キー（in-memory）
async function loadCustomFonts() {
  const files = (settings.customFonts || []).filter(f => f.source === "file");
  if (!files.length) return;
  let loaded = 0;
  for (const f of files) {
    try {
      const buf = await fontDBGet(f.key);
      if (!buf) { _cfMissing.add(f.key); continue; }
      const face = new FontFace(f.family, buf);
      await face.load();
      document.fonts.add(face);
      loaded++;
    } catch { _cfMissing.add(f.key); }
  }
  if (loaded) resizeAll();      // フォント差替でメトリクスが変わるため
  if (_cfMissing.size && settings.fontKey.startsWith("cf-") && _cfMissing.has(settings.fontKey))
    toast("⚠ 取込フォントのデータが見つかりません（設定 > フォントから再取込できます）");
}
loadCustomFonts();   // await しない — 描画をブロックしない（概要 §3.1）
```

- 読込完了までは `resolveFontFamily` のフォールバック（BIZ UDPGothic）で表示され、完了時に差し替わる（`font-display: swap` 相当の体験）。
- 失敗はすべて握りつぶして通常動作を継続（軽量起動の維持）。

### 4.5 設定パネル UI — カスタムグループ（L773 `ui-monospace` の radio の直後）

```html
<div class="font-group">カスタム</div>
<div id="custom-font-list"></div>
<button id="btn-add-font" style="font-size:12px">＋ フォントを追加...</button>
```

```js
function renderCustomFontOptions() {
  const list = document.getElementById("custom-font-list");
  list.innerHTML = "";
  (settings.customFonts || []).forEach(f => {
    const label = document.createElement("label");
    label.className = "font-option";
    const radio = Object.assign(document.createElement("input"),
      { type: "radio", name: "font-family", value: f.key, checked: settings.fontKey === f.key });
    radio.addEventListener("change", () => {          // 静的 radio(L3063) とは別に個別付与
      if (radio.checked) { settings.fontKey = f.key;
        applySettings(); saveSettings(); resizeAll(); updateFontBoldNote(); }
    });
    const name = document.createElement("span");
    name.textContent = f.label;
    if (!_cfMissing.has(f.key)) name.style.fontFamily = `'${f.family}'`;
    label.append(radio, name);
    if (_cfMissing.has(f.key)) {
      const warn = document.createElement("button");   // ⚠ 再取込 → #font-file-input を開き同一 key で上書き
      warn.textContent = "⚠ 再取込";
      warn.addEventListener("click", ev => { ev.preventDefault(); _reimportTarget = f.key;
        document.getElementById("font-file-input").click(); });
      label.appendChild(warn);
    }
    const del = document.createElement("button");
    del.textContent = "✕";
    del.addEventListener("click", ev => { ev.preventDefault(); deleteCustomFont(f.key); });
    label.appendChild(del);
    list.appendChild(label);
  });
  document.getElementById("btn-add-font").disabled = false;  // 上限判定はファイル取込側のみ（§4.6）
}
```

- `openSettings()` に `renderCustomFontOptions();` を追加。radio 選択状態の復元（L2869–2870）は `value="${settings.fontKey}"` の querySelector なので `cf-` キーでもそのまま機能する。
- **注意**: 既存の radio リスナー一括付与（L3063 `document.querySelectorAll`）は起動時の静的 12 書体にしか効かない。動的生成分は上記のとおり生成時に個別付与する。

```js
function deleteCustomFont(key) {
  if (!confirm("このフォントを削除しますか？")) return;
  const f = (settings.customFonts || []).find(x => x.key === key);
  settings.customFonts = (settings.customFonts || []).filter(x => x.key !== key);
  if (f && f.source === "file") fontDBDelete(key).catch(() => {});
  _cfMissing.delete(key);
  if (settings.fontKey === key) settings.fontKey = "biz-ud";
  applySettings(); saveSettings(); resizeAll();
  renderCustomFontOptions();
  const radio = document.querySelector(`input[name="font-family"][value="${settings.fontKey}"]`);
  if (radio) radio.checked = true;
}
```

### 4.6 フォント追加ダイアログ（`#font-add-overlay`）

pw-overlay（L2921 `showPasswordModal`）と同じモーダル構造・cloneNode によるリスナーリセット方式で実装。既定タブは**名前指定**（概要 §5.3）。

```
┌ カスタムフォントの追加 ──────────────────────┐
│ (◉) この端末のフォント名を指定                 │
│     [ 游明朝____________________ ]  ▾datalist  │
│     プレビュー: 京にとくあげ給て、物語の…      │ ← input と同時に反映
│ (○) フォントファイルを取り込む                 │
│     [ファイルを選択] .ttf .otf .woff .woff2    │
│     ※ この端末のブラウザ内に保存されます        │
│     （残り 2 / 3 書体・1ファイル 25MB まで）    │
│                        [キャンセル] [追加]      │
└───────────────────────────────────────────────┘
```

処理仕様:

| 項目 | 仕様 |
|---|---|
| 名前指定・確定 | サニタイズ（§4.2）→ 空なら無効。`{ key, label: 入力値, source: "named", family: 入力値 }` を `settings.customFonts` に push → `fontKey` を新キーに切替 → `applySettings/saveSettings/resizeAll` → `renderCustomFontOptions` → ダイアログ閉 |
| 名前指定・プレビュー | `input` イベントでダイアログ内プレビュー span の `style.fontFamily` を直接更新。フォールバック検知はしない（目視確認） |
| datalist 補完 | 名前入力の初回フォーカス時、`"queryLocalFonts" in window` なら `await queryLocalFonts()` → family 名を `Set` で一意化して `#local-fonts-list` に流し込む。拒否・非対応・例外はすべて無視（datalist が空のまま＝素のテキスト入力）。セッション内 1 回のみ試行 |
| ファイル取込・検証 | `source==="file"` の件数 >= 3 → ラジオ無効＋残数表示（決定事項 1）。`file.size > 25 * 1024 * 1024` → エラーノート表示で中断 |
| ファイル取込・確定 | `arrayBuffer()` → `key` 採番 → `family = "edjiki-" + key` → `new FontFace(family, buf).load()`（**ここで失敗＝不正なフォントファイルとして中断**・トースト「フォントを読み込めませんでした」）→ `document.fonts.add` → `fontDBPut(key, buf)` → customFonts に push → 以降は名前指定と同じ確定処理 |
| 再取込（⚠ 経由） | `_reimportTarget` にキーが入った状態で `#font-file-input` の change を受け、**同一 key・同一 family** で `FontFace` 登録と `fontDBPut` のみ実施（customFonts は変更しない）→ `_cfMissing.delete(key)` → `applySettings/resizeAll/renderCustomFontOptions` |

隠し input（L892 の `#file-input` の隣）:

```html
<input type="file" id="font-file-input" accept=".ttf,.otf,.woff,.woff2" class="hidden">
<input type="file" id="settings-file-input" accept=".json,application/json" class="hidden">
```

---

## 5. F4: 設定エクスポート/インポート

配置: `// ========== Download / Import ==========`（L2134）節の末尾。

### 5.1 UI（設定パネル最下部・「操作」行 L825 の直後）

```html
<div class="settings-row">
  <label class="settings-label">バックアップ</label>
  <div class="security-btns">
    <button id="btn-export-settings">⬇ 設定を書き出す</button>
    <button id="btn-import-settings">⬆ 設定を読み込む...</button>
  </div>
</div>
```

`.security-btns` の既存スタイルを流用（縦積みボタン）。

### 5.2 `exportSettings()`

```js
function exportSettings() {
  const d = new Date();
  const ymd = d.getFullYear() + String(d.getMonth() + 1).padStart(2, "0") + String(d.getDate()).padStart(2, "0");
  const data = {
    app: "edjiki", type: "settings", formatVersion: 1, exportedAt: nowIso(),
    settings: { ...settings },
    drive: {
      fileName:        state.driveFileName,
      folderId:        state.driveFolderId,
      pinnedFileId:    state.drivePinnedFileId,
      archiveFolderId: state.archiveFolderId,
    },
  };
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url; a.download = `edjiki-settings_${ymd}.json`; a.click();
  setTimeout(() => URL.revokeObjectURL(url), 1000);   // download() L2143 と同型
  toast(data.drive.pinnedFileId
    ? "設定を書き出しました（Drive のファイル ID を含みます）"
    : "設定を書き出しました");
  if ((settings.customFonts || []).some(f => f.source === "file"))
    toast("※ 取込フォントのデータ本体は含まれません");   // 直前のトーストを上書きするため文言は要調整（§7 注意 3）
}
```

- エクスポート対象外（概要 §3.4 スコープ表）: `entries` / `cryptoSalt` / `cryptoVerifier` / `accessToken` / `driveFileId` / `driveModifiedTime` / `driveFolderName` / フォントバイナリ / `EDJIKI_CLIENT_ID`。`settings` はスプレッドコピーのため**将来キーが増えても自動追随**する。

### 5.3 `importSettings()` — 検証

`#settings-file-input` の change ハンドラ（`#file-input` L2151–2173 と同型。サイズ上限 1 MB）:

```js
async function applySettingsImport(f) {
  let data;
  try { data = JSON.parse(await f.text()); }
  catch { toast("JSON として読み込めませんでした"); return; }
  if (!data || data.app !== "edjiki" || data.type !== "settings") {
    toast("edjiki の設定ファイルではありません"); return; }
  if (typeof data.formatVersion !== "number" || data.formatVersion > 1) {
    toast("新しいバージョンの設定ファイルです（このバージョンでは読み込めません）"); return; }
  if (!confirm("現在の設定を読み込んだ内容で上書きします。よろしいですか？")) return;
  importSettingsBody(data);
}
```

### 5.4 `importSettingsBody(data)` — settings 反映（ホワイトリスト＋型・値域検証）

```js
const SETTINGS_ENUMS = {
  theme:       THEMES.map(t => t.key),
  fontOutline: ["none", "weak", "medium", "strong"],
  lineEnding:  ["LF", "CRLF"],
  dateFormat:  ["slash", "dash", "iso"],
  isoTzMode:   ["stored", "utc", "fixed"],
};
function sanitizeImportedSettings(src) {
  const out = {};
  for (const k of Object.keys(SETTINGS_DEFAULTS)) {          // 未知キーは黙って捨てる
    if (!(k in src)) continue;
    const v = src[k], def = SETTINGS_DEFAULTS[k];
    if (k === "customFonts") {
      if (Array.isArray(v)) out[k] = v.filter(f =>
        f && typeof f.key === "string" && /^cf-[a-zA-Z0-9-]+$/.test(f.key)
          && typeof f.label === "string" && typeof f.family === "string"
          && (f.source === "file" || f.source === "named")
      ).slice(0, 20).map(f => ({ key: f.key, label: f.label, source: f.source,
                                 family: f.family.replace(/['"\\;{}<>]/g, "") }));
      continue;
    }
    if (typeof v !== typeof def) continue;
    if (SETTINGS_ENUMS[k] && !SETTINGS_ENUMS[k].includes(v)) continue;
    out[k] = v;
  }
  if (out.fontSize   !== undefined) out.fontSize   = Math.min(24,  Math.max(12,  Math.round(out.fontSize)));
  if (out.lineHeight !== undefined) out.lineHeight = Math.min(2.0, Math.max(1.2, out.lineHeight));
  if (out.daysToKeep !== undefined) out.daysToKeep = Math.max(1, Math.round(out.daysToKeep));
  return out;
}
```

`fontKey` の整合: 反映後に `settings.fontKey` が `FONT_FAMILIES` にも `settings.customFonts` の key にも無い場合は `"biz-ud"` に戻す。

### 5.5 `importSettingsBody(data)` — drive 反映（**null 以外のみ上書き**・決定事項 2）

各フィールドの副作用は現行コードの変更セマンティクスに厳密に合わせる:

| フィールド | 条件 | 反映と副作用 | 準拠する現行コード |
|---|---|---|---|
| `fileName` | 非 null かつ現値と異なる | `driveFileName = v; driveFileId = null; driveModifiedTime = null` | config.js 変更時リセット L1105–1109 |
| `folderId` | 同上 | `driveFolderId = v; driveFolderName = null; driveFileId = null; driveModifiedTime = null;` 反映後に `fetchAndCacheFolderName()`（未認証時は L2279 のガードで no-op、次回認証成功時 L2311 で取得される） | `showDriveFolderDialog` OK ハンドラ L3164–3172 |
| `pinnedFileId` | 同上 | `drivePinnedFileId = v; driveFileId = v; driveModifiedTime = null; tokenClient = null;`（スコープが `drive` に広がるため再初期化）＋トースト「Drive のフルアクセス権限が必要です。次回の Drive 操作時に再認証されます」 | ピン適用 L1127–1130・スコープ再初期化 L3286 |
| `archiveFolderId` | 同上 | `archiveFolderId = v; archiveCache = {}`（キャッシュは旧フォルダのファイルを指すため破棄） | `archiveCache` クリアの既存慣行（driveSignOut / _executeArchive） |

### 5.6 反映後の共通処理

```js
Object.assign(settings, sanitizeImportedSettings(data.settings || {}));
/* fontKey 整合 → drive 反映（§5.5） */
saveSettings();
applySettings();
resizeAll();
saveLocal(true);          // state（drive 系）を永続化
updateTitle();            // ヘッダの 📁/ファイル名 表示（L1159）
updateDriveStatusUI();
render();
closeSettings();          // パネル内ウィジェットの再同期は次回 openSettings() に任せる
loadCustomFonts();        // file ソースの customFonts が来た場合のバイナリ照合（無ければ _cfMissing 行き）
toast("設定を読み込みました");
if ((settings.customFonts || []).some(f => f.source === "file" && _cfMissing.has(f.key)))
  toast("※ 取込フォントはファイルの再取込が必要です");
```

- **entries に一切触れない** → `pushUndo()` 不要・`driveDirty` 不変・IME ガード不要。
- ロック中（`state.cryptoVerifier && !cryptoKey`）でも実行可。settings と drive 系フィールドは暗号化対象外であり、`saveLocal` は既存どおり `encryptEntry` 済みエントリをそのまま直列化する。

---

## 6. リリース分割と差分範囲

| リリース | 実装する節 | 触る行域（v0.8.0 基準） |
|---|---|---|
| v0.9.0（F2+F3+プレビュー） | §1.1–1.4, §2, §3, §4.1 | L8 / L155 / CSS 追加 / L720・L774 付近 HTML / L2781 / L2803 / L2864 / L3053 付近 |
| v0.9.1（F4） | §5（`customFonts` はサニタイズのみ先行実装） | L825 付近 HTML / L892 / L2134 節 / L3053 付近 |
| v0.10.0（F1） | §4.2–4.6 ＋ §2.2 の `cf-` 分岐 | L773 付近 HTML / L827 直後ダイアログ / L892 / 新セクション / L2806 |

v0.9.1 時点で `SETTINGS_DEFAULTS.customFonts` と §5.4 のサニタイズを先に入れておくことで、F1 実装後も設定ファイルの形式変更（formatVersion 繰上げ）が不要になる（概要 §7）。

各リリースで CLAUDE.md の Settings shape・Architecture 表・Key invariants を更新する。

## 7. 実装上の注意（ハマりどころ）

1. **既存 radio リスナーは静的要素のみ**（L3063）。カスタムフォントの radio は `renderCustomFontOptions()` 内で個別に付与する（§4.5）。
2. **`applySettings()` は起動直後 L2852 で一度走る**。`resolveFontFamily` が参照する `settings.customFonts` はその時点で読込済み（settings は同期ロード）だが、FontFace の登録は非同期（§4.4）— 未登録の間はフォールバック描画になるのは仕様。
3. **`toast()` は単一要素を使い回す**（L2177–2187、後続呼び出しが前のメッセージを上書き）。§5.2 のように連続トーストは後勝ちになるため、実装時は 1 本のメッセージに統合するか、2 本目を `setTimeout` で遅延させる。
4. **`confirm()`/`alert()` は同期ダイアログ**で IME 合成を壊さないが、インポート/削除は設定パネル操作中のみ発生するため `_imeComposing` ガードは不要。
5. **`FontFace.load()` の失敗**は不正ファイル・破損の主検出点。`fontDBPut` より**先に** load を成功させること（DB にゴミを残さない）。
6. **`state` への新規キー追加はない**（F4 は既存キーの読み書きのみ）。`loadLocal` のバリデーション（L1139）も変更不要。
7. **エクスポート JSON に `driveFolderName` を含めない**こと。含めると移行先で旧名称が表示されたまま `fetchAndCacheFolderName` のガード（`state.driveFolderName` 非 null で return、L2279）により更新されなくなる。
8. **`#font-preview` は `.entry-body` と同じ CSS 変数群を参照**する（§4.1）。`.entry-body` へのスタイル追加時は preview 側への追随を忘れない。

## 8. 手動テストチェックリスト

テストスイートは存在しないため（CLAUDE.md）、`python3 -m http.server 8080` で以下を確認する。

**F2 ボールド**
- [ ] 12 書体それぞれで太字 ON/OFF が本文・全画面・プレビューに反映される
- [ ] さわらび明朝・ひな明朝で太字 ON にすると注記が出る（他書体では出ない）。書体切替でも追従
- [ ] 太字 ON/OFF で textarea の高さが再計算される（長文エントリで折返し行数変化を確認）
- [ ] リロード後も設定が保持される

**F3 疑似ボールド**
- [ ] なし/弱/中/強 × 7 テーマで表示確認（文字色に追従すること）
- [ ] フォントサイズ 12px と 24px で線幅が追従する（em 指定の確認）
- [ ] 選択中もカーソル・IME 変換・選択ハイライトが正常
- [ ] 400 のみ書体＋「強」で疑似ボールドとして実用になるか目視確認（線幅仮値の調整判断）

**プレビュー行**
- [ ] フォント/サイズ/行間/太字/疑似ボールド/テーマの全ウィジェットで即時反映

**F1 カスタムフォント**
- [ ] 名前指定: インストール済みフォント名 → プレビュー即反映・確定後に本文反映
- [ ] 名前指定: 存在しない名前 → フォールバック表示のまま確定できる（エラーにならない）
- [ ] PC Chrome: 名前欄フォーカスで権限プロンプト → 許可で datalist 補完 / 拒否でも入力継続可
- [ ] ファイル取込: .ttf / .otf / .woff2 各 1 つ・25MB 超で拒否・4 書体目で拒否
- [ ] 不正ファイル（.txt を .ttf にリネーム）で「読み込めませんでした」
- [ ] リロード後に取込フォントで表示される（IndexedDB 再登録）・高さ再計算される
- [ ] DevTools で IndexedDB を消してリロード → ⚠ 再取込 → 復旧
- [ ] 使用中フォントを削除 → biz-ud に戻る
- [ ] Android Chrome: ファイル取込・表示・リロード永続化
- [ ] Android Chrome: 名前欄は datalist なしの素の入力として動作

**F4 設定入出力**
- [ ] 書き出し → シークレットウィンドウ（新規プロファイル相当）で読み込み → 全設定・Drive 設定が一致
- [ ] 壊れた JSON / `app` 不一致 / `formatVersion: 99` でそれぞれ中断メッセージ
- [ ] Drive 設定が null の設定ファイルを読み込んでも既存の Drive 接続が壊れない
- [ ] `pinnedFileId` 入りを読み込み → 再認証トースト → 次回 Drive 操作で scope=drive の同意画面
- [ ] `folderId` 入りを読み込み → ヘッダ表示が 📁 になり、認証後にフォルダ名が取得される
- [ ] ロック中に読み込み → 成功し、エントリの暗号化状態が不変
- [ ] 読み込みが Undo スタック・`driveDirty` に影響しない

**回帰**
- [ ] 既存 localStorage（v0.8.0 の settings）でリロード → 新キーが既定値で補完される
- [ ] Drive 保存/読込/自動保存・アーカイブ・検索・パスワード機能に影響なし
- [ ] `file://` 以外の既知制約（CLAUDE.md）に変化なし
