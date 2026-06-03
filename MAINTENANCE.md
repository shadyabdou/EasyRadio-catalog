# Maintaining the GitHub radio catalog

The iOS app loads station data from the **public** catalog repo (raw URLs). It does **not** call RapidAPI at runtime.

## What you must do

### 1. Publish catalog after each sync

Sync writes to `data/radio/` in this repo. The app reads the **public** mirror:

`https://raw.githubusercontent.com/shadyabdou/EasyRadio-catalog/main/`

```bash
# After syncing locally:
git add data/radio
git commit -m "chore(data): refresh radio catalog"
git push   # optional backup in private EasyRadio repo

./scripts/publish-radio-catalog.sh   # required — updates public EasyRadio-catalog
```

Change branch or mirror via UserDefaults key `GitHubRadioBaseURL` (full catalog base URL, optional).

**Local Xcode runs:** If `data/radio/` exists at `$(SRCROOT)/data/radio` (see `RadioCatalogRoot` in `Info.plist`), Debug builds load JSON from disk automatically. **Release / TestFlight** use the public catalog URL above.

### 2. Refresh data on a schedule

| Task | Command | How often |
|------|---------|-----------|
| Full station catalog | `npm run sync:stations` | Weekly or when 50k adds stations |
| Metadata (countries, genres, languages) | `npm run sync:metadata` | Same as above |
| Cities map | `SYNC_PROFILE=cities npm run sync` | Monthly |
| Popular / genres / languages | `npm run sync:extras` (needs `RAPIDAPI_KEY`) | After metadata + stations |
| Random pool only (no API) | `npm run sync:random-local` | After `sync:stations` |
| US states (API) | `npm run sync:states` (needs `RAPIDAPI_KEY`; `/states` + `/states/{id}/radios`) | After `sync:stations` |
| US states (offline from `us.json`) | `npm run build:us-states` or `npm run sync:states-catalog` | After `sync:stations` when API quota is unavailable |
| Search index only | `npm run sync:search-index` | After any station file change |

```bash
export RAPIDAPI_KEY="your-key"   # only for sync scripts, never in the app
cd tools/radio-sync
npm run sync:metadata
npm run sync:stations
npm run sync:extras
git add data/radio && git commit -m "chore(data): weekly catalog sync" && git push
./scripts/publish-radio-catalog.sh
```

### 3. GitHub Actions (optional automation)

Workflow: `.github/workflows/sync-50k-radio-data.yml`

- Add secret **`RAPIDAPI_KEY`**
- Run manually or on schedule (`metadata` weekly; run `stations` / `extras` separately when quota allows)

### 4. After publishing new data

Users pick up new JSON on next launch (or when background refresh runs). To force locally during development: Settings → refresh stations, or clear app cache.

## RapidAPI quota tips

- Station sync ≈ 550 API pages for all countries.
- Extras (popular + 277 genres + 89 languages) ≈ 600+ more calls.
- Use `SYNC_RESUME=1` and `SYNC_REFILL_UNDERCOUNT=1` to continue partial runs.
- If quota fails mid-run, rerun the same command the next day.

## Files the app depends on

| Required | Path |
|----------|------|
| Yes | `app-config.json` (force update policy) |
| Yes | `countries.json`, `genres.json`, `languages.json` |
| Yes | `stations/by-country/{iso}.json` + `by-country-index.json` |
| Yes | `search/stations-index.json` |
| Map | `cities-index.json`, `cities/page-*.json` |
| Browse by genre | `stations/by-genre/{slug}.json` |
| Browse by language | `stations/by-language/{code}.json` |
| Popular row | `stations/popular/{iso}.json` |
| US state submenu | `states/us.json`, `stations/by-state/{slug}.json`, `stations/by-state-index.json` |
| Random discovery | `stations/random.json` (built with `npm run sync:random-local` from by-country files; app never calls RapidAPI) |

## Force update (`app-config.json`)

The app loads `app-config.json` from the same catalog base URL as station JSON (`GitHubRadioConfig` / `EasyRadio-catalog`).

| Field | Purpose |
|-------|---------|
| `minimumVersion` | Marketing version users must meet when force update is on (matches `CFBundleShortVersionString`, e.g. `1.2.0`) |
| `forceUpdate` | `true` = enforce; `false` = ignore (disable without shipping a new app build) |
| `message` | Optional blocking-screen text |
| `appStoreURL` | App Store product URL for the Update button (e.g. `https://apps.apple.com/app/id…`) |

Example — require 1.2.0+ immediately after that build is live:

```json
{
  "minimumVersion": "1.2.0",
  "forceUpdate": true,
  "message": "Please update EasyRadio to continue listening.",
  "appStoreURL": "https://apps.apple.com/app/idYOUR_APP_ID"
}
```

Publish with `./scripts/publish-radio-catalog.sh` so production picks up the change. Debug builds read `data/radio/app-config.json` locally when `RadioCatalogRoot` is set.

Optional fallback: set `AppStoreListingURL` in `Info.plist` if `appStoreURL` is omitted in JSON.

## App-side maintenance

- No API keys in the app binary.
- Sync tooling lives in `tools/radio-sync/` (Node 20+).
- To point at another repo/branch, set `GitHubRadioBaseURL` to the catalog base URL (repo root, not `data/radio/`).
- Public catalog repo: https://github.com/shadyabdou/EasyRadio-catalog
