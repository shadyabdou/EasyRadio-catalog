# Station search (GitHub-hosted catalog)

Pre-synced search replaces live `/radios/search` for most queries. The sync job writes:

| File | Role |
|------|------|
| `search/stations-index.json` | Compact lookup: `id`, `n` (name), `cc`, optional `c` (city), `g` (genre) |
| `stations/by-country/{iso}.json` | Full station payloads per country |

## Recommended app behavior

### 1. Local index search (primary)

1. Download `search/stations-index.json` once (or on manifest version change); cache under Application Support.
2. On each keystroke (debounced ~300ms), filter `entries` where `n`, `g`, or `c` contains the query (case-insensitive; split on spaces = AND).
3. Rank matches: name prefix > name substring > genre > city.
4. Take top 25–50 hits; load full `Station` objects from `stations/by-country/{cc}.json` by `id` (only touch countries present in results).

**Pros:** No API key on device, works offline after cache, predictable cost.  
**Cons:** ~8–15 MB index download; first search after install needs index fetch.

### 2. Scoped local search (lighter)

Ship or lazy-load **per-country** mini-index shards: `search/by-country/us.json`.  
Search only: user home region + countries they browsed.  
Pro tier: download full index or search all shards.

### 3. On-device FTS (best UX, more work)

CI builds `stations.sqlite` with FTS5 on `name`, `genre`, `city`. App ships DB or downloads it. Same GitHub raw URL pattern as JSON.

### 4. API search fallback (optional)

Keep RapidAPI `/radios/search` only when:

- local results &lt; 5 and query length ≥ 3, or
- user enables “Search online” / Pro global discovery

Document in privacy policy as optional network call.

## What not to do

- **Precompute every search query** — impossible.
- **Prefix trie for all strings** — huge repo, diminishing returns vs FTS.
- **Single 100 MB JSON of all stations** — hurts mobile; use per-country files + index.

## Example ranking (Swift)

```swift
func score(entry: SearchIndexEntry, query: String) -> Int {
    let q = query.lowercased()
    let name = entry.n.lowercased()
    if name.hasPrefix(q) { return 100 }
    if name.contains(q) { return 80 }
    if entry.g?.lowercased().contains(q) == true { return 50 }
    if entry.c?.lowercased().contains(q) == true { return 40 }
    return 0
}
```

Hydrate: fetch `by-country/{entry.cc}.json`, find `data.first { String($0.id) == entry.id }`, decode as `Station`.
