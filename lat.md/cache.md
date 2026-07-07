# Cache

The cache module (`src/cache.ts`) provides a SHA-256-gated caching layer in front of the JSONL parser. Its goal is to avoid re-parsing all session files on every `/atlas` invocation.

## Cache File Location

`~/.pi/pi-atlas-cache.json`

## Cache Validation

Before using a cached `DayAgg[]`, the cache is validated via `isCacheValid()`:

1. The directory signature is recomputed from `~/.pi/agent/sessions/` — file paths, sizes, and modification times — hashed with SHA-256.
2. If the hash matches the stored signature, the cache is considered valid.

## Cache Loading Flow (`loadAggregate`)

The `loadAggregate()` function implements the cache strategy described above.

```
loadAggregate(summaries: Map<TimeRange, StatsSummary>)
  → readCache()                    ← load from ~/.pi/pi-atlas-cache.json
  → isCacheValid()                 ← SHA-256 directory signature check
  → if valid: return cached DayAgg[]
  → if invalid or miss:
      → findAllJsonlFiles(sessionsDir)
      → parseFile() per file       ← full re-parse
      → writeCache()               ← write new cache
      → return fresh DayAgg[]
```

## Debug Environment Variables

Two undocumented env vars control behavior during development:
- `PI_ATLAS_FORCE_CACHE=1` — skips cache, forces full re-parse
- `PI_ATLAS_SLOW_DELAY_MS=<n>` — adds n ms delay per file (for testing loading UI)

These are not gated behind `NODE_ENV=development` and could theoretically be abused on shared systems.

## Known Issues

Bug fixes that address edge cases in the cache and parser modules.

### Invalid Date Handling

`makeEmptySession()` would produce `NaN` ISO strings for malformed session timestamps. The fix adds `isNaN(date.getTime())` guard before `toISOString()` in [[src/helpers/session.helper.ts]], falling back to the current time.

## Cache Content

The cache stores only aggregated `DayAgg[]` arrays — session counts, token counts, cost totals, model usage, skill usage, language counts. It does **not** store message content, API keys, or credentials.
