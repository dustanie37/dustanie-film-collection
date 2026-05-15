# REFERENCES — Dustanie Film Collection

## Project Overview
Single-file web app (`index.html`) for tracking a personal movie collection. No framework — plain HTML/CSS/JS. Supabase for cloud sync (migrated from Firebase). Deployed on Cloudflare Pages at `movies.dustanie.com`. Auth via shared `auth.js` loaded from `dustanie.com`.

---

## Key Files
| File | Purpose |
|------|---------|
| `index.html` | Entire app — HTML, CSS, and JS in one file |
| `_headers` | Cloudflare cache-control headers (always deploy with index.html) |
| `project_notes.md` / `PROJECT_NOTES.md` | Architecture notes, API keys, known bugs, workflows |

---

## Architecture Summary

### Data Layer
- **Supabase is source of truth** — `init()` is `async` and awaits `onCloudInit()` before reading localStorage or loading seed
- **localStorage key:** `mc_movies_v2` — local cache only; populated from cloud on every load
- **SEED:** 42 hardcoded films used only when BOTH cloud and localStorage are empty (brand-new account)
- **sanitizeMovies():** always call before assigning any incoming array (localStorage, cloud, import)
- **persist():** saves to localStorage + auto-pushes to Supabase on every mutation (silent); skipped during bulk enrich
- **loadSeed():** writes localStorage only — never calls `persist()`, never touches Supabase

### Supabase / Cloud Sync
- **Project:** `ecckvouhkoiuhdbynbqa.supabase.co` (shared with dustanie.com project)
- **Table:** `movie_collection` — single row (id=1), `movies` (jsonb), `updated_at`, `count`
- **Auth:** anon key — shared collection, no per-user rows
- **onCloudInit():** awaited at top of `init()`. Count-based arbitration: if local > cloud → silent auto-push (recovery); if cloud ≥ local → load cloud (normal sync). Returns `true/false`.
- **pushToCloud(silent):** POST upsert with `Prefer: resolution=merge-duplicates`; silent=true for auto-saves
- **syncFromCloud():** still exists in code but no longer exposed in UI
- No external SDK — vanilla fetch calls only

### Settings (as of 2026-05-15)
Two sections only:
- **OMDb API Key** — stored in localStorage as `mc_omdb_key`
- **Backup** — Export JSON / Import JSON. Import calls `persist()` so cloud syncs immediately.

### OMDb Integration
- API key stored in `localStorage` as `mc_omdb_key`
- Search uses `?s=` endpoint for multi-result picker; `?i=` for detail fetch; `?t=` for single-title enrichment
- Bulk enrich: 220ms delay between requests to respect free-tier rate limit (1,000/day)

### State Variables
| Variable | Purpose |
|----------|---------|
| `movies` | Main array — source of truth for all rendering |
| `nextId` | Auto-incremented ID for new movies |
| `apiKey` | OMDb key (loaded from localStorage on init) |
| `editingId` | ID of movie currently in the edit modal (null = add mode) |
| `viewMode` | `'grid'` or `'list'` |
| `bulkRunning` | Lock flag to prevent concurrent bulk enrich runs |

### Render Pipeline
`render()` → `renderStats()`, `renderGenres()`, `renderCollections()`, `renderGrid()`, `renderEnrichBar()`  
`applyFilters()` → just calls `renderGrid()` (stats/genres/collections don't need re-render on filter)

### CSS Notes
- Two `@media (max-width: 640px)` blocks — first (line ~194) handles card layout; second (line ~561) handles general layout
- Mobile card fix: `.movie-card` becomes flex column; `.card-info` becomes `position: static` so it's always visible
- `overflow: hidden` on `.movie-card` is retained on mobile (doesn't clip static children)

---

## Bug History

### Fixed (2026-05-15)
- **Cross-device sync race condition** — new device loaded SEED, then `persist()` pushed 42 seed movies to Supabase before `onCloudInit()` could read the real collection. Fixed: `init()` now async, awaits cloud first; `loadSeed()` no longer calls `persist()`. Count-based arbitration added so local data auto-restores cloud if local has more movies.
- **Settings over-complexity** — removed manual Push/Pull buttons, Local Backup section, Export CSV, and Danger Zone. Not needed with automatic sync.

### Fixed (2026-04-23)
- **syncFromCloud() missing sanitizeMovies()** — pulled cloud data was not sanitized, risking render crashes with old-format data
- **Year dropdown no placeholder** — clearForm() set omdbY value to '' but no blank option existed, causing current year to stay selected and filter all searches
- **Duplicate seed entry** — "She Done Him Wrong" (1933) appeared twice in SEED (ids 32 and 43); duplicate removed
- **resetData() hardcoded count** — "43 movies" now uses `SEED.length` dynamically

### Known / Open
- **Mobile truncated titles** — card info always-visible fix is in CSS but reported as having "no visible effect" on some devices; needs on-device inspection
- **OMDb search "Nothing found"** — likely an expired/missing API key; the code handles it correctly, but the error message may not surface clearly enough
