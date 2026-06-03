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
| Popular / genres / languages / random | `npm run sync:extras` | After metadata + stations |
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
- To point at another repo/branch, set `GitHubRadioBaseURL` to the catalog base URL (repo root, not `data/radio/`).
- Public catalog repo: https://github.com/shadyabdou/EasyRadio-catalog
