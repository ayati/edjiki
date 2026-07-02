# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

edjiki is a single-file, no-build HTML app: an auto-timestamped diary editor that stores entries in descending order. Runs on PC and Android Chrome. All logic lives in `edjiki.html`.

## Running locally

```bash
cd /home/ayati/edjiki
python3 -m http.server 8080
# Open http://localhost:8080/edjiki.html
```

Opening via `file://` breaks Google OAuth — always use HTTP.

There is no build step, no test suite, and no linter. The only "development command" is the HTTP server above.

## Navigating the single-file codebase

All logic is in `edjiki.html`. Jump between sections with:

```bash
grep -n "// ==========" edjiki.html
```

## Config

Copy `config.sample.js` to `config.js` and insert your OAuth 2.0 Client ID. `config.js` is gitignored. Without it, all features work except Google Drive sync.

Optional `config.js` variables:
```js
window.EDJIKI_CLIENT_ID       = "…";           // required for Drive
window.EDJIKI_DRIVE_FILENAME  = "memo.txt";    // default: "edjiki.txt"
window.EDJIKI_DRIVE_FOLDER_ID = "<folder-id>"; // initial save folder (overridable via UI)
window.EDJIKI_DRIVE_FILE_ID   = "<file-id>";   // pin a pre-existing file; expands OAuth scope to `drive`
```

Note: `EDJIKI_DRIVE_FOLDER_ID` and `EDJIKI_DRIVE_FILE_ID` can also be set at runtime via the Drive settings panel (⋯ → Drive 設定). Runtime values are stored in `state` and take precedence over config.js on subsequent loads.

## Keyboard shortcuts (useful for manual testing)

| Key | Action |
|---|---|
| `Ctrl+D` / `N` / `M` | New entry |
| `Ctrl+Z` | Undo (up to 20 steps / 5 MiB; no-op when a text field is focused) |
| `Ctrl+S` | Flush save to localStorage |
| `Ctrl+Shift+S` | Download `.txt` |
| `Ctrl+Shift+D` | Save to Drive |
| `Ctrl+P` | Toggle public/private on focused entry |
| `Ctrl+Enter` | Enter / exit fullscreen for focused entry |
| `Ctrl+F` | Open search bar |
| `Esc` | Exit fullscreen / close search bar |

## Production deploy

`edjiki.html` and `config.js` are served from `https://www.ayati.com/`. The OAuth client ID in Google Cloud Console has that origin (plus `http://localhost:8080`) registered as an authorized JavaScript origin.

## Architecture (all in `edjiki.html`)

The script is organized into named sections separated by `// ========== Section ==========` comments:

