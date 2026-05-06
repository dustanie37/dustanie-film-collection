# Dustanie Film Collection — Project Notes

## What This Is
A single-file web app (`index.html`) for tracking a personal movie collection. Built in HTML/CSS/JavaScript with no framework. Hosted on Netlify, synced via Firebase Realtime Database, accessible from any device.

---

## File Location
- **Local file:** `iCloud / Claude Cowork / Movie collection / index.html`
- **Cloudflare URL:** your Cloudflare Pages deployment URL

---

## Features
- Add movies manually or search OMDb to auto-fill details (poster, director, actors, genre, IMDb rating, runtime)
- Track format: Blu-ray / DVD / Digital / 4K
- Track edition: Standard / Special / Director's Cut / Criterion / Collector's / Limited / Unrated
- Mark movies as part of a box set / collection (with autocomplete)
- Wishlist mode (separate from main collection)
- Watched / unwatched status (togglable from main screen)
- Grid and list view
- Search, filter (by format, genre, watched status, wishlist), and sort
- Poster images with hover card showing full details
- Duplicate detection when adding movies
- Cloud sync across devices via Firebase (sign in with Google)
- Export / Import JSON backup
- Local backup snapshot (auto-saved before any destructive operation)
- Bulk OMDb enrich (fills in missing posters, ratings, runtime for all movies at once)

---

## API Keys & Services

### OMDb API
- Get a free key at: https://www.omdbapi.com/apikey.aspx
- Enter in the app via ⚙ Settings → OMDb API Key
- Stored in browser `localStorage` as `mc_omdb_key`

### Firebase (Cloud Sync)
- **Project:** `movie-collection-9432b`
- **Console:** https://console.firebase.google.com/project/movie-collection-9432b
- **Database type:** Realtime Database (NOT Firestore)
- **Database URL:** `https://movie-collection-9432b-default-rtdb.firebaseio.com`
- **Auth:** Google Sign-In enabled

**Firebase config (hardcoded in index.html):**
```javascript
const FIREBASE_CONFIG = {
  apiKey:            "AIzaSyBRyJT87IvRpQlGg1mWHqJKOXZm2vuZ8XQ",
  authDomain:        "movie-collection-9432b.firebaseapp.com",
  databaseURL:       "https://movie-collection-9432b-default-rtdb.firebaseio.com",
  projectId:         "movie-collection-9432b",
  storageBucket:     "movie-collection-9432b.firebasestorage.app",
  messagingSenderId: "422767045152",
  appId:             "1:422767045152:web:e99bf06765b9e9bfddaa45"
};
```

**Firebase Realtime Database rules** (set in Firebase Console → Realtime Database → Rules):
```json
{
  "rules": {
    "users": {
      "$uid": {
        ".read": "$uid === auth.uid",
        ".write": "$uid === auth.uid"
      }
    }
  }
}
```

**Data path in RTDB:** `users/{uid}/collection` → `{ movies: [...], updatedAt: timestamp, count: N }`

**Authorized domains** (Firebase Console → Authentication → Settings → Authorized domains):
- `localhost`
- `movie-collection-9432b.firebaseapp.com`
- Your Cloudflare Pages domain (e.g. `yourapp.pages.dev`)

---

## Local Storage Keys
| Key | Contents |
|-----|----------|
| `mc_movies_v2` | Full movies array (JSON) |
| `mc_omdb_key` | Your OMDb API key |
| `mc_last_push` | Timestamp of last successful cloud push |
| `mc_movies_backup` | Snapshot of collection before last import or pull |

---

## How Cloud Sync Works (Explicit Push/Pull Model)

Cloud sync is **intentionally explicit** — nothing syncs automatically. You are always in control of when data moves between devices.

**Sign-in behavior:**
- If local collection is empty, auto-pulls from cloud
- If local and cloud differ, shows a notification with counts but does NOT auto-overwrite
- Google Sign-In uses popup on desktop, falls back to redirect if popup is blocked (common on mobile)

**Push to Cloud:**
- Writes your entire local collection to Firebase
- Shows a real error toast if the write fails (no false success messages)
- Use this after importing a JSON file, or after adding movies on one device

