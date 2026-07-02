# edjiki 機能拡張 設計書案 — フォント拡張・設定入出力

対象バージョン: v0.9.x（案） / 作成日: 2026-07-02 / 更新日: 2026-07-02 / ステータス: レビュー反映済み（§8 決定事項確定）
詳細設計: [design-font-and-settings-io-detail.md](design-font-and-settings-io-detail.md)

## 1. 目的とスコープ

| # | 機能 | 概要 |
|---|---|---|
| F1 | ローカルフォント取込描画 | ユーザーの手元にあるフォント（インストール済み or フォントファイル）を本文フォントとして使えるようにする |
| F2 | ボールド設定 | 選択中フォントを太字で描画するトグル |
| F3 | 疑似ボールド | 文字の輪郭線描画（ストローク）による疑似的な太らせ（なし・弱・中・強の4段階） |
| F4 | 設定エクスポート/インポート | Drive 連携設定を含む全設定を JSON ファイルとして書き出し・読み込み |

前提条件（全機能共通・非交渉）:

- 単一 HTML ファイル・ビルドなし・CDN 追加なしを維持する。
- 対象ブラウザは PC Chrome / Android Chrome（現行 CLAUDE.md 準拠）。Firefox/Safari は動けば良い（グレースフルデグレード）。
- 日記データ（`edjiki.data`）のスキーマ・テキストファイル形式には一切手を触れない。

---

## 2. 方式調査

### 2.1 F1: ローカルフォントの利用方式（3方式比較）

| 方式 | 仕組み | 対応環境 | 永続性 | 実装コスト |
|---|---|---|---|---|
| A. フォント名直接指定 | `font-family: '游明朝'` のように**名前で参照するだけ**。API 不要 | 全ブラウザ・全OS（そのフォントがインストール済みなら） | 名前文字列のみ（settings に保存） | 極小 |
| B. Local Font Access API | `queryLocalFonts()` でインストール済みフォントを列挙し、選択 UI を出せる | **Chromium 系デスクトップのみ**（Chrome 103+）。Android Chrome・Firefox・Safari 非対応。権限プロンプトあり | 列挙結果は保存不可。選んだ名前を A 方式で保存 | 小 |
| C. フォントファイル取込 | `<input type="file">` で .ttf/.otf/.woff2 を読み、`new FontFace(name, buffer)` → `document.fonts.add()`。バイナリを IndexedDB に保存し起動時に再登録 | 全モダンブラウザ（**Android Chrome 含む**） | IndexedDB（日本語フォントは 3〜20 MB 級のため localStorage は不可） | 中 |

調査メモ:

- A 方式は「インストール済みフォントを名前で使う」だけなら十分に機能する。CSS の `font-family` はローカルフォントを昔から参照できる。弱点はタイポと「入っていないフォント名を指定するとフォールバックに落ちて気づきにくい」こと。
- B 方式は A の入力補助（正確な名前の一覧提示）としてのみ価値がある。Android で使えないため主経路にはできない。
- C 方式だけが「デバイスにインストールされていないフォント」を持ち込める。IndexedDB のバイナリは `FontFace` に直接渡せるため、起動時の再登録は数十 ms 程度。
- **フォントのライセンス注意**: 取込はユーザー自身の端末内で完結し配布を伴わないため、一般的なフォントライセンス上は問題になりにくいが、README に一言注意書きを載せる。

**採用案: A（名前指定）+ C（ファイル取込）の2本立て。B は対応環境でのみ A の入力補助（datalist）として使う。**

### 2.2 F2: ボールドの実現方式

現状の Google Fonts 読み込み URL はウェイト指定なし（= 400 のみ取得）。太字表示には2経路ある:

| 経路 | 内容 | 品質 |
|---|---|---|
| 本物の 700 ウェイト | Google Fonts URL に `:wght@400;700` を追加して取得 | ◎ デザイナー設計の太字 |
| 合成ボールド（font-synthesis） | 400 グリフを機械的に太らせる。ブラウザ既定動作 | △ 日本語ではにじみ気味 |

プリセット12書体のウェイト保有状況（Google Fonts 調査）:

- 700 あり: BIZ UDPGothic, M PLUS 1p, M PLUS Rounded 1c, Noto Sans JP, Noto Serif JP, Zen Old Mincho, しっぽり明朝
- **400 のみ**: さわらび明朝, ひな明朝（→ 合成ボールドに落ちる）
- 汎用 serif/sans-serif/monospace: OS 依存（通常 700 あり）

`unicode-range` サブセット配信のため、700 を URL に足しても**実際に太字を使うまでダウンロードは発生しない**。初期表示コストはほぼ増えない。

**採用案: URL に `:wght@400;700` を追加し、`--entry-font-weight` CSS 変数で切替。400 のみの書体は合成ボールド（既定挙動）に任せ、設定 UI に「※このフォントは太字データを持たないため簡易表示になります」の注記を出す。**

### 2.3 F3: 疑似ボールドの実現方式

| 方式 | 内容 | 評価 |
|---|---|---|
| `-webkit-text-stroke` + `paint-order: stroke fill` | 文字の輪郭線を文字色で描く。`paint-order` で線を塗りの**背面**に回すと字画が潰れない | ◎ 推奨。textarea 内テキストにも効く。`paint-order` の HTML テキスト対応は Chrome 123+（2024-03）で、対象環境では問題なし |
| `text-shadow` 多方向重ね | 4〜8方向に文字色の影をずらして重ねる | ○ 互換性最強だがエッジがわずかに甘い。フォールバック用 |
| SVG 描画 | textarea では使えない | ✕ 除外 |

`paint-order` なしで `-webkit-text-stroke` を使うとストロークが塗りの中心に乗り、細い明朝では字画が痩せて見える。線幅が小さければ許容範囲のため、非対応ブラウザ向けフォールバックは「そのまま細めに描く」で十分（機能検出: `CSS.supports("paint-order", "stroke")`）。

段階設計（フォントサイズに追従させるため em 単位）:

| 段階 | `-webkit-text-stroke-width` | 16px 時の実寸 |
|---|---|---|
| なし | 0 | — |
| 弱 | 0.015em | ≒ 0.24px |
| 中 | 0.03em | ≒ 0.48px |
| 強 | 0.05em | ≒ 0.8px |

補足:

- ストロークは**グリフのメトリクスを変えない**ため `autoResize` への影響なし。一方 F2 の font-weight 変更はメトリクスが変わるので、既存パターン通り設定変更時に `resizeAll()` を呼ぶ。
- F2 と F3 は併用可。F3 は太字ウェイトを持たない書体（さわらび明朝・ひな明朝・取込フォント）で特に効果が大きく、これが本機能の主用途になる見込み。

### 2.4 F4: 設定エクスポート/インポートの方式

既存の `download()`（Blob + a[download]）/ `importFile()`（input[type=file] + FileReader）と同じ土台で JSON を扱うだけ。新規 API 不要。論点は方式より**スコープ**（§3.4）。

---

## 3. 機能設計

### 3.1 F1: ローカルフォント取込

#### UI（設定パネル > フォント欄の末尾に「カスタム」グループを追加）

```
┌ フォント ─────────────────────────┐
│ ゴシック                           │
│  ○ BIZ UDPGothic  … (既存12書体)  │
│ カスタム                           │
│  ○ 游明朝 Demibold        [✕]     │ ← 取込済み/名前指定済みフォント
│  [＋ フォントを追加...]            │
└───────────────────────────────────┘
```

「＋ フォントを追加...」を押すと小ダイアログ:

```
┌ カスタムフォントの追加 ─────────────────┐
│ ● フォントファイルを取り込む             │
│    [ファイルを選択] .ttf .otf .woff .woff2 │
│    ※ この端末のブラウザ内に保存されます   │
│ ● この端末のフォント名を指定             │
│    [ 游明朝________________ ▾ ]          │ ← 対応環境では datalist 補完
│                        [キャンセル][追加] │
└─────────────────────────────────────────┘
```

- 名前指定欄は入力と同時にプレビュー行へ即時反映（フォールバックしていないかを目で確認できる）。
- `queryLocalFonts()` が使える環境（PC Chrome）では、名前欄フォーカス時に一度だけ権限を求めて datalist に候補を流し込む。非対応環境では素のテキスト入力。
- 上限: ファイル取込は **3 書体まで**・1 ファイル 25 MB まで（IndexedDB 逼迫と起動時間の抑制）。名前指定は無制限（文字列のみのため）。