| Section | Responsibility |
|---|---|
| **State** | `state` object + `loadLocal` / `saveLocal` (debounced 300 ms, immediate on flush) |
| **Undo** | `undoStack` (max `UNDO_MAX_STEPS=20` snapshots of `state.entries`, max `UNDO_MAX_BYTES=5 MiB` total — each item is `{ snap, bytes, _cmp }` where `_cmp` is the JSON used for dedup); `undoBytes` tracks cumulative size; `pushUndo()` — deep-clones entries, skips if `_cmp` matches the top, evicts from the head when either limit is exceeded (the most recent entry is always kept); `undo()` — pops and restores, decrements `undoBytes`; `clearUndo()` — empties the stack and resets counters, called from `lockEntries` / `removePassword` / `changePassword` / `driveSignOut`. Hybrid trigger inside the textarea `input` handler: pushes when `_undoCharsSinceLastPush >= 200` or `Date.now() - _undoLastPushAt >= 30000`, gated by `!_imeComposing`. Structural triggers: textarea first focus, `✕` delete, fullscreen empty-entry deletion, `editTime` confirm, `importFile`, archive search import, `driveSave` conflict merge, `driveLoad` merge, `_executeArchive`. |
| **Time utils** | `nowIso`, `isoFromParts`, `fmtDisplay`, `fmtLocalInput`, `tzSuffix` — all times stored as ISO 8601 with local TZ offset |
| **Normalize** | `normalizeText` strips empty lines and trailing whitespace — entries with empty normalized text are auto-deleted |
| **Txt serialize/parse** | `serializeTxt` / `parseTxt` — text format is `YYYY/MM/DD HH:MM:SS body` (private entries use `-HH:MM:SS`) |
| **Merge** | `mergeEntries(local, incoming)` — deduplicates by `id` first, then by `time+private+normalizedText` signature; winner on `id` collision is the entry with the later `time` |
| **Crypto** | `deriveKey` (PBKDF2/SHA-256, 300k iterations), `encryptText`/`decryptText` (AES-GCM 256bit), `encryptEntry`/`decryptEntry`, `setupPassword`/`removePassword`/`changePassword`/`unlockWithPassword`/`lockEntries` |
| **ID** | `newId()` — wraps `crypto.randomUUID()` with a manual UUID v4 fallback |
| **Custom Fonts** | `settings.customFonts` (`[{ key:"cf-"+uuid8, label, source:"file"\|"named", family }]`) holds metadata only; binaries live in IndexedDB `edjiki-fonts`/`fonts` (`fontDBPut`/`fontDBGet`/`fontDBDelete`, keyed by `cf-…`). `resolveFontFamily(key)` maps a `cf-` key to `'family', <biz-ud fallback>` — used by `applySettings` in place of the raw `FONT_FAMILIES` lookup. `named` fonts reference an installed font by name (sanitized, best-effort — no missing-detection possible); `file` fonts use `family = "edjiki-"+key` (collision-proof) and are re-registered by `loadCustomFonts()` (non-blocking; `FontFace(...).load()` then `document.fonts.add`, `resizeAll()` on success; `_cfLoaded` skips already-loaded keys and prunes stale keys). `loadCustomFonts()` runs **on startup and at the end of `importSettingsBody`** — the latter is essential: a settings JSON restores `customFonts` metadata but not binaries, so importing on a cleared device must detect the gap. Missing binaries go into in-memory `_cfMissing` → `renderCustomFontOptions()` renders that row with a `.cf-missing` (danger-colored) treatment, a 「（データなし）」 label, and a ⚠ 再取込 button (`_reimportTarget` + `#font-file-input`); a toast fires when the *selected* font is missing. Add dialog (`#font-add-overlay`) has a name field (`queryLocalFonts()` datalist when available — Chromium desktop only) and a file picker (≤3 file fonts, ≤25 MiB, `FontFace.load()` is the validity gate before `fontDBPut`). |
| **Render** | Full DOM re-render on every state change; `autoResize` keeps textareas fitted to content; `buildOnboardingCard()` renders first-run guidance when `state.entries` is empty (only when `EDJIKI_CLIENT_ID` is set; otherwise a one-line hint). Its footer has two `onboarding-alt` rows: 「Drive を使わずにすぐ始める」→ `newEntry`, and 「保存済みの設定から復元」→ opens `#settings-file-input` (settings-backup import) so a returning user can restore Drive target IDs, then re-render shows the folder step as ✓ and proceed to Drive load; `safeRender()` defers `render()` while `_imeComposing` is true (set via document-level `compositionstart`/`compositionend` listeners) so async paths like `driveLoad` and `driveSave` conflict-merge don't destroy IME composition state |
| **Actions** | `newEntry` — creates entry, prepends to `state.entries`, sets `driveDirty`, calls `render()`, focuses textarea; `newEntryFullscreen` — same but calls `enterFullscreen(e)` instead of render+focus; `updateFabUI()` — syncs FAB icon/title (`＋` or `✏`) to `settings.autoFullscreenOnNew` |
| **Search** | `matchQuery` — partial text match OR date-prefix match against `fmtDisplay` output |
| **Archive Search** | `listArchiveFiles()` — Drive API list of `jk*.txt` in `state.archiveFolderId`; `searchArchives(query)` — downloads missing files, caches parsed entries in `archiveCache` (`{ fileId: {name, entries[]} }`), applies `matchQuery`; `renderArchiveSection()` — renders trigger button / loading / results below `<main>`; results are read-only cards with an import button; `archiveCache` is cleared on `driveSignOut` and after `_executeArchive`; result list DOM rebuild is skipped when state+count+query are unchanged (`list.dataset.renderKey`) |
| **Download / Import** | `download` (serializeTxt → Blob), `importFile` (parseTxt + mergeEntries); `exportSettings`/`applySettingsImport`/`importSettingsBody` — settings backup as `edjiki-settings_YYYYMMDD.json` (`{ app:"edjiki", type:"settings", formatVersion:1, settings, drive:{ fileName, folderId, pinnedFileId, archiveFolderId } }`). Import validates app/type/formatVersion, `confirm()`s, then `sanitizeImportedSettings` (whitelist by `SETTINGS_DEFAULTS` keys + enum/range checks, unknown keys dropped) and applies drive fields **only when non-null**. Never touches `entries`/crypto → no `pushUndo`, works while locked. Setting a new `drivePinnedFileId` resets `tokenClient` (scope widens to `drive`); changing `archiveFolderId` clears `archiveCache`. |
| **Toast** | `toast(msg)` — creates `.toast` div, auto-removes after 3.5 s |
| **Drive** | GSI token client; scope is `drive.file` normally, `drive` when `state.drivePinnedFileId` or `CONFIG_FILE_ID` is set; `driveSave({ silent, keepalive })` checks `modifiedTime` before upload and prompts merge on conflict; `driveLoad({ initial })` merges remote into local — when `initial: true` skips download if `meta.modifiedTime === state.driveModifiedTime` and uses 「同期」-flavored toasts; both blocked when locked; `driveSignOut` clears token and resets UI; `driveDirty` flag + `updateDriveStatusUI()` controls `#drive-status` header element (states: hidden / ☁ pending / 保存中... / ✓ 保存済 / ✕ 保存失敗); `☁ Drive 接続` tap dispatches by state — error → `driveSave()` (retry); `!accessToken && driveDirty` → `driveSave()` (auth+save); `!accessToken && !driveDirty` → `driveLoad({ initial: true })` (auth+sync); auto-save uses two debounce timers — `scheduleDriveAutoSave()` (10 s idle) and `scheduleCharCountSave()` (2 s debounce, fired every 100 chars typed via `driveCharsSinceLastSave`); also triggered by `visibilitychange` (both hidden and visible), `online` event, and `exitFullscreen()`; all paths fire `driveSave({ silent: true })` only when `accessToken` is set. `keepalive: true` is only set on the `visibilitychange → hidden` path because fetch's keepalive caps the body at 64 KiB. |
| **UI wire** | Event listeners, keyboard shortcuts, online/offline banner, `beforeunload`/`visibilitychange` flush (localStorage) + Drive auto-save |
| **Settings** | Color theme (7 presets: 自動/ダーク/ライト/セピア/さくら/ブルー/ノルド), font (12 presets + user custom fonts — see **Custom Fonts** row), font-size (12–24 px), line-height, `fontBold` (bool, default false — sets `--entry-font-weight` 700/400; Google Fonts URL carries `wght@400;700` for the 7 weight-capable presets, さわらび/ひな明朝 fall back to synthetic bold and show a note), `fontOutline` (`none`/`weak`/`medium`/`strong`, default `none` — pseudo-bold via `--entry-text-stroke` `0`/`0.015em`/`0.03em`/`0.05em` with `-webkit-text-stroke: … currentColor; paint-order: stroke fill` on `.entry-body`; UI label 疑似ボールド), `daysToKeep` (default 60), `lineEnding` (LF/CRLF), `dateSep` (`/` or `-`), `autoFullscreenOnNew` (bool, default false), `driveAutoSave` (bool, default true) — stored separately in `localStorage["edjiki.settings"]`, applied via CSS custom properties on `<html>`; opened via ⚙ header button. A live `#font-preview` row (populated by `updateFontPreview()` via `fmtDisplay(nowIso())`) mirrors font/size/line-height/bold/outline through the same CSS vars. |
| **Drive Settings** | `openDriveSettings` / `closeDriveSettings` — side panel opened from ⋯ menu; `showDriveFolderDialog` / `showDriveFileDialog` / `showArchiveFolderDialog` — parse Google Drive URLs to extract folder/file IDs, update `state.driveFolderId` / `state.drivePinnedFileId` / `state.archiveFolderId` |
| **Archive** | `createZipBlob(files)` — pure-JS ZIP writer (CRC32 + `CompressionStream('deflate-raw')` with STORE fallback, no CDN); `archiveAndTruncate()` → `_executeArchive(stats)` — for each year: downloads existing `jkY.txt` from `state.archiveFolderId`, merges with new cut entries (ensuring no gaps from Jan 1), re-uploads `jkY.txt` + `jkY.zip` to archive folder, then removes successfully archived entries from state; locked entries skip archive but stay in current data; Drive upload uses generic `driveUploadToFolder` / `driveFindInFolder` helpers; clears `archiveCache` on completion |

