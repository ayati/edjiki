# edjiki

自動タイムスタンプ降順テキストエディタ。PC / Android Chrome で動作する単一HTMLアプリ。

- 新規エントリを先頭に挿入し、常に降順で管理
- ローカルは localStorage に自動保存
- `.txt` のダウンロード / インポート
- Google Drive (`edjiki.txt`) への保存・読込（マージ方式）
- public / private フラグ（テキスト出力時は `YYYY/MM/DD -HH:MM:SS` で区別）

## ファイル

| ファイル | 役割 |
|---|---|
| `edjiki.html` | 本体（単一ファイル、ビルド不要） |
| `config.sample.js` | OAuth Client ID 設定のひな形 |
| `config.js` | **自分で作成**（`config.sample.js` をコピーして実値を入れる。`.gitignore` 済み） |
| `LICENSE` | MIT License |

## 使い方

1. ヘッダ右上の **＋** または画面右下の FAB、または `Ctrl+D` で新規エントリを先頭に追加。
2. テキストエリアに本文を入力。`input` で自動保存（localStorage、debounce 300ms）。
3. エントリヘッダの時刻をクリックすると `datetime-local` で秒単位まで編集可能。
4. **🔓 public / 🔒 private** ボタンで切替。private エントリはテキスト出力時のみ `YYYY/MM/DD -HH:MM:SS` で記録。
5. **✕** ボタンで即削除（確認ダイアログなし）。
6. ヘッダの **⋯** メニューからダウンロード・インポート・Drive 保存/読込。
7. **🔍** または `Ctrl+F` で本文または `YYYY/MM/DD` の前方一致検索。

## テキスト形式

```
2026/04/13 21:43:10 公開エントリの本文
複数行の本文も可
2026/04/13 -20:15:00 非公開エントリの本文
```

- 行頭が `YYYY/MM/DD HH:MM:SS ` ならヘッダ、以降は本文。
- ハイフン `-` が秒前にあれば private。
- 空行は禁止（パース時に無視）。

## ショートカット

| キー | 動作 |
|---|---|
| `Ctrl+D` | 新規エントリ |
| `Ctrl+S` | ローカル保存（即時フラッシュ） |
| `Ctrl+Shift+S` | `.txt` ダウンロード |
| `Ctrl+Shift+D` | Drive に保存 |
| `Ctrl+P` | フォーカス中エントリの private 切替 |
| `Ctrl+F` / `Esc` | 検索の開閉 |

## Google Drive 設定手順

1. https://console.cloud.google.com でプロジェクトを作成（既存でも可）。
2. 「APIとサービス」→「ライブラリ」で **Google Drive API** を有効化。
3. 「OAuth 同意画面」を設定（外部、自分のアカウントをテストユーザに追加）。
4. 「認証情報」→「認証情報を作成」→ **OAuth クライアント ID** → アプリの種類「ウェブアプリケーション」。
5. **承認済みの JavaScript 生成元** に以下を追加:
   - `https://www.ayati.com`
   - ローカル動作確認用に `http://localhost:8080` も追加推奨
6. 発行されたクライアント ID を `config.js` に記入:
   ```js
   window.EDJIKI_CLIENT_ID = "1234567890-xxxxx.apps.googleusercontent.com";
   ```
7. スコープは `https://www.googleapis.com/auth/drive.file` のみ使用（このアプリが作成または開いたファイルのみアクセス可能）。

## 配信

### PC ローカル確認

```bash
cd /home/ayati/edjiki
python3 -m http.server 8080
# ブラウザで http://localhost:8080/edjiki.html
```

`file://` で直接開くと OAuth が通らないため、必ず HTTP 経由で開いてください。

### 本番

`edjiki.html` と `config.js` を `https://www.ayati.com/` 配下にアップロード。Android Chrome からそのURLをブックマークして使用。

## データモデル

内部は配列で保持、localStorage には JSON で永続化。

```js
{
  version: 1,
  updatedAt: "2026-04-13T21:50:00+09:00",
  driveFileId: "1AbC..." | null,
  driveFileName: "edjiki.txt",
  driveModifiedTime: "2026-04-13T12:50:00.000Z" | null,
  entries: [
    { id: "uuid", time: "2026-04-13T21:43:10+09:00", private: false, text: "本文\n複数行可" },
    ...
  ]
}
```

- `entries` は常に `time` 降順。
- `text` は blur 時に正規化（空行削除・末尾空白削除）。結果が空なら自動削除。
- テキスト出力/入力は `YYYY/MM/DD (-)HH:MM:SS 本文` 形式（private のみ秒前に `-`）。

## データ保存先

- **PC / Android 共通**: `localStorage["edjiki.data"]`（JSON）
- **Google Drive**: マイドライブ直下に `edjiki.txt`（単一ファイル）
- 複数端末で編集した場合、Drive 保存/読込時に `id` + 内容シグネチャ（`time`+`private`+`text`）でマージし、双方の変更を保持します。
- Drive 保存前に `modifiedTime` を再取得して競合を検知し、ローカルと未マージの変更があれば確認ダイアログを出します。

## 既知の制約

- localStorage 容量は約 5MB。超過しそうなら Drive 保存 + 古いエントリの別ファイル退避を検討。
- Drive アクセストークンはタブを閉じると揮発。次回操作時にサイレント再認証。
- `drive.file` スコープのため、他アプリが作った `edjiki.txt` にはアクセス不可。このアプリで最初に作成した 1 ファイルを使い続けます。
