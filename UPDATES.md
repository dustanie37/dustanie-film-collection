# UPDATES — Dustanie Film Collection

Prioritized backlog of requested and identified improvements.  
Format: **Priority · Complexity · Est. tokens**

---

## ✅ Completed (2026-05-15) — Cross-device sync fix + Settings cleanup

- **Root cause fixed** — `init()` was calling `loadSeed()` → `persist()` → `pushToCloud()` before `onCloudInit()` resolved, causing a race condition that overwrote Supabase with 42 placeholder movies on any new device
- **Cloud-first init** — `init()` is now `async` and `await`s `onCloudInit()` before touching localStorage or seed data
- **Count-based arbitration** — `onCloudInit()` compares local vs cloud counts: local wins if it has more (silent auto-restore), cloud wins otherwise (normal sync). Eliminates the need for manual push/pull entirely
- **`loadSeed()` isolated** — no longer calls `persist()`, writes localStorage only, never pushes placeholder data to Supabase
- **Settings stripped down** — removed Cloud Sync section (Pull from Cloud, Force Push), Local Backup buttons, Export CSV, and Danger Zone. Settings now contains only OMDb API Key and Export/Import JSON
- **Import auto-syncs** — `importJSON()` now calls `persist()` so imported data pushes to cloud immediately

---

## ✅ Completed (2026-05-06) — Firebase → Supabase Migration

- **Removed Firebase** — stripped 3 SDK scripts, all Firebase auth/database code, Google Sign-In flow
- **Supabase cloud sync** — single shared row (`movie_collection`, id=1) in existing Supabase project; no user auth required
- **Auto-sync** — every change (`persist()`) silently pushes to cloud; bulk enrich pushes once at the end
- **Auto-pull on load** — `onCloudInit()` pulls automatically if local collection is empty; notifies if counts differ
- **Simplified Settings UI** — removed signed-in/signed-out states; sync is always active; Pull from Cloud and Force Push available as manual overrides
- **Sync status icon** — ☁ in header shows syncing/synced/error state

---

## ✅ Completed (2026-04-23)

- **Mobile card info visibility** — Changed `aspect-ratio: unset` to `aspect-ratio: auto` for broader browser compatibility; added `overflow: visible` on mobile `.movie-card` so the info strip renders below the poster
- **Director & Actor filter dropdowns** — Added `fDirFilter` and `fActorFilter` selects to the controls bar, properly wired into `getFiltered()` and populated by `render()`
- **IMDb Rating sort** — Added "IMDb Rating ↓" option to the sort dropdown
- **Multi-genre support** — `fillForm`, `enrichOne`, `bulkEnrich` now store all genres (comma-separated) instead of just the first; genre filter and dropdown split/deduplicate across all genres
- **Keyboard shortcut hint** — `⌨` icon in the controls bar with tooltip showing `A = Add movie · Esc = Close modal`
- **No API key banner** — Add Movie modal shows a prominent warning with a Settings shortcut when no OMDb key is set
- **Last-pushed timestamp** — Settings now displays "Last pushed: Apr 22, 2026 at 3:14 PM" after each successful cloud push

---

## 🟢 Future / Nice to Have

### 1. On-device mobile verification
The mobile card info CSS fix is in place (`position: static`, `aspect-ratio: auto`, `overflow: visible`) but was previously reported as having no effect. Recommend opening the deployed Cloudflare URL on an actual mobile device to confirm the info strip is now visible below each poster card.

### 2. Multi-genre: update genre form field label/hint
The genre input in the Add/Edit modal now stores all genres (e.g. "Drama, Romance") but the field label still just says "Genre". Consider adding a hint: "e.g. Drama, Romance (all genres stored)".

### 3. Bulk enrich rate-limit feedback
During bulk enrich, if OMDb returns a rate-limit error mid-run, it's silently skipped. Could surface a toast or stop the run and show how many were completed before hitting the limit.
