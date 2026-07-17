# Plan: Fix Missing PWA Icons (Al-Sajdah Tracker Legend)

## Context / Problem
- The AI model could not read `icon-192x192.png` / `icon-512x512.png` because this model does not support image (PNG) input. That is a model capability limit, NOT a file corruption issue.
- Investigation revealed a real bug: the referenced icon files **do not exist** in the project. There is no `icons/` directory and no `*.png` files anywhere in the repo.
- Three icon paths are referenced but missing:
  - `./icons/icon-192x192.png`
  - `./icons/icon-512x512.png`
  - `./icons/icon-maskable-512x512.png`
- Referenced from:
  - `manifest.json` lines 13–31 (all three icons)
  - `index.html` line 13 (`rel="icon"` 192) and line 14 (`apple-touch-icon` 192)
  - `sw.js` lines 6–8 (`ASSETS_TO_CACHE`)

### Impact
- `sw.js` uses `cache.addAll(ASSETS_TO_CACHE)` on install (line 17). `addAll` rejects the whole batch if ANY asset returns non-OK (404). So the missing icons cause the service worker install to fail → no offline caching.
- Without valid manifest icons, the PWA is not installable (`beforeinstallprompt` may not fire; install button flow at `index.html` lines 741–780 never triggers).

## Goal
Make the PWA install and cache correctly by (1) providing the three icon files at the exact referenced paths, and (2) hardening `sw.js` so a single missing/unavailable cached asset never again breaks service worker installation.

## Decisions (confirmed with user)
- **Icon production:** User will supply the three PNG files themselves. The implementation agent does NOT generate binary PNGs (a text-editing agent cannot author PNG content). Implementation is limited to: creating the `icons/` folder placeholder (via user-provided files), and editing text files (`sw.js`, optionally cache version).
- **SW hardening:** Yes — change `sw.js` so `install` does not fail when an individual asset is missing.
- **Filenames/paths:** Keep exactly as currently referenced (no renames), so the user-provided files must match the required specs below.

## Required icon files (user provides these exact files)
Place all three in a new `icons/` directory at the project root:

| File | Path | Size (px) | Format | `purpose` | Notes |
|------|------|-----------|--------|-----------|-------|
| icon-192x192.png | `./icons/icon-192x192.png` | 192×192 | PNG | any | Also used as favicon + apple-touch-icon |
| icon-512x512.png | `./icons/icon-512x512.png` | 512×512 | PNG | any | Standard install icon |
| icon-maskable-512x512.png | `./icons/icon-maskable-512x512.png` | 512×512 | PNG | maskable | Keep important content within the central ~80% "safe zone" (add padding) so Android maskable cropping looks correct |

Design guidance (optional, to match app theme): background `#0a0e14`, gold accent `#d4af37` (theme colors from `manifest.json` lines 7–8).

Validation for provided files:
- Exact pixel dimensions as above.
- Valid PNG (opens in an image viewer).
- Maskable icon has adequate padding (test in a maskable preview tool such as maskable.app).

## Implementation Tasks (ordered)
> Note: Task 1 is a user action (supply binaries). Tasks 2–3 are text edits an implementation agent can perform.

1. **Add icon files** — Create directory `icons/` at project root and place the three user-provided PNGs there with the exact names/sizes in the table above.

2. **Harden `sw.js` install** so one missing asset does not abort caching. Replace the `install` handler's `cache.addAll(ASSETS_TO_CACHE)` (currently `sw.js` lines 14–18) with a resilient per-asset cache that tolerates individual failures. Suggested approach:
   - Open the cache, then `Promise.allSettled` over `ASSETS_TO_CACHE`, calling `cache.add(url)` for each and catching/logging any rejection instead of failing the whole install.
   - Keep `self.skipWaiting()` behavior.
   - Example intent (implementation agent to finalize):
     ```js
     event.waitUntil(
       caches.open(CACHE_NAME).then((cache) =>
         Promise.allSettled(
           ASSETS_TO_CACHE.map((url) =>
             cache.add(url).catch((err) => {
               console.warn('[SW] Skipped caching:', url, err);
             })
           )
         )
       )
     );
     ```

3. **Bump the service worker cache version** in `sw.js` line 1 (`CACHE_NAME = 'al-sajdah-v1.0.0'` → e.g. `'al-sajdah-v1.0.1'`) so returning users get the new SW/activate cleanup that removes the old (possibly empty/broken) cache. The existing `activate` handler (`sw.js` lines 24–39) already deletes non-matching caches.

## Out of Scope
- No changes to `manifest.json` or `index.html` icon references (existing paths are correct once files are added). Only revisit if the user decides to rename/relocate icons.
- No script-based/canvas icon generation (user opted to provide PNGs).
- No switch to SVG icons.
- No functional/app-logic changes beyond service worker resilience.

## Risks / Notes
- If a provided PNG has wrong dimensions, some browsers still accept it but installability/quality may degrade — verify sizes.
- After changes, the old service worker may be cached in the browser; a hard reload or closing all tabs may be needed to pick up the new `CACHE_NAME`.
- Hardening `sw.js` means a truly missing asset now installs silently without that asset cached; rely on the warning logs to catch regressions.

## Validation Plan
1. Confirm the three files exist: `ls ./icons` shows the three PNGs with correct sizes.
2. Serve the project over HTTP(S) (service workers require a secure/localhost origin; opening via `file://` will not register the SW).
3. In browser DevTools:
   - **Application → Manifest:** all three icons load with no errors; app shows as installable.
   - **Application → Service Workers:** SW is `activated and running`, no install errors in console.
   - **Application → Cache Storage:** new cache (e.g. `al-sajdah-v1.0.1`) contains `index.html`, `manifest.json`, and the icon files.
   - **Console:** no 404s for the icon paths; no `SW registration failed`.
4. Trigger install: confirm the "📲 Pasang Aplikasi" button appears (`beforeinstallprompt`) and install completes.
5. Offline test: with SW active, go offline and reload — app still loads.

## Handoff
Implementation requires (a) the user to drop in binary PNG files and (b) text edits to `sw.js`. Switch to an implementation-capable agent to perform the `sw.js` edits; the icon binaries must be added by the user (or an agent/tool able to write binary image files).
