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

## Config

Copy `config.sample.js` to `config.js` and insert your OAuth 2.0 Client ID. `config.js` is gitignored. Without it, all features work except Google Drive sync.

## Architecture (all in `edjiki.html`)

The script is organized into named sections separated by `// ========== Section ==========` comments:

| Section | Responsibility |
|---|---|
| **State** | `state` object + `loadLocal` / `saveLocal` (debounced 300 ms, immediate on flush) |
| **Time utils** | `nowIso`, `isoFromParts`, `fmtDisplay`, `fmtLocalInput`, `tzSuffix` — all times stored as ISO 8601 with local TZ offset |
| **Normalize** | `normalizeText` strips empty lines and trailing whitespace — entries with empty normalized text are auto-deleted |
| **Txt serialize/parse** | `serializeTxt` / `parseTxt` — text format is `YYYY/MM/DD HH:MM:SS body` (private entries use `-HH:MM:SS`) |
| **Merge** | `mergeEntries(local, incoming)` — deduplicates by `id` first, then by `time+private+normalizedText` signature; winner on `id` collision is the entry with the later `time` |
| **Render** | Full DOM re-render on every state change; `autoResize` keeps textareas fitted to content |
| **Actions** | `newEntry` — creates entry, prepends to `state.entries`, calls `render()`, focuses textarea |
| **Search** | `matchQuery` — partial text match OR date-prefix match against `fmtDisplay` output |
| **Download / Import** | `download` (serializeTxt → Blob), `importFile` (parseTxt + mergeEntries) |
| **Toast** | `toast(msg)` — creates `.toast` div, auto-removes after 3.5 s |
| **Drive** | GSI token client (`drive.file` scope, or `drive` when `EDJIKI_DRIVE_FILE_ID` is set); `driveSave` checks `modifiedTime` before upload and prompts merge on conflict; `driveLoad` merges remote into local; both are blocked when entries are locked |
| **UI wire** | Event listeners, keyboard shortcuts, online/offline banner, `beforeunload`/`visibilitychange` flush |
| **Settings** | Font, font-size, line-height — stored separately in `localStorage["edjiki.settings"]`, applied via CSS custom properties on `<body>` |
| **Crypto** | `deriveKey` (PBKDF2/SHA-256, 300k iterations), `encryptText`/`decryptText` (AES-GCM 256bit), `encryptEntry`/`decryptEntry`, `setupPassword`/`removePassword`/`changePassword`/`unlockWithPassword`/`lockEntries` |

### State shape

```js
{
  version: 1 | 2,              // 2 = password configured
  updatedAt: "<ISO>",
  driveFileId: "<id>" | null,
  driveFileName: "edjiki.txt",
  driveModifiedTime: "<ISO>" | null,
  cryptoSalt: "<Base64>",      // present when version === 2
  cryptoVerifier: { iv, ct },  // AES-GCM encrypted "edjiki-verified", for password check
  entries: [{ id, time, private, text }, ...]  // always time-descending
}
```

Persisted to `localStorage["edjiki.data"]` as JSON.

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
- Settings (`edjiki.settings`) are independent of diary data — resetting `localStorage["edjiki.data"]` does not affect font/size preferences.
- The Google Drive token (`accessToken`) is in-memory only; it is never persisted to `localStorage`. `pendingAction` queues one Drive operation while awaiting token acquisition.
- `cryptoKey` (a non-extractable `CryptoKey`) is in-memory only — never stored anywhere. Lost on page close.
- Locked entries (`text === null`) are skipped by `serializeTxt`, `matchQuery`, and `Ctrl+P`. Drive operations require the app to be unlocked.
- On startup: entries with `enc` field but no `text` field get `text: null` injected before first render. If `state.cryptoVerifier` exists, `showPasswordModal("unlock")` is called after `render()`.
