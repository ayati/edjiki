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
| `Ctrl+Z` | Undo (up to 5 steps; no-op when a text field is focused) |
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
| **Undo** | `undoStack` (max 5 snapshots of `state.entries`); `pushUndo()` — deep-clones entries onto the stack, skips if identical to top; `undo()` — pops and restores; triggered on textarea `focus` (first time per element) and `delBtn` click |
| **Time utils** | `nowIso`, `isoFromParts`, `fmtDisplay`, `fmtLocalInput`, `tzSuffix` — all times stored as ISO 8601 with local TZ offset |
| **Normalize** | `normalizeText` strips empty lines and trailing whitespace — entries with empty normalized text are auto-deleted |
| **Txt serialize/parse** | `serializeTxt` / `parseTxt` — text format is `YYYY/MM/DD HH:MM:SS body` (private entries use `-HH:MM:SS`) |
| **Merge** | `mergeEntries(local, incoming)` — deduplicates by `id` first, then by `time+private+normalizedText` signature; winner on `id` collision is the entry with the later `time` |
| **Crypto** | `deriveKey` (PBKDF2/SHA-256, 300k iterations), `encryptText`/`decryptText` (AES-GCM 256bit), `encryptEntry`/`decryptEntry`, `setupPassword`/`removePassword`/`changePassword`/`unlockWithPassword`/`lockEntries` |
| **ID** | `newId()` — wraps `crypto.randomUUID()` with a manual UUID v4 fallback |
| **Render** | Full DOM re-render on every state change; `autoResize` keeps textareas fitted to content; `buildOnboardingCard()` renders first-run guidance when `state.entries` is empty |
| **Actions** | `newEntry` — creates entry, prepends to `state.entries`, sets `driveDirty`, calls `render()`, focuses textarea |
| **Search** | `matchQuery` — partial text match OR date-prefix match against `fmtDisplay` output |
| **Download / Import** | `download` (serializeTxt → Blob), `importFile` (parseTxt + mergeEntries) |
| **Toast** | `toast(msg)` — creates `.toast` div, auto-removes after 3.5 s |
| **Drive** | GSI token client; scope is `drive.file` normally, `drive` when `state.drivePinnedFileId` or `CONFIG_FILE_ID` is set; `driveSave` checks `modifiedTime` before upload and prompts merge on conflict; `driveLoad` merges remote into local; both blocked when locked; `driveSignOut` clears token and resets UI; `driveDirty` flag + `updateDriveDirtyUI()` shows/hides the ☁ header button |
| **UI wire** | Event listeners, keyboard shortcuts, online/offline banner, `beforeunload`/`visibilitychange` flush |
| **Settings** | Color theme (7 presets: 自動/ダーク/ライト/セピア/さくら/ブルー/ノルド), font (12 families), font-size (12–24 px), line-height, `daysToKeep` (default 60), `lineEnding` (LF/CRLF), `dateSep` (`/` or `-`) — stored separately in `localStorage["edjiki.settings"]`, applied via CSS custom properties on `<html>`; opened via ⚙ header button |
| **Drive Settings** | `openDriveSettings` / `closeDriveSettings` — side panel opened from ⋯ menu; `showDriveFolderDialog` / `showDriveFileDialog` / `showArchiveFolderDialog` — parse Google Drive URLs to extract folder/file IDs, update `state.driveFolderId` / `state.drivePinnedFileId` / `state.archiveFolderId` |
| **Archive** | `createZipBlob(files)` — pure-JS ZIP writer (CRC32 + `CompressionStream('deflate-raw')` with STORE fallback, no CDN); `archiveAndTruncate()` → `_executeArchive(stats)` — for each year: downloads existing `jkY.txt` from `state.archiveFolderId`, merges with new cut entries (ensuring no gaps from Jan 1), re-uploads `jkY.txt` + `jkY.zip` to archive folder, then removes successfully archived entries from state; locked entries skip archive but stay in current data; Drive upload uses generic `driveUploadToFolder` / `driveFindInFolder` helpers; clears `archiveCache` on completion |
| **Archive Search** | `listArchiveFiles()` — Drive API list of `jk*.txt` in `state.archiveFolderId`; `searchArchives(query)` — downloads missing files, caches parsed entries in `archiveCache` (`{ fileId: {name, entries[]} }`), applies `matchQuery`; `renderArchiveSection()` — renders trigger button / loading / results below `<main>`; results are read-only cards with an import button; `archiveCache` is cleared on `driveSignOut` and after `_executeArchive`; result list DOM rebuild is skipped when state+count+query are unchanged (`list.dataset.renderKey`) |

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
- `driveDirty` (in-memory boolean) is set to `true` on any edit/add/delete/import; cleared on successful `driveSave` or `driveLoad`. `updateDriveDirtyUI()` shows/hides `#btn-drive-dirty` (the ☁ header button) based on `driveDirty && !!window.EDJIKI_CLIENT_ID`.
- `driveUpload()` omits `name` from metadata when updating an existing file (PATCH) to avoid renaming it. `name` is only sent on initial file creation (POST).
- `state.driveFolderId` is used by `driveFindByName()` and `driveUpload()` as the parent folder. `state.drivePinnedFileId` forces a specific file ID and widens the OAuth scope to `drive` (full access).
- `archiveCache` (in-memory, `{}` on init) caches downloaded archive file contents as parsed entries. Cleared on `driveSignOut()` and after `_executeArchive()`. Never persisted.

## Known constraints

| Constraint | Notes |
|---|---|
| localStorage limit ~5MB | If approaching limit, export to Drive and retire old entries to a file |
| Drive access token lost on tab close | Silent re-auth on next Drive action |
| `drive.file` scope restriction | Cannot access files created by other apps with same name — use `drivePinnedFileId` as workaround |
| `file://` protocol unsupported | OAuth requires HTTP; always use the local server |
| Password loss is unrecoverable | Private entries cannot be restored; Drive saves are plaintext `.txt` |
| Drive operations blocked when locked | Must unlock (enter password) before saving/loading from Drive |
| Undo stack resets on page reload | In-memory only; returns to 0 steps after reload |
| Shift-JIS output not supported | Export (download / Drive save) is always UTF-8; only import auto-detects Shift-JIS |