### State shape

```js
{
  version: 1 | 2,               // 2 = password configured
  updatedAt: "<ISO>",
  driveFileId: "<id>" | null,   // Drive file ID of the current save file
  driveFileName: "edjiki.txt",
  driveFolderId: "<id>" | null, // save folder (null = My Drive root)
  drivePinnedFileId: "<id>" | null, // explicit file ID set by user or config; expands scope to `drive`
  driveFolderName: "<name>" | null, // fetched from Drive API; displayed in header
  driveModifiedTime: "<ISO>" | null,
  cryptoSalt: "<Base64>",       // present when version === 2
  cryptoVerifier: { iv, ct },   // AES-GCM encrypted "edjiki-verified", for password check
  entries: [{ id, time, private, text }, ...]  // always time-descending
}
```

Persisted to `localStorage["edjiki.data"]` as JSON.

`state.archiveFolderId` — Drive folder ID for archive files (`jkYYYY.txt` / `.zip`). Separate from `driveFolderId`. Not in the original state shape — added by a migration guard (`if (state.archiveFolderId === undefined)`) on load.

**Entry shapes (in-memory vs. localStorage):**

| Situation | In-memory | localStorage |
|---|---|---|
| Public entry | `{ id, time, private:false, text }` | same |
| Private, no password | `{ id, time, private:true, text }` | same |
| Private, unlocked | `{ id, time, private:true, text }` | `{ id, time, private:true, enc:{iv,ct} }` |
| Private, locked | `{ id, time, private:true, text:null, enc:{iv,ct} }` | `{ id, time, private:true, enc:{iv,ct} }` |