#### データ設計

```js
// settings に追加
customFonts: [
  { key: "cf-<uuid先頭8桁>", label: "游明朝 Demibold",
    source: "file" | "named",
    family: "游明朝 Demibold" }   // named: CSS参照名 / file: FontFace登録名
]
// fontKey は既存キーに加えて "cf-…" を取り得る
```

- フォントバイナリは IndexedDB `edjiki-fonts` DB / `fonts` ストア（key = `cf-…`, value = ArrayBuffer + MIME）。settings/localStorage には**バイナリを置かない**。
- 起動時: `settings.customFonts` の `source==="file"` を IndexedDB から読み `FontFace` 登録。**登録完了を待たずに描画開始**（`font-display: swap` 相当の挙動 — 一瞬フォールバックで出て差し替わる）。軽量起動を損なわない。
- バイナリ欠落時（別端末に設定だけ持ち込んだ場合など）: フォント一覧の該当項目に ⚠ を出し「ファイルを再取込」導線を表示。描画はフォールバックで継続。
- 削除 [✕]: settings から除去 + IndexedDB から削除。使用中フォントを消したら `fontKey` を既定 `biz-ud` に戻す。

#### 制約・注意

- IndexedDB はブラウザのサイトデータ消去で消える（localStorage と同運命）。日記データと違い Drive バックアップはしない — 再取込で足りるため。
- ロック/パスワードとは無関係（フォントは秘匿情報ではない）。

### 3.2 F2: ボールド設定

- `settings.fontBold: false`（既定）を追加。
- 設定パネルのフォント欄の直下にチェックボックス「太字で表示」。
- `applySettings()` で `--entry-font-weight: 700 | 400` を設定し、`.entry textarea` / `.fullscreen-entry textarea` の `font-weight` に反映。変更時 `resizeAll()`。
- Google Fonts の `<link>` URL を `family=BIZ+UDPGothic:wght@400;700&…`（700 保有書体のみ）へ更新。
- 400 のみの書体・取込フォント選択時はチェックボックス脇に注記「※簡易的な太字になります」を表示（判定は書体キーの静的テーブルで十分。取込フォントは一律注記）。

### 3.3 F3: 疑似ボールド

- `settings.fontOutline: "none" | "weak" | "medium" | "strong"`（既定 `"none"`）を追加（内部キー名は実装機構＝ストロークに合わせて `fontOutline` のまま。UI 表記はすべて「疑似ボールド」）。
- UI はセグメント選択（テーマ選択と同様のボタン列）: `なし / 弱 / 中 / 強`。ラベルは「疑似ボールド」。
- `applySettings()` で `--entry-text-stroke: 0 | 0.015em | 0.03em | 0.05em` を設定。CSS:

```css
.entry textarea, .fullscreen-entry textarea {
  -webkit-text-stroke: var(--entry-text-stroke, 0) currentColor;
  paint-order: stroke fill;
}
```

- `currentColor` なのでテーマ切替（7 プリセット）に自動追従する。追加のテーマ別調整は不要。
- メトリクス不変のため `resizeAll()` 不要（呼んでも害はないので F2 と共通ハンドラで可）。

### 3.4 F4: 設定エクスポート/インポート

#### スコープ（何を含め、何を含めないか）

| 区分 | 項目 | 判断 |
|---|---|---|
| ✅ 含む | `edjiki.settings` 全キー（theme, fontSize, lineHeight, fontKey, daysToKeep, lineEnding, dateFormat, isoTz*, autoFullscreenOnNew, driveAutoSave, fontBold, fontOutline, customFonts） | 本体 |
| ✅ 含む | `state` の Drive 設定 4 項目: `driveFileName`, `driveFolderId`, `drivePinnedFileId`, `archiveFolderId` | 「Drive 連携含む」の要件。ID は URL から抽出済みの文字列でありトークンではない |
| ❌ 含まない | `entries`（日記本文） | 別機能（txt/Drive）の責務 |
| ❌ 含まない | `cryptoSalt` / `cryptoVerifier` / `cryptoKey` | 暗号材料は entries と不可分。設定ファイルに混ぜると危険かつ無意味 |
| ❌ 含まない | `accessToken` / `driveFileId` / `driveModifiedTime` | トークンは秘匿・非永続。fileId/modifiedTime は端末ローカルの同期状態であり持ち出すと衝突検知が壊れる |
| ❌ 含まない | カスタムフォントの**バイナリ** | 数 MB〜のため既定で除外。`customFonts` メタのみ移し、移行先で ⚠ 再取込導線（§3.1）に乗せる |
| ❌ 含まない | `EDJIKI_CLIENT_ID`（config.js） | デプロイ設定。gitignore された config.js の責務 |

