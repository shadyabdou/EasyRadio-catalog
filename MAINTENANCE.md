# Maintaining the GitHub radio catalog

The iOS app loads all station data from **`data/radio/`** on GitHub (default: `main` branch, raw URLs). It does **not** call RapidAPI at runtime.

## What you must do

### 1. Keep `data/radio/` on GitHub

Push the folder to your default branch after every sync:

```bash
git add data/radio
git commit -m "chore(data): refresh radio catalog"
git push
```

The app reads:

`https://raw.githubusercontent.com/shadyabdou/EasyRadio/main/data/radio/`

Change branch or fork via UserDefaults key `GitHubRadioBaseURL` (optional).

**Local Xcode runs:** If `data/radio/` exists at `$(SRCROOT)/data/radio` (see `RadioCatalogRoot` in `Info.plist`), Debug builds load JSON from disk automatically. **Release / TestFlight** still need the catalog pushed to GitHub.

### 2. Refresh data on a schedule

| Task | Command | How often |
|------|---------|-----------|
| Full station catalog | `npm run sync:stations` | Weekly or when 50k adds stations |
| Metadata (countries, genres, languages) | `npm run sync:metadata` | Same as above |
| Cities map | `SYNC_PROFILE=cities npm run sync` | Monthly |
| Popular / genres / languages / random | `npm run sync:extras` | After metadata + stations |
| Search index only | `npm run sync:search-index` | After any station file change |

```bash
export RAPIDAPI_KEY="your-key"   # only for sync scripts, never in the app
cd tools/radio-sync
npm run sync:metadata
npm run sync:stations
npm run sync:extras
git add data/radio && git commit -m "chore(data): weekly catalog sync" && git push
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
| Yes | `countries.json`, `genres.json`, `languages.json` |
| Yes | `stations/by-country/{iso}.json` + `by-country-index.json` |
| Yes | `search/stations-index.json` |
| Map | `cities-index.json`, `cities/page-*.json` |
| Browse by genre | `stations/by-genre/{slug}.json` |
| Browse by language | `stations/by-language/{code}.json` |
| Popular row | `stations/popular/{iso}.json` |
| Random discovery | `stations/random.json` |

## App-side maintenance

- No API keys in the app binary.
- Sync tooling lives in `tools/radio-sync/` (Node 20+).
- To point at another repo/branch, set `GitHubRadioBaseURL` to the full `data/radio` base URL.
