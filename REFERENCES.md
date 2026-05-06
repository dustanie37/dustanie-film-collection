# REFERENCES — Dustanie Film Collection

## Project Overview
Single-file web app (`index.html`) for tracking a personal movie collection. No framework — plain HTML/CSS/JS. Firebase Realtime Database for cloud sync. Deployed on Cloudflare Pages.

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
- **localStorage key:** `mc_movies_v2` — full movies array as JSON
- **SEED:** 42 hardcoded films (Cary Grant collection) used on first load
- **sanitizeMovies():** always call this before assigning any incoming array (localStorage, cloud, import) to ensure all fields exist with correct types
- **persist():** saves to localStorage only — never auto-writes to Firebase

### Supabase / Cloud Sync
- **Project:** `ecckvouhkoiuhdbynbqa.supabase.co` (shared with dustanie.com project)
- **Table:** `movie_collection` — single row (id=1), `movies` (jsonb), `updated_at`, `count`
- **Auth:** none required — anon key, shared collection for Dustin & Stephanie
- **Sync model:** explicit push/pull only — no real-time listener
- **pushToCloud():** POST with `Prefer: resolution=merge-duplicates` (upsert)
- **syncFromCloud():** GET `movie_collection?id=eq.1`
- **onCloudInit():** called on startup — auto-pulls if local is empty, notifies if counts differ
- No external SDK — vanilla fetch calls only

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

### Fixed (2026-04-23)
- **syncFromCloud() missing sanitizeMovies()** — pulled cloud data was not sanitized, risking render crashes with old-format data
- **Year dropdown no placeholder** — clearForm() set omdbY value to '' but no blank option existed, causing current year to stay selected and filter all searches
- **Duplicate seed entry** — "She Done Him Wrong" (1933) appeared twice in SEED (ids 32 and 43); duplicate removed
- **resetData() hardcoded count** — "43 movies" now uses `SEED.length` dynamically

### Known / Open
- **Mobile truncated titles** — card info always-visible fix is in CSS but reported as having "no visible effect" on some devices; needs on-device inspection
- **OMDb search "Nothing found"** — likely an expired/missing API key; the code handles it correctly, but the error message may not surface clearly enough
