# Plan: Fix Missing PWA Icons (Al-Sajdah Tracker Legend)

## Context / Problem
- The PWA references icon files that do not exist inside the project. There is no `icons/` directory in the project root (`C:\Users\SMKTAMANMEDANPERANTI\Documents\New folder\al-Sajdah-Legend`) — verified 2026-07-18: root contains only `.git`, `.kilo`, `data.json`, `index.html`, `manifest.json`, `sw.js`. No `favicon.ico` at root either.
- The user has a generated icon set in an **external** folder: `C:\Users\SMKTAMANMEDANPERANTI\Documents\al-Sajdah Legend\pwa_icons` (NOTE: different path — no "New folder", space in "al-Sajdah Legend").

### IMPORTANT: duplicate project + icon placement
- There are **two copies** of this project:
  1. **This workspace (the REAL/target project):** `C:\Users\SMKTAMANMEDANPERANTI\Documents\New folder\al-Sajdah-Legend` — confirmed by the user as the real one.
  2. A duplicate: `C:\Users\SMKTAMANMEDANPERANTI\Documents\al-Sajdah Legend` (contains `pwa_icons/` and loose `icon-192x192.png` / `icon-512x512.png` at its root; its `manifest.json` is byte-identical to this workspace's).
- The user reported "icons already added", but they were dropped as **loose root files in the DUPLICATE folder**, NOT in this workspace and NOT in any `icons/` subfolder. Therefore the icons still must be copied INTO this workspace's `icons/` subfolder (Task 1 is still required).
- The `manifest.json`/`sw.js`/`index.html` reference `./icons/...` paths, so loose root icons would not satisfy them regardless of folder.

### Icon files available in `pwa_icons/` (verified by directory listing)
`favicon.ico`, and PNGs: `16, 32, 48, 57, 60, 72, 76, 96, 114, 120, 128, 144, 152, 167, 180, 192, 256, 384, 512`, plus `manifest-icons-snippet.json`.
The snippet declares 8 icons (72, 96, 128, 144, 152, 192, 384, 512) with `192` and `512` marked `"purpose": "any maskable"`. **There is NO separate `icon-maskable-512x512.png` file.**

### Current references in the project (verified)
- `manifest.json` lines 13–31: `icon-192x192.png` (any), `icon-512x512.png` (any), `icon-maskable-512x512.png` (maskable — does NOT exist).
- `index.html` line 13: favicon → `./icons/icon-192x192.png`; line 14: `apple-touch-icon` → `./icons/icon-192x192.png`.
- `sw.js` `ASSETS_TO_CACHE` (lines 2–9): `'./icons/icon-192x192.png'`, `'./icons/icon-512x512.png'`, `'./icons/icon-maskable-512x512.png'` (the last does NOT exist).

### Already done (verified in current sw.js — DO NOT repeat)
- `CACHE_NAME` is already `'al-sajdah-v1.0.1'` (line 1).
- `install` already uses `Promise.allSettled` + `cache.add(url).catch(...)` (lines 17–23), so a missing asset no longer aborts install. Hardening is complete.
- Remaining defect: `ASSETS_TO_CACHE` still lists `./icons/icon-maskable-512x512.png` (line 8) which will not exist → silently skipped with a warning. Must be removed/aligned.

### Impact if left unaddressed
- Without valid manifest icons at correct paths, the PWA is not installable (`beforeinstallprompt` won't fire; install button in `index.html` lines 741–780 never triggers).
- The dangling maskable path in `sw.js` produces a console warning on every install.

## Goal
Make the PWA installable and clean by copying the user's generated icons into the project and aligning `manifest.json`, `index.html`, and `sw.js` so every referenced icon path exists.

## Decisions (confirmed with user)
1. **Icon source:** Use the user's existing generated PNGs in `pwa_icons/` (do not generate new images).
2. **Maskable handling:** Use snippet approach — no separate maskable file; mark `192x192` and `512x512` with `"purpose": "any maskable"`. Remove the `icon-maskable-512x512.png` reference everywhere.
3. **Icon set to copy:** snippet set (72, 96, 128, 144, 152, 192, 384, 512) **+** `icon-16x16.png`, `icon-32x32.png`, `icon-180x180.png`, and `favicon.ico` at project root.
4. **Favicon/apple-touch-icon:** keep `favicon.ico` at project **root** (`./favicon.ico`, standard discovery); add 16/32 favicons + 180 apple-touch-icon links in `index.html`.

## Implementation Tasks (ordered)
> Require an implementation-capable agent: binary file copies (shell) + text edits. This plan-mode agent cannot copy files or edit source.

### 1. Create `icons/` folder and copy files
Create `icons/` at project root and copy from the external source. Source dir must be quoted (contains a space):
`C:\Users\SMKTAMANMEDANPERANTI\Documents\al-Sajdah Legend\pwa_icons`

Copy these into `.\icons\`:
`icon-72x72.png, icon-96x96.png, icon-128x128.png, icon-144x144.png, icon-152x152.png, icon-192x192.png, icon-384x384.png, icon-512x512.png, icon-16x16.png, icon-32x32.png, icon-180x180.png`

Copy `favicon.ico` to the **project root**: `.\favicon.ico`

Suggested PowerShell:
```powershell
New-Item -ItemType Directory -Path ".\icons" -Force
$src = "C:\Users\SMKTAMANMEDANPERANTI\Documents\al-Sajdah Legend\pwa_icons"
$pngs = @("icon-72x72.png","icon-96x96.png","icon-128x128.png","icon-144x144.png",
          "icon-152x152.png","icon-192x192.png","icon-384x384.png","icon-512x512.png",
          "icon-16x16.png","icon-32x32.png","icon-180x180.png")
foreach ($f in $pngs) { Copy-Item -LiteralPath (Join-Path $src $f) -Destination ".\icons\" }
Copy-Item -LiteralPath (Join-Path $src "favicon.ico") -Destination ".\favicon.ico"
```

### 2. Update `manifest.json` icons array
Replace the 3-entry `icons` array (lines 13–32) with the full snippet set, `./icons/`-prefixed, `any maskable` on 192 & 512 (removes `icon-maskable-512x512.png`):
```json
"icons": [
  { "src": "./icons/icon-72x72.png",   "sizes": "72x72",   "type": "image/png" },
  { "src": "./icons/icon-96x96.png",   "sizes": "96x96",   "type": "image/png" },
  { "src": "./icons/icon-128x128.png", "sizes": "128x128", "type": "image/png" },
  { "src": "./icons/icon-144x144.png", "sizes": "144x144", "type": "image/png" },
  { "src": "./icons/icon-152x152.png", "sizes": "152x152", "type": "image/png" },
  { "src": "./icons/icon-192x192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
  { "src": "./icons/icon-384x384.png", "sizes": "384x384", "type": "image/png" },
  { "src": "./icons/icon-512x512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
]
```

### 3. Update `index.html` head icon links (lines 12–14)
Replace the two existing links with a consistent set referencing only copied files:
```html
<link rel="icon" href="./favicon.ico" sizes="any">
<link rel="icon" type="image/png" sizes="32x32" href="./icons/icon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="./icons/icon-16x16.png">
<link rel="apple-touch-icon" sizes="180x180" href="./icons/icon-180x180.png">
<link rel="apple-touch-icon" href="./icons/icon-192x192.png">
```
(All referenced files — `favicon.ico`, 32, 16, 180, 192 — are produced in Task 1.)

### 4. Clean up `sw.js` cached asset list
- Remove the dangling line `'./icons/icon-maskable-512x512.png'` from `ASSETS_TO_CACHE` (line 8). Keep `'./'`, `'./index.html'`, `'./manifest.json'`, `'./icons/icon-192x192.png'`, `'./icons/icon-512x512.png'`.
- Optional: also cache the other copied sizes (72/96/128/144/152/384) for fuller offline coverage — only if desired; not required for install.
- **Do NOT** change `CACHE_NAME` (already `v1.0.1`) or the `Promise.allSettled` install logic (already correct).

## Out of Scope
- Generating new icon images (reusing existing ones).
- Copying the extra iOS/legacy sizes (48, 57, 60, 76, 114, 120, 167, 256) — not required for install; revisit only if broader legacy iOS coverage is desired.
- Any app-logic or feature changes.

## Risks / Notes
- **Path correctness:** source path has a space and differs from project path; use `-LiteralPath`/quotes. Verify `pwa_icons/` contents before copy (already listed; `icon-180x180.png` confirmed present).
- **Maskable design:** now 192/512 serve as maskable. Confirm adequate safe-zone padding (test in a maskable preview, e.g. maskable.app); if edges crop on Android, a dedicated padded maskable icon may be needed later.
- **Silent skips:** hardened `sw.js` installs even if an asset is missing; rely on `console.warn` logs to catch regressions. After Task 4 there should be zero warnings for icon paths.
- **Stale SW:** returning users may hold the old SW; the activate handler deletes old caches, but a hard reload / closing tabs may be needed to pick up new `v1.0.1`.
- **No dangling references:** every path in `manifest.json`, `index.html`, and `sw.js` must map to a file copied in Task 1.

## Validation Plan
1. List `.\icons` and confirm all 11 PNGs present; confirm `.\favicon.ico` at root.
2. Serve over HTTP/localhost (service workers require a secure/localhost origin; `file://` will NOT register the SW).
3. DevTools:
   - **Application → Manifest:** all 8 listed icons load with no errors; app is installable; maskable preview correct.
   - **Application → Service Workers:** status "activated and running"; no `SW registration failed`.
   - **Application → Cache Storage:** `al-sajdah-v1.0.1` cache contains `index.html`, `manifest.json`, and the icon entries; **no** `console.warn` about skipped icon paths.
4. Trigger install: "📲 Pasang Aplikasi" button appears (`beforeinstallprompt`) and install completes; installed app shows the correct icon.
5. Offline test: with SW active, go offline and reload — app still loads.

## Handoff
Switch to an implementation-capable agent to: (a) create `icons/` and copy the binary files from `pwa_icons/`, (b) edit `manifest.json`, `index.html`, and `sw.js` per Tasks 2–4. This plan-mode agent cannot copy files or edit source.