`saveLocal` is async — it calls `encryptEntry` on every entry before serialization. Callers that need to await completion use `await saveLocal(true)`.

### Text file format

```
2026/04/13 21:43:10 first line of public entry
continuation lines (no blank lines allowed)
2026/04/13 -20:15:00 private entry (note the dash)
```

Header regex: `^(\d{4})\/(\d{2})\/(\d{2}) (-?)(\d{2}):(\d{2}):(\d{2})(?: (.*))?$`

### Key invariants

- `state.entries` is always kept time-descending; `sortEntries()` is called by `render()`.
- `normalizeText` is called on `blur` of each textarea; an entry whose normalized text is empty is deleted from `state.entries` before the next save.
- Drive conflict detection compares `meta.modifiedTime` (Drive) with `state.driveModifiedTime` (last known); a mismatch triggers a merge-and-upload confirmation dialog.
- Settings (`edjiki.settings`) are independent of diary data — resetting `localStorage["edjiki.data"]` does not affect font/size/`daysToKeep` preferences.
- The Google Drive token (`accessToken`) is in-memory only; it is never persisted to `localStorage`. `pendingAction` queues one Drive operation while awaiting token acquisition.
- `cryptoKey` (a non-extractable `CryptoKey`) is in-memory only — never stored anywhere. Lost on page close.
- Locked entries (`text === null`) are skipped by `serializeTxt`, `matchQuery`, and `Ctrl+P`. Drive operations require the app to be unlocked.
- Decryption failure sets `corrupted: true` on the entry (in-memory only, never persisted); render shows a distinct "復号失敗" message instead of "ロック中".
- `private` toggle (button and Ctrl+P) is disabled when the entry is public and no password is set (`!state.cryptoVerifier`). Already-private entries can still be toggled back to public without a password.
- `download()` shows a warning toast if any private entry with visible text (`text !== null`) is present — `.txt` export is always plaintext regardless of password.
- On startup: entries with `enc` field but no `text` field get `text: null` injected before first render. If `state.cryptoVerifier` exists, `showPasswordModal("unlock")` is called after `render()`.
- `driveDirty` (in-memory boolean) is set to `true` on any edit/add/delete/import; cleared on successful `driveSave` or `driveLoad`. `updateDriveStatusUI()` controls the `#drive-status` `<button>` in the header: hidden (idle+clean), ☁ `ds-pending` (dirty+no token — tap to authenticate), `保存中...` `ds-saving`, `✓ 保存済` `ds-saved` (auto-hides after 3 s), `✕ 保存失敗` `ds-error` (tap to retry). `driveAutoSaveStatus` tracks these states; `driveAutoSaveTimer` and `_driveStatusFadeTimer` are the associated timers.
- Auto-save triggers: (1) `scheduleDriveAutoSave()` — 10 s idle debounce, reset on every `driveDirty = true`; (2) `scheduleCharCountSave()` — 2 s debounce, fired once `driveCharsSinceLastSave >= 100` (counter resets on each save); (3) `visibilitychange` hidden *and* visible; (4) `online` event when reconnecting; (5) `exitFullscreen()` entry. All paths guard on `accessToken && settings.driveAutoSave && navigator.onLine && !locked && status !== "saving"`. Never trigger OAuth automatically — require user gesture for first auth.
- `driveSave({ silent, keepalive })` — `silent: true` suppresses the success toast and the failure toast (the header status pill is the only signal); `keepalive: true` is passed through to `driveGetMeta`/`driveUpload` and is **only** safe to set on the `visibilitychange → hidden` path. fetch's `keepalive` caps the request body at 64 KiB, so enabling it for foreground saves causes large files to fail silently.
- `settings.autoFullscreenOnNew` — when true, FAB shows `✏` and calls `newEntryFullscreen()` (new entry + immediate fullscreen); header `＋新規` always calls `newEntry()` regardless of setting.
- `driveUpload()` omits `name` from metadata when updating an existing file (PATCH) to avoid renaming it. `name` is only sent on initial file creation (POST).
- `state.driveFolderId` is used by `driveFindByName()` and `driveUpload()` as the parent folder. `state.drivePinnedFileId` forces a specific file ID and widens the OAuth scope to `drive` (full access).
- `archiveCache` (in-memory, `{}` on init) caches downloaded archive file contents as parsed entries. Cleared on `driveSignOut()` and after `_executeArchive()`. Never persisted.