**Pull from Cloud:**
- Fetches cloud data and shows you cloud count vs local count
- Asks for confirmation before overwriting
- Automatically saves a local backup snapshot first
- If pull fails, shows an error — your local data is untouched

**No real-time listener** — there is no background sync. Changes made on one device do not automatically appear on another. Push from one device, then pull on the other.

---

## Backup System

**Local backup** (in-browser, survives page reloads):
- Saved automatically before any import or cloud pull
- Can be saved manually via Settings → Save Backup Now
- Restored via Settings → Restore Last Backup
- Stored in `localStorage` as `mc_movies_backup`

**File backup** (recommended — keep a copy outside the browser):
- Use Settings → Export JSON regularly
- Save to iCloud, Dropbox, email to yourself, etc.
- This is your ultimate safety net — it works even if Firebase goes down

---

## Deploy to Cloudflare Pages
1. Go to https://dash.cloudflare.com → Log in
2. Navigate to Workers & Pages → Create → Pages → Upload assets
3. Drag and drop **both** `index.html` and `_headers` and deploy
4. Copy the generated URL and open it in any browser or on mobile

> The `_headers` file tells Cloudflare never to cache the page, so future deployments are live immediately at the same URL — no cache clearing needed. Always include both files when deploying.

---

## Standard Workflows

### Moving your collection to a new device
1. On new device: open Netlify URL, go to ⚙ Settings, sign in with Google
2. If local is empty, it auto-loads from cloud
3. If not, tap **Pull from Cloud** — confirm the movie counts look right

### After importing a JSON backup file
1. Settings → Import JSON → select file → confirm
2. Verify movie count is correct
3. Settings → **Push to Cloud** to save to Firebase
4. On other devices: Settings → Pull from Cloud

### Regular backup habit
1. Settings → Export JSON → save file to iCloud or email to yourself
2. Do this any time you make significant additions

---

## Architecture Notes
- SDKs: `firebase-app-compat`, `firebase-auth-compat`, `firebase-database-compat` (v9.22.2)
- Do NOT use `firebase-firestore-compat` — this app uses Realtime Database only
- No real-time listener (`ref.on()`) — all sync is explicit push/pull only
- `persist()` saves to localStorage only — it does NOT auto-write to Firebase
- `pushToCloud()` and `syncFromCloud()` are the only functions that touch Firebase data

---

## Known Bugs / To Fix

### 🐛 Mobile: truncated titles in both grid and list view
**Status:** Not fixed — confirmed still occurring after migration to Cloudflare
**Description:** On mobile, titles are truncated in both grid view and list view. In grid view, card info (title, year, collection) is not persistently visible — it hides behind the hover overlay which doesn't work on touch screens. In list view, long titles are cut off. Desired behavior: show full title (or at minimum 2-line wrap), year, and collection name on mobile in both views, without requiring a hover/tap into the detail screen.
**Attempted fix:** Added `@media (max-width: 640px)` rules to make `.card-info` `position: static` and the card a flex column; simplified `card-sub` to show only year + IMDb rating; added `.card-coll` for collection name; hid `.card-badges-desktop` on mobile. Had no visible effect.
**Next steps:** Inspect the rendered card HTML on mobile to confirm whether the media query is firing at all. The `overflow: hidden` on `.movie-card` combined with the `aspect-ratio: 2/3` may be preventing the flex column layout from working. Most likely fix: restructure the card HTML so the info section is a sibling element outside the poster container (not an absolute overlay inside it), then use CSS to show/hide it per breakpoint.

### 🐛 Add Movie: OMDb search always returns "Nothing found" error
**Status:** Not fixed — reported 2026-04-03
**Description:** When attempting to add a film via the Add Movie flow, every search returns the red "Nothing found" error regardless of the title entered. The OMDb lookup is failing for all queries.
**Likely causes:** OMDb API key may be missing, expired, or over the free-tier daily limit; the API key stored in `localStorage` (`mc_omdb_key`) may have been cleared; or the fetch request to OMDb is failing silently and falling through to the "not found" state.
**Next steps:** Check Settings → OMDb API Key to confirm a key is saved. Test the key directly at `https://www.omdbapi.com/?t=test&apikey=YOUR_KEY` to see if it returns results. If the key is valid, inspect the network request in browser DevTools to see the actual OMDb response.