#### ファイル形式

```json
{
  "app": "edjiki",
  "type": "settings",
  "formatVersion": 1,
  "exportedAt": "2026-07-02T21:00:00+09:00",
  "settings": { "...": "edjiki.settings 全体" },
  "drive": {
    "fileName": "edjiki.txt",
    "folderId": "1AbC...",
    "pinnedFileId": null,
    "archiveFolderId": "1XyZ..."
  }
}
```

ファイル名: `edjiki-settings_YYYYMMDD.json`

#### UI と挙動

- 設定パネル最下部に新設 settings-row「バックアップ」: `[⬇ 設定を書き出す]` `[⬆ 設定を読み込む...]` の 2 ボタン。
- **エクスポート**: 即ダウンロード。`drivePinnedFileId` を含む場合のみ「Drive のファイル ID が含まれます」とトースト（軽い注意喚起。ID は秘密鍵ではないため確認ダイアログまでは出さない）。
- **インポート**:
  1. `app === "edjiki" && type === "settings"` を検証。不一致は「edjiki の設定ファイルではありません」でトースト中断。`formatVersion` が新しすぎる場合も中断。
  2. `confirm("現在の設定を読み込んだ内容で上書きします。よろしいですか？")` — 既存の importFile と同じ温度感のシンプルな確認 1 回。差分プレビューは軽量方針に反するため作らない。
  3. `settings` は既知キーのみ採用（`SETTINGS_DEFAULTS` にあるキー + `customFonts` のホワイトリスト方式。未知キーは黙って捨てる → 将来版からの後方互換も自然に成立）。
  4. `drive.*` は **null でない値のみ** `state` に反映（設定ファイル側が空でも既存接続を壊さない）。`drivePinnedFileId` が新規に入る場合はスコープが `drive` に広がるため、次回 Drive 操作時に再認証が走る旨をトースト。
  5. `applySettings()` → `saveSettings()` → `saveLocal()` → `render()` → トースト「設定を読み込みました」。
  6. `customFonts` に `source:"file"` があれば「取込フォントはファイルの再取込が必要です」をトーストし、フォント欄に ⚠ 表示。
- インポートは**エントリに一切触れない**ため Undo 対象外（pushUndo 不要）。ロック中でも実行可（entries 非依存）。ただし drive.* の反映だけはロック中も安全。

---

## 4. スキーマ変更まとめ

```js
// settings（edjiki.settings）— 追加キーのみ
{
  fontBold: false,          // F2
  fontOutline: "none",      // F3: "none"|"weak"|"medium"|"strong"
  customFonts: []           // F1: [{ key, label, source, family }]
}
// state（edjiki.data）— 変更なし（F4 は既存キーを読み書きするだけ）
// 新規ストレージ: IndexedDB "edjiki-fonts" / store "fonts"（F1 ファイル取込時のみ作成）
```

既存データへのマイグレーション: 不要（全て追加キー + `SETTINGS_DEFAULTS` スプレッドで自然に補完される）。

---

## 5. UX 提案（軽量ツールとしての使い勝手を守るために）

