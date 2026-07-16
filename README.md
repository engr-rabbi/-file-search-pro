# File Search Pro

A professional, **local-only** desktop file search & indexing application for Windows — inspired by
[Everything Search](https://www.voidtools.com/), built with Electron, Node.js, and SQLite, and extended
with content search, filters, a preview panel, duplicate detection, a disk usage analyzer, and
OCR-based image search.

No cloud, no telemetry, no paid APIs — everything runs on your machine against a local SQLite index.

---

## ✨ Features

| Area | What's included |
|---|---|
| **Indexing** | Full multi-drive scan, incremental rescans, background worker threads (one per drive), pause/resume/stop, live progress, live file-watching (chokidar) for instant updates |
| **Search** | Name / partial / wildcard (`*`, `?`) / regular expression / fuzzy (typo-tolerant) / full **content search** inside `.txt .md .pdf .docx .xlsx` |
| **Filters** | Type (file/folder), extension, drive, folder path, size range, modified/created date, hidden files |
| **Results** | Sortable columns, file-type icons, native OS drag-and-drop into Explorer, keyboard navigation |
| **Right-click menu** | Open, Open Containing Folder, Copy Path, Copy Name, Rename, Delete (Recycle Bin, with confirmation), Properties, Add/Remove Favorite |
| **Preview panel** | Images, PDFs (opens in default viewer), text/CSV/JSON/code files, DOCX & XLSX text snippets |
| **UI** | Windows 11 / Fluent-inspired design, light & dark theme, sidebar navigation, animations, full keyboard shortcuts |
| **Performance** | SQLite (WAL mode) with indexed columns + FTS5 for content search, batched bulk inserts, capped fuzzy-search candidate sets |
| **Settings** | Include/exclude folders, auto-startup, auto-reindex interval, live watching, backup/restore index, export results (CSV / Excel / PDF) |
| **Security** | Delete confirmation + Recycle Bin (never a hard delete), system-folder exclusion, per-error logging to disk + DB |
| **Extras** | Recent files, favorites, search history, drag & drop, voice search (Web Speech API), OCR search (Tesseract.js), duplicate file detection (size + SHA‑256 hash), empty folder finder, large file finder, disk usage analyzer |

---

## 🏗 Architecture

```
file-search-pro/
├── package.json              # deps + electron-builder config (Windows NSIS + portable)
├── build/                    # app icon (icon.ico / icon.png)
├── src/
│   ├── main/                 # Electron main process (Node.js)
│   │   ├── main.js           # app bootstrap, window creation
│   │   ├── preload.js        # contextBridge — the ONLY thing the renderer can call into Node with
│   │   ├── menu.js           # native application menu
│   │   ├── autostart.js      # "start with Windows"
│   │   ├── logger.js         # file + DB error logging
│   │   ├── db/
│   │   │   ├── schema.sql    # SQLite schema (files table, FTS5, favorites, settings, ...)
│   │   │   └── database.js   # better-sqlite3 wrapper + prepared statements
│   │   ├── indexer/
│   │   │   ├── driveDetector.js   # finds C:\ D:\ ... (or /Volumes, /mnt on macOS/Linux)
│   │   │   ├── scanWorker.js      # worker_thread: walks one drive, streams batches back
│   │   │   ├── indexer.js         # orchestrates workers, progress, incremental scans, file watching
│   │   │   └── contentExtractor.js# pulls text out of txt/pdf/docx/xlsx for content search
│   │   ├── search/
│   │   │   └── searchEngine.js    # name/wildcard/regex/fuzzy/content query building
│   │   ├── features/
│   │   │   ├── duplicateFinder.js # size-group + SHA-256 hash confirmation
│   │   │   ├── diskAnalyzer.js    # per-drive + per-folder size rollups from the index
│   │   │   ├── emptyAndLarge.js   # empty folder + largest file queries
│   │   │   ├── exporter.js        # CSV / Excel (xlsx) / PDF (via Electron printToPDF) export
│   │   │   ├── backupRestore.js   # copy/restore the SQLite index file
│   │   │   └── ocrSearch.js       # Tesseract.js image OCR → content index
│   │   └── ipc/
│   │       └── registerIpcHandlers.js  # every ipcMain.handle() channel lives here
│   └── renderer/              # the UI (plain HTML/CSS/JS, no framework, no build step)
│       ├── index.html
│       ├── css/                # variables (theme tokens), base, layout, components
│       └── js/                 # state store, icons, search bar, results table, preview,
│                                # context menu, filters flyout, tool views, settings, shortcuts
```

**Why this shape?** The renderer never touches Node or the filesystem directly — it only calls the
typed `window.api.*` functions exposed by `preload.js` over `contextBridge`, and `contextIsolation` is
on. All real work (scanning, SQL, file I/O) happens in the main process, so a compromised or buggy
renderer page can't do anything the API surface doesn't explicitly allow.

---

## 🚀 Getting started (development)

```bash
npm install
npm run rebuild      # rebuilds better-sqlite3's native binding against Electron's Node ABI
npm start             # or: npm run dev   (opens DevTools)
```

On first launch you'll be prompted to run a **Full Scan**. After that, incremental scans (or live
file-watching, toggle in Settings) keep the index fresh — you don't need to rescan everything again.

## 📦 Building the Windows installer

```bash
npm run build:win        # NSIS installer + portable .exe, output in /dist
```

See **INSTALL.md** for the full step-by-step guide (including first-time Windows build machine setup).

---

## ⌨️ Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl+F` | Focus the search box |
| `Ctrl+R` | Incremental rescan |
| `Ctrl+Shift+R` | Full rescan |
| `Ctrl+E` | Export current results |
| `Ctrl+T` | Toggle light/dark theme |
| `Ctrl+P` | Toggle preview panel |
| `Delete` | Delete selected file (Recycle Bin, confirmed) |
| `F2` | Rename selected file |
| `Enter` | Open selected file |
| `Esc` | Close menus/dialogs |

---

## ⚠️ Known limitations & honest notes

- **OCR and voice search need one-time downloads.** Tesseract.js downloads its English language model
  the first time OCR indexing runs, and the Web Speech API used for voice search calls a browser speech
  service — both need an internet connection the first time, even though search itself is 100% local.
- **Video thumbnails** are not generated (would require bundling ffmpeg, a heavy binary dependency);
  videos show a generic icon in the preview panel instead of a frame thumbnail.
- **`better-sqlite3` is a native module.** `npm run rebuild` (via `@electron/rebuild`) must be run after
  `npm install` and again after upgrading Electron, so the compiled binding matches Electron's Node ABI.
- **Administrator Mode**: to index protected system folders, run the built `.exe` "as Administrator" —
  the app itself doesn't self-elevate (Windows will prompt if you check that option in future builds by
  adding `"requestedExecutionLevel": "requireAdministrator"` to the NSIS config).
- This was built and unit-tested (database, search engine, duplicate finder, exporters) in a Linux
  sandbox without access to a real Windows filesystem or an on-screen Electron session — please do a
  pass of manual testing on your own Windows machine before relying on it for anything critical,
  especially the Delete and Rename actions.

---

## 🔒 Privacy

Everything indexed stays in a local SQLite file under your Windows user profile
(`%APPDATA%\file-search-pro\index.sqlite`). No data is sent anywhere; there is no analytics, no ads, and
no paid API keys required.

