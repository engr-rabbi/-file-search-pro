# Installation & Build Guide (Windows)

This guide takes you from a fresh Windows machine to a working `.exe` installer.

## 1. Prerequisites

Install these once:

1. **Node.js 20 LTS or newer** — https://nodejs.org (installs `npm` too)
2. **Git** (optional, only if you're cloning from a repo) — https://git-scm.com
3. **Visual Studio Build Tools** (needed to compile the `better-sqlite3` native module):
   - Run `npm install --global windows-build-tools` **or**
   - Install "Desktop development with C++" from the Visual Studio Installer
   (`better-sqlite3` ships prebuilt binaries for most common Node/Electron versions, so this step is
   often unnecessary — only do it if `npm install` reports a build failure for `better-sqlite3`.)

## 2. Get the project onto your machine

Copy the whole `file-search-pro` folder to your Windows PC (USB drive, zip, git clone — any method).
Open **PowerShell** or **Command Prompt** and `cd` into it:

```powershell
cd C:\path\to\file-search-pro
```

## 3. Install dependencies

```powershell
npm install
```

This installs Electron, better-sqlite3, chokidar, mammoth, xlsx, tesseract.js, fuse.js, pdf-parse,
electron-builder, and everything else listed in `package.json`.

## 4. Rebuild the native SQLite binding for Electron

Electron bundles its own Node.js build, which is usually a different ABI version than your system
Node — native modules like `better-sqlite3` must be rebuilt against it:

```powershell
npm run rebuild
```

## 5. Run it in development mode

```powershell
npm start
```

The app window should open. On first launch, click **Start Full Scan** in the welcome dialog (or
`Ctrl+Shift+R`) to build your search index — this walks every detected drive once. After that,
`Ctrl+R` (incremental rescan) or the **Live file watching** toggle in Settings keeps it up to date
without a full rescan.

> First run tip: a full scan of a large drive (500k+ files) can take a few minutes. Progress, pause,
> resume, and stop controls are in the status bar at the bottom while a scan runs.

## 6. Build the Windows installer (.exe)

```powershell
npm run build:win
```

This produces, inside the generated `dist/` folder:

- `File Search Pro Setup <version>.exe` — a standard NSIS installer (Start Menu shortcut, uninstaller,
  choice of install directory)
- `FileSearchPro-Portable-<version>.exe` — a single portable `.exe`, no installation needed

If you only want an unpacked folder to test without building an installer:

```powershell
npm run pack
```

(output goes to `dist/win-unpacked/`)

## 7. (Optional) Run as Administrator for full-system indexing

Some system folders require elevated permissions to read. Right-click the built `.exe` → **Run as
administrator** to index everything, including protected folders. To make this automatic for every
launch, add this to the `"nsis"` block in `package.json` before rebuilding:

```json
"win": {
  "requestedExecutionLevel": "requireAdministrator"
}
```

## 8. Where your data lives

- **Search index**: `%APPDATA%\file-search-pro\index.sqlite` (plus `-wal`/`-shm` files while running)
- **Logs**: `%APPDATA%\file-search-pro\logs\app-YYYY-MM-DD.log`
- **Settings**: stored in the same SQLite database (`settings` table)

Use **Settings → Backup Index** / **Restore Index** in the app to save or move this file.

## Troubleshooting

| Problem | Fix |
|---|---|
| `npm install` fails compiling `better-sqlite3` | Install Visual Studio Build Tools (C++ workload), then retry. Or upgrade Node.js to a version with prebuilt binaries available. |
| App opens to a blank white window | Run `npm run dev` to open DevTools and check the console for the failing script. |
| "Module did not self-register" / native module errors | Run `npm run rebuild` again — the native binding doesn't match Electron's ABI. |
| Full scan is slow | This is expected on the very first scan of a large drive. Subsequent scans are incremental (only changed files touch the DB) or you can enable live file watching instead. |
| OCR / voice search don't work | Both need an internet connection: Tesseract.js downloads its language model on first use, and voice search uses the Chromium speech service. Search itself works fully offline. |
