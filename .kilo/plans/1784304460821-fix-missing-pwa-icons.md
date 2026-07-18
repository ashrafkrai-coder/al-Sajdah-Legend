# Plan: Fix PWA Icons for Install (Al-Sajdah Legend)

## Current State (verified 2026-07-18 ~08:29)
Project root: `C:\Users\SMKTAMANMEDANPERANTI\Documents\New folder\al-Sajdah-Legend`

Root contents: `.git`, `.kilo`, `data.json`, `index.html`, `manifest.json`, `sw.js`, **`icon-192x192.png`**, **`icon-512x512.png`**.

- The two icon PNGs now live at the **project ROOT** (`./icon-192x192.png` = 79,875 B; `./icon-512x512.png` = 434,398 B). Sizes match the earlier verified generated set → valid 192×192 and 512×512 PNGs.
- There is **NO `icons/` subfolder** and **no** `icon-maskable-512x512.png` anywhere.

### THE BUG: path mismatch → PWA cannot install
All code references the non-existent `./icons/` folder, so every icon URL 404s:
- `manifest.json` (lines 13–31): `./icons/icon-192x192.png`, `./icons/icon-512x512.png`, `./icons/icon-maskable-512x512.png` — all wrong; maskable file also does not exist.
- `index.html` (lines 13–14): `rel="icon"` and `apple-touch-icon` → `./icons/icon-192x192.png` — wrong path.
- `sw.js` `ASSETS_TO_CACHE` (lines 6–8): three `./icons/...` entries — wrong paths (silently skipped by the hardened installer, but leaves console warnings and nothing cached).

Because no manifest icon resolves, `beforeinstallprompt` will not fire and the app is **not installable**.

### Already correct (do NOT change)
- `sw.js` line 1: `CACHE_NAME = 'al-sajdah-v1.0.1'`.
- `sw.js` install handler already uses `Promise.allSettled` + `cache.add(url).catch(...)` — robust to missing assets. Keep as-is.

### Note (not in scope this task)
- `manifest.json` `name` is now `"Al-Sajdah Legend"` while `index.html` `<title>` is `"Al-Sajdah Tracker Legend"`. User chose to LEAVE names as-is.

## Decisions (confirmed with user)
1. **Fix by repointing code to the ROOT icon paths** — do NOT move files into an `icons/` folder.
2. **Use only the two existing root icons** (192, 512), both `"purpose": "any maskable"`. **Remove** the non-existent `icon-maskable-512x512.png` entry entirely.
3. **Leave app names** as they currently are (no title/name alignment).

## Implementation Tasks (ordered)
> Requires an implementation-capable agent (text edits only; no file moves/copies needed). This plan-mode agent cannot edit source.

### 1. `manifest.json` — replace the `icons` array (lines 13–32)
Point to root paths and use two icons with `any maskable`:
```json
"icons": [
  {
    "src": "./icon-192x192.png",
    "sizes": "192x192",
    "type": "image/png",
    "purpose": "any maskable"
  },
  {
    "src": "./icon-512x512.png",
    "sizes": "512x512",
    "type": "image/png",
    "purpose": "any maskable"
  }
]
```
This removes the non-existent maskable entry and the wrong `./icons/` prefix.

### 2. `index.html` — fix icon links (lines 13–14)
Change both `./icons/icon-192x192.png` references to root path:
```html
<link rel="icon" type="image/png" sizes="192x192" href="./icon-192x192.png">
<link rel="apple-touch-icon" href="./icon-192x192.png">
```
(Optional enhancement: add `<link rel="icon" ... sizes="512x512" href="./icon-512x512.png">`; not required for install.)

### 3. `sw.js` — fix `ASSETS_TO_CACHE` (lines 6–8)
Replace the three `./icons/...` lines with the two real root files and drop the maskable entry:
```js
const ASSETS_TO_CACHE = [
  './',
  './index.html',
  './manifest.json',
  './icon-192x192.png',
  './icon-512x512.png'
];
```
- Do NOT change `CACHE_NAME` or the `Promise.allSettled` install logic (already correct).

## Out of Scope
- Moving icons into an `icons/` subfolder.
- Adding extra icon sizes (16/32/72/…/384) or a favicon — the two icons satisfy install requirements (192 + 512). Can be revisited later using `pwa_icons/` if broader coverage is wanted.
- Aligning `name`/`title` strings.
- Any app-logic/feature changes.

## Risks / Notes
- **Maskable safe zone:** the 192/512 icons are now declared `any maskable`. If Android crops edges, a dedicated padded maskable icon may be needed later (test in maskable.app). Not blocking install.
- **Stale service worker/cache:** returning users may hold an old SW; the `activate` handler deletes non-matching caches, but a hard reload / closing all tabs may be needed. `CACHE_NAME` stays `v1.0.1` (no functional cache-key change needed since only asset URLs corrected); if testers see stale 404s cached, bump to `v1.0.2` to force refresh — optional.
- **Consistency:** after edits, confirm NO remaining `./icons/` references exist in `manifest.json`, `index.html`, or `sw.js` (grep for `icons/`).
- **Duplicate project folder** exists at `C:\Users\...\Documents\al-Sajdah Legend` (no "New folder"); it is NOT the target. Ignore it for this task.

## Validation Plan
1. Grep the three files for `icons/` → expect zero matches after edits.
2. Confirm `./icon-192x192.png` and `./icon-512x512.png` exist at root (already verified present).
3. Serve over HTTP/localhost (service workers need a secure/localhost origin; `file://` will NOT register the SW).
4. DevTools:
   - **Application → Manifest:** both icons load, no errors; "installability" shows app is installable.
   - **Network/Console:** no 404s for icon URLs; no `SW registration failed`; no `[SW] Skipped caching` warnings for icons.
   - **Application → Service Workers:** status "activated and running".
   - **Application → Cache Storage:** `al-sajdah-v1.0.1` contains `index.html`, `manifest.json`, `icon-192x192.png`, `icon-512x512.png`.
5. Trigger install: "📲 Pasang Aplikasi" button appears (`beforeinstallprompt`) and install completes; installed icon renders correctly.
6. Offline test: with SW active, go offline and reload — app still loads.

## Handoff
Switch to an implementation-capable agent to make the three text edits (Tasks 1–3). No file copying/moving is required. This plan-mode agent cannot edit source files.