### Settings shape

Persisted to `localStorage["edjiki.settings"]` as JSON. Independent of diary data.

```js
{
  theme: "auto",        // "auto" | "dark" | "light" | "sepia" | "sakura" | "blue" | "nord"
  fontSize: 16,         // 12–24 px
  lineHeight: 1.6,      // 1.2–2.0
  fontKey: "biz-ud",    // font preset key
  fontBold: false,      // → --entry-font-weight 700/400
  fontOutline: "none",  // "none" | "weak" | "medium" | "strong" → --entry-text-stroke (疑似ボールド)
  customFonts: [],      // [{ key:"cf-xxxxxxxx", label, source:"file"|"named", family }] — バイナリは IndexedDB
  daysToKeep: 60,       // archive cutoff threshold
  lineEnding: "LF",     // "LF" | "CRLF"
  dateSep: "/",         // "/" | "-"
  autoFullscreenOnNew: false,
  driveAutoSave: true
}
```

## Known constraints

| Constraint | Notes |
|---|---|
| localStorage limit ~5MB | If approaching limit, export to Drive and retire old entries to a file |
| Drive access token lost on tab close | Silent re-auth on next Drive action |
| `drive.file` scope restriction | Cannot access files created by other apps with same name — use `drivePinnedFileId` as workaround |
| `file://` protocol unsupported | OAuth requires HTTP; always use the local server |
| Password loss is unrecoverable | Private entries cannot be restored; Drive saves are plaintext `.txt` |
| Drive operations blocked when locked | Must unlock (enter password) before saving/loading from Drive |
| Undo stack resets on page reload | In-memory only; returns to 0 steps after reload. Also cleared on lock / password change-or-remove / Drive sign-out. |
| Shift-JIS output not supported | Export (download / Drive save) is always UTF-8; only import auto-detects Shift-JIS |
| Settings export excludes secrets/state | `entries`, crypto material, `accessToken`, `driveFileId`/`driveModifiedTime`, `driveFolderName`, and custom-font binaries are intentionally omitted; a settings JSON only moves preferences + Drive target IDs |
| Custom-font binaries are device-local | Stored in IndexedDB (`edjiki-fonts`), not localStorage/Drive. Cleared by site-data reset → the entry survives in `settings.customFonts` but goes to `_cfMissing`; recover via ⚠ 再取込. Not backed up (settings export carries metadata only). `named` fonts are best-effort (silently fall back if the font isn't installed on the device) |
| `queryLocalFonts()` desktop-Chromium-only | Local Font Access API (name autocomplete datalist) is unavailable on Android Chrome/Firefox/Safari and needs permission; degrades to a plain text field. File import (`FontFace`+IndexedDB) works everywhere including Android |