1. **設定パネルにライブプレビュー行を 1 行追加**（フォント欄の直上）。サンプル文は「**〈今日の日付〉 京にとくあげ給て、物語の多く候ふなる、あるかぎり見せ給へ**」（例: `2026/07/02 京にとくあげ給て…`）。日付部は**実際の当日日付を動的に表示**し、本文エントリ（自動タイムスタンプ）の見た目を忠実に再現する。仮名・漢字の混ざった本文で、フォント・サイズ・行間・太字・疑似ボールドが即時反映される。3 機能が絡むため、本文で試行錯誤させるよりダイアログ内で完結させる方が速い。実装は div 1 つ ＋ 日付を差し込む数行の JS。
2. **400 のみ書体への誘導**。太字チェック時に太字データを持たない書体なら「このフォントには疑似ボールドがおすすめです」の一行を出す。
3. **フォント追加の既定タブは「ファイル取込」ではなく「名前指定」**。インストール済みフォントを使いたいだけのケースが大半で、名前入力ならストレージも消費しない。ファイル取込は第二選択。
4. **設定の書き出し先はローカルファイルのみに留める**（Drive への設定保存機能は作らない）。JSON ファイルは Drive に手動で置けば端末間移動でき、専用同期を作ると Drive 節の状態機械（driveDirty/conflict）が肥大する。コスト対効果が合わない。
5. **起動パスに同期処理を足さない**。カスタムフォントの IndexedDB 読込は非同期・非ブロッキングで行い、失敗してもフォールバックフォントで普通に使える設計とする（§3.1）。
6. **やらないことリスト**: フォントごとのウェイト数値指定（100–900 スライダー）、疑似ボールドのストローク色カスタム、エクスポートへのフォントバイナリ同梱、設定の自動クラウド同期 — いずれも設定 UI と状態管理を太らせる割に日記ツールの本務に寄与しない。

## 6. 互換性・制約一覧

| 事項 | 内容 |
|---|---|
| `paint-order`（HTML テキスト） | Chrome/Edge 123+（2024-03）。非対応ではストロークが字画中心に乗るが線幅が細く実害小。`CSS.supports` で検出可 |
| `queryLocalFonts()` | Chromium デスクトップのみ・要権限。**Android Chrome 不可** → datalist 補完が出ないだけで機能自体は成立 |
| `FontFace` + IndexedDB | 全対象環境で利用可。Android Chrome 含む |
| IndexedDB の揮発性 | サイトデータ消去で消える。消えても再取込のみで復旧（日記データ非依存） |
| Google Fonts URL 変更 | `:wght@400;700` 追加。unicode-range サブセットにより未使用ウェイトはダウンロードされない |
| インポートの脅威モデル | JSON は自作ファイル前提だがホワイトリスト検証を通す（innerHTML に流さない・未知キー破棄・型チェック） |

## 7. 実装ステップ案（リリース分割）

| リリース | 内容 | 規模感 |
|---|---|---|
| v0.9.0 | F2 ボールド + F3 疑似ボールド（CSS 変数 2 本・設定 UI・プレビュー行） | 小。単独で完結し F1 の注記系の土台になる |
| v0.9.1 | F4 設定エクスポート/インポート | 小〜中。download/importFile の既存パターン流用 |
| v0.10.0 | F1 カスタムフォント（名前指定 → ファイル取込 → datalist 補完の順に段階実装） | 中〜大。IndexedDB・追加ダイアログ・起動時登録 |

F4 を F1 より先にする理由: F1 の `customFonts` スキーマが決まっていれば F4 は最初から素通しできる（本書 §3.4 で織込済）。逆順だと F4 改修が二度手間になる。

## 8. 決定事項（2026-07-02 レビューで確定）

1. ファイル取込フォントの上限: **3 書体 / 1 ファイル 25 MB** で確定。
2. F4 インポート時の Drive 設定反映: **null 以外のみ上書き**で確定（null を正として接続解除を移送するユースケースは不要と判断）。
3. 疑似ボールド 4 段階の線幅（0.015/0.03/0.05em）: **プロトタイプ検証で妥当性確認済み**（Chromium 145 / textarea + 実 mincho で「強」でも字画が潰れず均一に太る）。詳細は detail §1.4.1。
4. プリセット書体の 700 ウェイト: **最初から Google Fonts URL に含める**（サブセット配信のため実コスト小）。
5. 表記: 機能名・UI 文言とも「縁取り」を廃し**「疑似ボールド」に統一**（内部キー `fontOutline` は据え置き）。
6. ライブプレビュー行のサンプル文: **「〈今日の日付〉 京にとくあげ給て、物語の多く候ふなる、あるかぎり見せ給へ」**で確定。日付部はプレースホルダ（yyyy/mm/dd）ではなく**当日日付を動的表示**（例: `2026/07/02`）。
