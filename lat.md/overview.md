# Overview

pi-atlas is a TUI dashboard extension for pi. It reads session logs from `~/.pi/agent/sessions/*.jsonl`, aggregates them, and renders an interactive six-tab dashboard.

The extension registers as the `/atlas` command via the `pi` field in `package.json`, pointing to `src/index.ts` as the extension entry point. It has no runtime network activity and stores only aggregated statistics in a local cache file.

## Core Modules

The extension has four pure/data modules:

- **[[lat.md/parser]]** — parses `.jsonl` files into per-day `DayAgg` arrays
- **[[lat.md/cache]]** — SHA-256-gated cache with JSONL fallback
- **[[lat.md/compute]]** — pure `summarize()` merging and filtering
- **[[lat.md/format]]** — cost, number, language, and model name formatting

The UI layer consists of:
- **[[lat.md/dashboard]]** — main container, keyboard routing, range selector
- **[[lat.md/tabs]]** — six tab content components fed by `StatsSummary`
- **[[lat.md/components]]** — reusable TUI building blocks (BarChart, SortedTable, etc.)

## Data Flow

Session log files flow through the extension in four stages:

```
~/.pi/agent/sessions/*.jsonl
  → parser.parseFile()         → DayAgg[]  (per calendar day)
  → cache.loadAggregate()       → DayAgg[]  (cache or fresh)
  → compute.summarize()        → StatsSummary  (per TimeRange)
  → Dashboard component tree    → terminal render
```

## Keyboard Navigation

pi-atlas supports both arrow keys and vim-style navigation. See [[lat.md/navigation]].

## Version

Current version is `0.2.0`. Published as `@mohndoe/pi-atlas` on npm.

## Dependencies

Runtime dependencies are minimal: `chalk` for terminal colors and `@local/pi-tui-extras` (a local file: dependency). All AI SDKs are devDependencies only and are not shipped to users.
