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
| **Drive** | GSI token client (`drive.file` scope); `driveSave` checks `modifiedTime` before upload and prompts merge on conflict; `driveLoad` merges remote into local |

### State shape

```js
{
  version: 1,
  updatedAt: "<ISO>",
  driveFileId: "<id>" | null,
  driveFileName: "edjiki.txt",
  driveModifiedTime: "<ISO>" | null,
  entries: [{ id, time, private, text }, ...]  // always time-descending
}
```

Persisted to `localStorage["edjiki.data"]` as JSON.

### Text file format

```
2026/04/13 21:43:10 first line of public entry
continuation lines (no blank lines allowed)
2026/04/13 -20:15:00 private entry (note the dash)
```

Header regex: `^(\d{4})\/(\d{2})\/(\d{2}) (-?)(\d{2}):(\d{2}):(\d{2})(?: (.*))?$`
