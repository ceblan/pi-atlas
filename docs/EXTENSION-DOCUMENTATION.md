# pi-atlas Extension — Complete Documentation

**Version:** 0.2.0  
**Package:** `@mohndoe/pi-atlas`  
**License:** MIT  
**Author:** Kevin P. (@mohndoe)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Data Flow](#data-flow)
3. [Entry Point — `src/index.ts`](#entry-point--srcindexts)
4. [Parser — `src/parser.ts`](#parser--srcparserts)
5. [Compute — `src/compute.ts`](#compute--srccomputets)
6. [Cache — `src/cache.ts`](#cache--srccachets)
7. [Types — `src/types.ts`](#types--srctypests)
8. [Format Utilities — `src/format.ts`](#format-utilities--srcformatmts)
9. [Color Palette — `src/colorPalette.ts`](#color-palette--srccolorpalettets)
10. [Helpers](#helpers)
11. [Components](#components)
12. [Tabs](#tabs)
13. [Configuration](#configuration)
14. [Tests](#tests)
15. [Security Considerations](#security-considerations)
16. [Environment Variables](#environment-variables)

---

## Architecture Overview

pi-atlas is a TUI extension for the [pi](https://pi.dev) coding agent that provides an interactive dashboard for viewing agent usage analytics. It parses session log files (`.jsonl` format) stored in `~/.pi/agent/sessions/`, aggregates the data into daily summaries, and renders an interactive terminal dashboard with multiple tabs showing costs, languages, models, projects, skills, and tool usage.

### Module Structure

```
src/
├── index.ts              — Extension entry point. Registers /atlas command.
├── types.ts              — Shared TypeScript types and interfaces.
├── parser.ts             — JSONL parser: .jsonl files → SessionAgg per file.
├── compute.ts            — Pure summarize(): SessionAgg[] × TimeRange → StatsSummary.
├── cache.ts              — loadAggregate(): SHA-256-gated cache with JSONL fallback.
├── format.ts             — Cost/number/date formatters + language detection.
├── colorPalette.ts       — Chalk color mappings per language and provider.
├── helpers/
│   └── session.helper.ts — SessionAgg factory functions.
├── components/           — Reusable TUI components.
│   ├── Dashboard.ts      — Main dashboard container (extends BorderBox).
│   ├── Header.ts         — Header with range selector box.
│   ├── TabBar.ts         — Horizontal tab navigation bar.
│   ├── RangeSelector.ts  — Time range selector (1d/7d/30d/All).
│   ├── KpiCards.ts       — Grid of 6 KPI stat cards.
│   ├── BarChart.ts       — ASCII bar chart (daily/hourly spend).
│   ├── SortedTable.ts    — Scrollable table with sort indicators and cursor.
│   ├── RankedBarList.ts  — Ranked bar visualization (languages, tools).
│   ├── UsageRow.ts       — Two-line row for ranked bar lists.
│   ├── StatCard.ts       — Compact label + value card.
│   ├── MarqueeText.ts    — Auto-scrolling text for overflow.
│   ├── LoadingView.ts    — Progress bar shown during parsing.
│   ├── cells.ts          — CellComponent factories for SortedTable.
│   └── shared/
│       ├── Bar.ts        — Horizontal bar renderer (■ characters).
│       └── GridRow.ts    — Percentage-based column layout.
├── tabs/                 — Tab content implementations.
│   ├── Overview.ts       — KPI cards + bar chart + top items.
│   ├── Languages.ts      — Language table (SortedTable).
│   ├── Models.ts         — Model table (SortedTable).
│   ├── Projects.ts       — Project table (SortedTable).
│   ├── Skills.ts         — Skill table (SortedTable).
│   └── Usage.ts          — Token breakdown + tool table.
└── __tests__/            — Test files mirror src layout.
```

### Integration with pi

The extension registers via the `pi` field in `package.json`:

```json
{
  "pi": {
    "extensions": ["./src/index.ts"],
    "image": "https://github.com/mohndoe/pi-atlas/raw/main/media/screenshot.png"
  }
}
```

The entry point (`src/index.ts`) exports a default function that receives an `ExtensionAPI` object. It registers the `/atlas` command, which opens the dashboard when invoked.

### Dependencies

**Runtime:**
- `chalk` (^5.6.2) — Terminal string styling
- `@mohndoe/pi-tui-extras` (^0.1.4) — TUI components (BorderBox, align utilities)

**Peer:**
- `@earendil-works/pi-tui` (>=0.74.0 <0.77.0) — pi TUI framework
- `typescript` (^5)

**Dev:**
- `@earendil-works/pi-coding-agent` (>=0.74.0 <0.77.0) — pi coding agent core
- `@earendil-works/pi-ai` (>=0.74.0 <0.77.0) — AI provider types
- `@earendil-works/pi-agent-core` (>=0.74.0 <0.77.0) — Agent message types
- `@types/bun` — Bun type definitions

---

## Data Flow

```
~/.pi/agent/sessions/*.jsonl
         │
         ▼ parseFile()        ◄── per-file parsing
  ┌──────────────────┐
  │  SessionAgg[]     │   one per JSONL file
  └────────┬─────────┘
           │
           ▼ summarize(days, range, filters)
  ┌──────────────────┐
  │ StatsSummary × 4  │   1d, 7d, 30d, All pre-computed
  └────────┬─────────┘
           │
           ▼
  Tab receives StatsSummary  ──→  Component render
```

### Step-by-step:

1. **Command invocation:** User types `/atlas` in pi terminal.
2. **Loading phase:** `loadAggregate()` checks for cached data. If cache is valid (SHA-256 signature matches session directory), returns cached `SessionAgg[]`. Otherwise, parses all `.jsonl` files.
3. **Parsing:** Each `.jsonl` file is parsed line-by-line. Each line is a JSON object representing a session entry (session header, message, model change, etc.). Entries are aggregated into a single `SessionAgg` per file.
4. **Computation:** `summarize()` filters `SessionAgg[]` by time range and aggregates into a `StatsSummary` containing all metrics. All four ranges (1d, 7d, 30d, All) are computed upfront.
5. **Rendering:** The Dashboard displays the appropriate tab using the selected range's `StatsSummary`.

---

## Entry Point — `src/index.ts`

### `default function(pi: ExtensionAPI)`

The extension's main export. Registers the `/atlas` command with pi.

**Constants:**
- `SESSIONS_DIR` = `~/.pi/agent/sessions/` — Where pi stores session logs.
- `CACHE_PATH` = `~/.pi/pi-atlas-cache.json` — Where the cache is stored.

**Command handler flow:**

1. **Validation:** Checks `ctx.hasUI`. If false, notifies user that interactive mode is required and returns.
2. **Overlay options:** Configures the dashboard as a centered overlay popup:
   - `minWidth: 100`
   - `width: "50%"`
   - `maxHeight: "80%"`
   - `anchor: "top-center"`
   - `margin: 2`
3. **Read cache timestamp:** Calls `getCacheTimestamp()` to get the last update time before loading (the cache file may be rewritten during loading).
4. **Phase 1 — Loading:** Shows `LoadingView` with progress bar. Calls `loadAggregate()` which either returns cached data or parses all JSONL files. Progress updates are forwarded to the loading view.
5. **Phase 2 — Dashboard:** Pre-computes `StatsSummary` for all four time ranges. Creates `RangeSelector` with options "Today", "Last 7 days", "Last 30 days", "All time". Instantiates `Dashboard` and passes input events to it.

**Key detail:** The `done(undefined)` callback in the loading phase allows the user to cancel (via Esc/q), which causes `days` to be `undefined` and the handler returns early.

---

## Parser — `src/parser.ts`

The parser converts `.jsonl` session log files into `SessionAgg` objects. Each file produces one `SessionAgg` that aggregates all entries in that file.

### Global State: Active Skill Tracking

```typescript
let activeSkill: { name: string; counted: boolean } | null = null;
```

The parser maintains a module-level variable tracking the currently active skill. This is used for cost attribution to skills.

**Functions:**
- `getActiveSkill(): string | null` — Returns the name of the currently active skill.
- `resetActiveSkills(): void` — Clears the active skill. Called at the start of each `parseFile()` and each `parseUserMessage()`.

### Skill Detection

Skills are detected via two paths:

1. **Explicit invocation:** User message contains `<skill name="X">`. Detected via regex `/<skill\s+name="([^"]+)"/g` in `parseUserMessage()`. Only the last tag wins if multiple appear.
2. **Implicit invocation:** Agent reads a `SKILL.md` file via the `read` tool. Detected in `parseAssistantMessage()` as a tool call with path matching `/\/SKILL\.md$/`. The skill name is extracted from the parent directory name.

**Active skill stack:** A skill is pushed when invoked (user tag or agent SKILL.md read). The stack is cleared at the start of each user message (new user topic = scope boundary). Assistant message cost is attributed to all skills on the active stack.

### Entry Point Functions

#### `parseFile(filePath: string, onWarning?: (count: number) => void): SessionAgg | null`

Parses a complete `.jsonl` file. Reads the file synchronously, splits by newline, and processes each line.

**Behavior:**
- Returns `null` if file cannot be read or contains no valid entries.
- Calls `resetActiveSkills()` before parsing and in a `finally` block.
- Counts corrupt lines and reports via `onWarning` callback.
- Corrupt lines are logged to `console.error` but do not stop parsing.

#### `parseSessionLogEntry(entry: FileEntry): SessionAgg | null`

Top-level dispatch for a single JSONL entry. Handles the following entry types:

| Entry Type | Handler | Tracked Data |
|------------|---------|--------------|
| `session` | `parseSessionHeader()` | Session ID, timestamp, project, cwd |
| `message` | `parseAgentMessage()` → role-specific handlers | User messages, tool results, assistant messages, model usage, tool calls, language usage, skill costs |
| `model_change` | `parseModelChangeEntry()` | Model change count |
| `thinking_level_change` | `parseThinkingLevelChangeEntry()` | Per-level thinking count |
| `compaction` | `parseCompactionEntry()` | Compaction count, compacted tokens |
| `branch_summary` | Skipped | — |
| `custom` | Skipped | — |
| `custom_message` | Skipped | — |
| `label` | Skipped | — |
| `session_info` | Skipped | — |

### Message Parsing Functions

#### `parseSessionHeader(entry: SessionHeader): SessionAgg`

Creates a `SessionAgg` from a session header entry. Extracts project name from `cwd` using `projectNameFromCwd()` (takes the basename of the directory).

#### `parseUserMessage(msg: UserMessage): SessionAgg`

Processes a user message. Increments `userMsgs` count. Resets the active skill stack (new user message = new topic boundary). Detects explicit skill invocations via `<skill name="X">` tags in the message content.

#### `parseToolResultMessage(msg: ToolResultMessage): SessionAgg`

Processes a tool result message. Increments `toolResults` count. Does not attribute tool name to any specific model (tool results don't carry model info).

#### `parseAssistantMessage(msg: AssistantMessage): SessionAgg`

Processes an assistant message. This is the most complex parser function.

**Behavior:**
1. Detects implicit skill invocation (agent reads `SKILL.md` via `read` tool).
2. Attributes cost to the active skill (if any) from `msg.usage`.
3. Builds a `SessionModelUsage` entry for the model used.
4. Extracts tool calls from content blocks and attributes to the model.
5. For `edit` and `write` tool calls, counts language usage (lines and edits).

**Language counting for `edit` tool:**
- Each edit entry counts `countNewLines(newText) + countNewLines(oldText) + 1` lines.
- Each edit entry increments `edits` by 1.
- `countNewLines()` counts occurrences of `\n` in the string.

**Language counting for `write` tool:**
- Counts `1 + countNewLines(content)` lines.
- No edit count increment.

#### `parseModelChangeEntry(entry: ModelChangeEntry): SessionAgg`

Increments `modelChanges` count.

#### `parseThinkingLevelChangeEntry(entry: ThinkingLevelChangeEntry): SessionAgg`

Increments the count for the specific thinking level (e.g., "low", "high", "xhigh") in `thinkingLevelCount`.

#### `parseCompactionEntry(entry: CompactionEntry): SessionAgg`

Increments `compactionCount` by 1 and `compactedTokens` by `entry.tokensBefore`.

### Merge Functions

#### `mergeToSession(base: SessionAgg, update: SessionAgg): void`

Merges a partial `SessionAgg` into a base session. Used to combine multiple entries from the same file into a single aggregate.

**Merging logic:**
- Scalar fields: simple addition (`userMsgs`, `toolResults`, `compactionCount`, `compactedTokens`, `modelChanges`).
- `thinkingLevelCount`: per-level addition.
- `skills`: deep merge with cost, token, and call accumulation.
- `models`: deep merge per provider → model. Usage fields (cost, tokens) are summed. Tools and languages are merged per-key.

### Utility Functions

#### `safeJsonParse(raw: unknown): Record<string, unknown> | undefined`

Parses JSON safely, returning `undefined` on failure instead of throwing. Handles string, object, null, and undefined inputs.

#### `sanitizeToolName(name: string): string`

Strips control characters (`\n`, `\r`, `\t`, etc.) from tool names to prevent rendering issues.

#### `mergeLangUsage(modelUsage: SessionModelUsage, toolName: string, args: unknown): void`

Extracts language usage from `edit` or `write` tool call arguments. Uses `langFromPath()` to detect language from file extension.

---

## Compute — `src/compute.ts`

The compute module is a pure function that filters and aggregates `SessionAgg[]` into a `StatsSummary`.

### `summarize(sessions: SessionAgg[], range: TimeRange, filters?: Filters): StatsSummary`

**Parameters:**
- `sessions` — Array of `SessionAgg` objects (one per parsed JSONL file).
- `range` — Time range filter: `"1d"`, `"7d"`, `"30d"`, or `"All"`.
- `filters` — Optional filters for project, model, or provider.

**Processing steps:**

1. **Time filtering** via `daysInRange()`:
   - `"1d"`: Sessions from today only.
   - `"7d"`: Sessions from the last 7 days (including today).
   - `"30d"`: Sessions from the last 30 days (including today).
   - `"All"`: All sessions.

2. **Project filtering:** If `filters.project` is set, only sessions matching that project are included.

3. **Accumulation loop:** Iterates over filtered sessions and accumulates:
   - Total cost, tokens (input/output/cache read/cache write), messages
   - Per-model usage (cost, calls)
   - Per-tool call counts
   - Per-language lines and edits
   - Per-project cost and session count
   - Per-skill cost, tokens, calls, sessions
   - Compaction stats, model changes, thinking level counts
   - Today's cost (for the "Today" KPI card)

4. **Derived metrics:**
   - `sessionCount`: Unique session IDs with at least one matching model.
   - `daysActive`: Unique dates with sessions having matching models.
   - `avgCostPerDay`: `totalCost / daysActive` (0 if no active days).
   - `todayCost`: Cost accumulated from sessions dated today.

5. **Sorting:**
   - `languages`: Sorted by lines descending.
   - `models`: Sorted by cost descending (secondary: calls descending).
   - `projects`: Sorted by cost descending (secondary: sessions descending).
   - `tools`: Sorted by count descending.
   - `providers`: Sorted by cost descending (secondary: calls descending).
   - `skills`: Sorted by cost descending (secondary: calls descending).

6. **Daily spend** via `fillDailySpend()`:
   - Groups sessions by date, sums cost per day.
   - For bounded ranges (1d, 7d, 30d): zero-fills gaps between first and last date.
   - For "All": returns only dates with actual data (no zero-fill).

7. **Hourly spend** via `buildHourlySpend()`:
   - Only computed for `"1d"` range.
   - Groups sessions by hour (0-23), sums cost per hour.
   - Returns 24 entries (one per hour), with 0 cost for hours with no activity.

### Helper Functions

#### `daysInRange(sessions: SessionAgg[], range: TimeRange): SessionAgg[]`

Filters sessions by date range. Uses `dateFromISOString()` to extract YYYY-MM-DD from timestamps.

#### `fillDailySpend(sessions: SessionAgg[], range: TimeRange): DaySpend[]`

Creates an array of `{ date, cost }` for the bar chart. For bounded ranges, fills gaps with zero-cost days.

#### `buildHourlySpend(filtered: SessionAgg[], range: TimeRange): HourSpend[]`

Creates an array of `{ hour, cost }` for the hourly bar chart (1d range only).

#### `filteredModels(session: SessionAgg, filters?: Filters): Generator`

Yields model entries from a session that pass the active filters (model name, provider).

---

## Cache — `src/cache.ts`

The cache module provides SHA-256-gated persistence for parsed session aggregates.

### Signature Computation

#### `computeSignature(sessionsDir: string): Promise<string>`

Computes a SHA-256 hash of the session directory's file structure.

**Algorithm:**
1. Recursively walks the directory, collecting all `.jsonl` files.
2. For each file: records `{ path, size, mtimeMs }`.
3. Sorts entries by path (deterministic ordering).
4. Hashes `${path}\n${size}\n${mtimeMs}\n` for each entry.
5. Returns the hex digest, or empty string if no files found.

**Invalidation triggers:**
- File added or removed (path changes).
- File modified (size or mtime changes).

### Cache I/O

#### `writeCache(cachePath: string, signature: string, sessions: SessionAgg[]): Promise<void>`

Writes a `CachePayload` to disk as JSON. Includes:
- `version`: Package version from `package.json`.
- `signature`: SHA-256 hash of the session directory.
- `generatedAt`: ISO timestamp of when the cache was written.
- `sessions`: The aggregated session data.

#### `readCache(cachePath: string): Promise<CachePayload | null>`

Reads and parses the cache file. Returns `null` if:
- File doesn't exist.
- JSON parsing fails.
- Missing `signature` field.
- `sessions` is not an array.

**Note:** Does not validate the signature — use `isCacheValid()` for that.

#### `getCacheTimestamp(cachePath: string): Promise<string | null>`

Returns the `generatedAt` timestamp from the cache, or `null` if cache doesn't exist.

#### `isCacheValid(cachePath: string, sessionsDir: string): Promise<boolean>`

Checks if the cache is still valid by comparing the stored signature against a freshly computed one.

**Important:** Internally re-reads the cache file (minor I/O overhead on cache-hit path).

### Aggregate Loading

#### `loadAggregate(cachePath: string, sessionsDir: string, force?: boolean, onProgress?: (p: LoadingProgress) => void): Promise<SessionAgg[]>`

Main entry point for loading session data.

**Algorithm:**
1. Check `PI_ATLAS_FORCE_CACHE` env var (overrides `force` parameter).
2. If not forced: read cache, check version matches current package version, check signature validity.
3. If cache is valid: return cached sessions.
4. Otherwise: find all `.jsonl` files in `sessionsDir`, parse each one, collect `SessionAgg[]`.
5. Sort sessions chronologically (by timestamp, then sessionId).
6. Write new cache with current signature.
7. Return sessions.

**Progress reporting:**
- Calls `onProgress` with `{ total, done, pct, remainingTimeMs }` after each file.
- `remainingTimeMs` is only estimated after 3+ files (too noisy before that).
- `PI_ATLAS_SLOW_DELAY_MS` env var adds artificial delay per file (for testing progress UI).

**Debug flags:**
- `PI_ATLAS_FORCE_CACHE=1` — Skip cache, always re-parse.
- `PI_ATLAS_SLOW_DELAY_MS=<ms>` — Add delay per file.

---

## Types — `src/types.ts`

### Core Types

#### `TimeRange`
```typescript
type TimeRange = "1d" | "7d" | "30d" | "All";
```

#### `DaySpend`
```typescript
interface DaySpend {
  date: string; // "YYYY-MM-DD"
  cost: number;
}
```

#### `HourSpend`
```typescript
interface HourSpend {
  hour: number; // 0-23
  cost: number;
}
```

#### `LangStat`
```typescript
interface LangStat {
  language: string;
  lines: number;
  edits: number;
}
```

#### `ModelStat`
```typescript
interface ModelStat {
  provider: string;
  model: string;
  cost: number;
  calls: number;
}
```

#### `ProjectStat`
```typescript
interface ProjectStat {
  project: string;
  cost: number;
  sessions: number;
}
```

#### `ToolStat`
```typescript
interface ToolStat {
  name: string;
  count: number;
}
```

#### `ProviderStat`
```typescript
interface ProviderStat {
  provider: Provider;
  cost: number;
  calls: number;
}
```

#### `SkillUsage`
```typescript
interface SkillUsage {
  cost: number;
  tokens: { input: number; output: number; total: number };
  calls: number;
}
```

#### `SkillStat`
```typescript
interface SkillStat {
  name: string;
  calls: number;
  sessions: number;
  cost: number;
  tokens: number;
}
```

### Aggregate Types

#### `StatsSummary`
The final aggregated statistics passed to tabs. Contains all metrics for a given time range.

```typescript
interface StatsSummary {
  totalCost: number;
  sessionCount: number;
  totalMessages: number;
  totalTokens: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCacheReadTokens: number;
  totalCacheWriteTokens: number;
  daysActive: number;
  avgCostPerDay: number;
  todayCost: number;
  languages: LangStat[];
  models: ModelStat[];
  projects: ProjectStat[];
  tools: ToolStat[];
  providers: ProviderStat[];
  skills: SkillStat[];
  compactionCount: number;
  compactedTokens: number;
  modelChanges: number;
  thinkingLevelCount: Record<string, number>;
  dailySpend: DaySpend[];
  hourlySpend: HourSpend[];
}
```

#### `SessionAgg`
Per-file aggregate of all session log entries.

```typescript
interface SessionAgg {
  timestamp: SessionHeader["timestamp"];
  sessionId: SessionHeader["id"];
  project: string;
  cwd: SessionHeader["cwd"];
  models: Record<Provider, Record<Model<Api>["name"], SessionModelUsage>>;
  skills: Record<string, SkillUsage>;
  userMsgs: number;
  toolResults: number;
  compactionCount: number;
  compactedTokens: number;
  modelChanges: number;
  thinkingLevelCount: Record<string, number>;
}
```

#### `SessionModelUsage`
Per-model usage within a session.

```typescript
interface SessionModelUsage {
  provider: Provider;
  api: Api;
  usage: Usage;
  calls: number;
  asstMsgs: number;
  tools: Record<string, number>; // tool name → call count
  languages: Record<string, LangUsage>;
}
```

#### `LangUsage`
```typescript
interface LangUsage {
  lines: number;
  edits: number;
}
```

#### `Filters`
Optional filters for `summarize()`.

```typescript
interface Filters {
  project?: string;
  model?: string;
  provider?: string;
}
```

#### `CachePayload`
Structure of the cache file on disk.

```typescript
interface CachePayload {
  version: string;
  signature: string;
  generatedAt: string;
  sessions: SessionAgg[];
}
```

---

## Format Utilities — `src/format.ts`

### Language Detection

#### `EXT_TO_LANG: Record<string, string>`

Maps 70+ file extensions to language names. Examples:
- `.ts`, `.tsx`, `.mts`, `.cts` → `"TypeScript"`
- `.js`, `.jsx`, `.mjs`, `.cjs` → `"JavaScript"`
- `.py`, `.pyi` → `"Python"`
- `.rs` → `"Rust"`
- `.go` → `"Go"`

#### `langFromPath(path: string): string`

Extracts the file extension from a path and returns the corresponding language name. Falls back to `"Other"` for unknown extensions.

### Project Name Extraction

#### `projectNameFromCwd(cwd: string): string`

Returns the basename of the given directory path. Used to extract project name from session `cwd`.

### Date Utilities

#### `dateFromISOString(str: string): string`

Extracts the first 10 characters (YYYY-MM-DD) from an ISO date string.

#### `MONTH_NAMES: string[]`

Array of 3-letter month abbreviations: `["Jan", "Feb", ..., "Dec"]`.

### Number Formatting

#### `formatNumber(n: number): string`

Formats numbers with compact notation:
- `≥ 1B` → `"1.5B"`
- `≥ 1M` → `"1.5M"`
- `≥ 1k` → `"1.5k"`
- Otherwise → locale-formatted with up to 2 decimal places.

#### `formatCost(n: number): string`

Formats costs as USD with smart precision:
- `≥ $1M` → `"$1.5M"`
- `≥ $1k` → `"$1.5k"`
- `< $0.01` → Up to 4 decimal places (e.g., `"$0.0012"`).
- `≥ $1` → Rounded to nearest cent with floating-point compensation.
- `$0.01–$0.99` → Standard rounding to nearest cent.

### Timestamp Formatting

#### `formatCacheTimestamp(iso: string): string`

Formats a cache timestamp for display in the dashboard footer:
- Today: `"3:45 PM"`
- Yesterday: `"Yesterday 3:45 PM"`
- This year: `"Jun 8, 3:45 PM"`
- Other years: `"Jun 8, 2025"`

### Model Name Formatting

#### `stripAnsi(text: string): string`

Removes ANSI escape sequences and control characters from text.

#### `formatModelName(raw: string): string`

Formats model names for display:
1. Strips date suffixes (`-20250514` or `-2025-05-14`).
2. Replaces `-` and `_` with spaces.
3. Title-cases each word.

Example: `"claude-sonnet-4-20250514"` → `"Claude Sonnet 4"`.

---

## Color Palette — `src/colorPalette.ts`

### `ColorPalette` Class

```typescript
class ColorPalette {
  constructor(private mapping: Record<string, ChalkInstance>) {}
  getColor(name: string): ChalkInstance;
}
```

Returns the chalk instance for a given name, or `chalk.white` if not found.

### `langPalette: ColorPalette`

Maps 50+ language names to their official GitHub/linguist colors. Examples:
- TypeScript → `#3178C6`
- Python → `#3572A5`
- Rust → `#DEA584`
- Go → `#00ADD8`

### `modelPalette: ColorPalette`

Maps 30+ AI provider names to brand colors. Examples:
- `anthropic` → `#d66545`
- `openai` → `#0f9f7c`
- `google` → `#4285F4`
- `deepseek` → `#4D6BFE`
- `xiaomi` → `#FF6900`

---

## Helpers

### `src/helpers/session.helper.ts`

#### `makeSessionAgg(overrides: Partial<SessionAgg> & { sessionId: string }): SessionAgg`

Factory function that creates a `SessionAgg` with defaults. All fields default to zero/empty except `sessionId` (required) and `timestamp` (defaults to `new Date().toISOString()`).

#### `makeEmptySession(sessionId: string, date: Date, project?: string, cwd?: string): SessionAgg`

Convenience wrapper around `makeSessionAgg()` that takes a `Date` object and converts it to ISO string.

---

## Components

### `Dashboard` (`src/components/Dashboard.ts`)

**Extends:** `BorderBox` (from `@mohndoe/pi-tui-extras`)

The main dashboard container. Manages tab switching, range switching, and input routing.

**Constructor parameters:**
- `summaries: Map<TimeRange, StatsSummary>` — Pre-computed summaries for all ranges.
- `theme: Theme` — pi theme for styling.
- `tui: TUI` — TUI instance for terminal info and render requests.
- `updateLabel: string | null` — Footer label showing last cache update time.
- `rangeSelector: RangeSelector` — Range selector component.
- `onClose?: () => void` — Callback when dashboard is closed.

**Key behavior:**
- Title bar: `"Pi Atlas · v0.2.0"` (left) + range label with `[r]` hint (right).
- Footer: Last update timestamp (if available).
- Chrome rows constant: 3 (TabBar + separator + footer hint).
- Content height: `floor(terminal.rows × 0.8) - chrome - 2` (border lines).
- Rebuilds tabs when terminal is resized (content height changes).

**Input handling:**
- `Esc` / `q` / `Q` → Close dashboard.
- `←` / `→` → Switch tabs.
- `r` → Cycle time range (1d → 7d → 30d → All → 1d).
- `↑` / `↓` → Dispatch to active tab (for table scrolling).

**Empty states:**
- If all summaries have `sessionCount === 0`: Shows "No sessions found in ~/.pi/agent/sessions".
- If current range's summary has `sessionCount === 0`: Shows "No data for this time range".

### `Header` (`src/components/Header.ts`)

**Implements:** `Component`

A header component that wraps the `RangeSelector` in a `BorderBox`. Currently unused in the main dashboard (range label is in the Dashboard title bar instead).

**Layout:** 3 rows × full width. Left side: title/version (currently empty strings). Right side: Range selector box (17 chars wide).

### `TabBar` (`src/components/TabBar.ts`)

**Implements:** `Component`

Horizontal tab navigation bar. Renders tab labels with the active tab highlighted.

**Constructor:** `TabBar(tabs: string[], theme: Theme, activeIndex?: number)`

**Rendering:**
- Active tab: `theme.bg("selectedBg", theme.fg("accent", " Label "))`.
- Inactive tabs: `theme.fg("muted", " Label ")`.
- Tabs separated by spaces.
- Truncates to fit width if needed.

**Input:** `←` moves left (stops at 0), `→` moves right (stops at last tab).

**Default tabs:** `["Overview", "Languages", "Models", "Projects", "Skills", "Usage"]`

### `RangeSelector` (`src/components/RangeSelector.ts`)

**Implements:** `Component`

Displays the currently selected time range label. The `r` key cycles through options.

**Constructor:** `RangeSelector(theme: Theme, ranges: RangeOption[], selectedIndex?: number)`

**Properties:**
- `selectedValue: TimeRange` — The value of the selected range (e.g., `"1d"`).
- `selectedLabel: string` — The display label (e.g., `"Today"`).
- `selectedIndex: number` — Current index in the ranges array.

**Rendering:** Single line showing the selected label in accent color.

**Input:** `r` cycles to next option (wraps around). `enter` is a no-op.

### `KpiCards` (`src/components/KpiCards.ts`)

**Implements:** `Component`

Displays 6 KPI (Key Performance Indicator) cards in a single row.

**Constructor:** `KpiCards(kpis: KpiData, theme: Theme)`

**Layout:** Single `GridRow` with 6 columns at widths `[17%, 17%, 17%, 17%, 17%, 15%]`.

**Card order (left to right):**
1. Total Cost — `formatCost()`, "success" color
2. Tokens — `formatNumber()`, "error" color
3. Messages — `formatNumber()`, "borderAccent" color
4. Sessions — `formatNumber()`, "accent" color
5. Active Days — `formatNumber()`, "warning" color
6. Avg/Day — `formatCost()`, "border" color

### `BarChart` (`src/components/BarChart.ts`)

**Implements:** `Component`

ASCII bar chart for visualizing daily or hourly spend.

**Constructor:** `BarChart(data: DaySpend[], range: TimeRange, maxHeight: number, theme: Theme, yAxisSpacing?: number, hourlyData?: HourSpend[])`

**Rendering modes:**
- **Daily mode** (7d, 30d, All): Shows one bar per day with date labels.
- **Hourly mode** (1d): Shows 24 bars (one per hour) with hour labels.

**Bar rendering:**
- Uses `█` (full block) and `▄` (lower half block) characters.
- Auto-scales so the tallest bar fills available height.
- Half-block threshold: 0.5 (bar extends beyond integer row).

**Y-axis:**
- Shows formatted cost labels.
- Separator: `" │ "` (space, box-drawing line, space).
- Auto-density based on bar area height:
  - `≤ 6` rows: Every row gets a label.
  - `≤ 14` rows: Every other row.
  - `> 14` rows: Every 3rd row.
- Always shows `$0.00` baseline.

**X-axis labels:**
- 7d: Day-of-week abbreviations (Mon, Tue, etc.).
- 30d: Day numbers (1, 5, 10, 15, 20, 25, 1, first/last always shown).
- All: Month labels when month changes, first entry always labeled.
- 1d (hourly): Hour labels at intervals (4h, 6h, or 12h depending on width).

**Downsampling:** If data points exceed available columns, consecutive items are aggregated (costs summed).

**Granularity footer:** Right-aligned italic text showing "Daily", "Hourly", or "~Nd avg" / "~Nh avg" when downsampling.

### `SortedTable` (`src/components/SortedTable.ts`)

**Implements:** `Component`

Interactive scrollable table with sort indicators and cursor navigation.

**Constructor:** `SortedTable(config: SortedTableConfig, theme: Theme)`

**Config:**
- `columns: ColumnDef[]` — Column definitions with header and width.
- `rows: CellComponent[][]` — Data rows (each row is an array of cells).
- `maxHeight: number` — Maximum visible rows (including header).
- `sort?: SortConfig` — Initial sort column and direction.
- `cursor?: CursorOptions` — Cursor visibility and character.
- `tui: TUI` — TUI reference for marquee cells.

**Column widths:**
- Fixed number: exact character count.
- Percentage string (e.g., `"50%"`): percentage of available width.
- `"fill"`: Takes remaining space after fixed/percentage columns. Only one fill column allowed.

**Cursor:**
- Default character: `▌` (left half block).
- Follows focus (auto-scrolls when cursor moves past visible area).
- Can be disabled with `cursor: { enabled: false }`.

**Sort indicators:**
- Header cells show `▲` (ascending) or `▼` (descending) next to the column name.
- Indicators are appended after the header text, truncating if needed.

**Input:** `↑` moves cursor up, `↓` moves cursor down. Stops at boundaries.

### `RankedBarList` (`src/components/RankedBarList.ts`)

**Implements:** `Component`

Renders a list of items as ranked bars with percentages.

**Constructor:** `RankedBarList(items: RankedBarItem[], theme: Theme)`

**Rendering:** Each item produces 3 lines via `UsageRow`:
1. Name (left) + values (right).
2. Colored progress bar + percentage.
3. Empty spacer.

**Bar calculation:**
- `pct` = item's `primaryValue` / total `primaryValue` × 100.
- `barPct` = `pct` / `highestItemPct` × 100 (tallest bar = 100%).

### `UsageRow` (`src/components/UsageRow.ts`)

**Implements:** `Component`

Two-line row used inside `RankedBarList`.

**Line 1:** Bold name (left) + optional secondary text (muted) + main value (bold) (right).
**Line 2:** Colored progress bar (■ characters) + percentage (dimmed).

### `StatCard` (`src/components/StatCard.ts`)

**Implements:** `Component`

Compact two-line component: muted label on line 1, accent-colored value on line 2.

**Constructor:** `StatCard(params: StatCardParams, theme: Theme)`

Uses a `Box` internally with configurable padding (default 0).

### `MarqueeText` (`src/components/MarqueeText.ts`)

**Implements:** `Component`

Animated text that scrolls left-to-right when content overflows available width.

**Constructor:** `MarqueeText(text: string, tui: TUI)`

**Behavior:**
- If text fits within width: renders static text.
- If text overflows: starts a `setInterval` timer at 150ms per tick.
- Each tick advances the scroll offset by 1 character.
- Gap of 5 spaces separates end of text from its beginning when wrapping.
- Calls `tui.requestRender()` on each tick to trigger re-render.

**Methods:**
- `advance()` — Manually advance tick counter (for testing).
- `reset()` — Reset to start position and stop timer.
- `destroy()` — Stop timer and mark as closed.

### `LoadingView` (`src/components/LoadingView.ts`)

**Extends:** `BorderBox`

Progress bar shown during session log parsing.

**Constructor:** `LoadingView(message: string, theme: Theme, onClose?: () => void | null)`

**Layout:**
- Title: `"Pi Atlas · v0.2.0"`.
- Content: Loading message (left) + progress info (right).
- Progress bar: `█` (filled) and `░` (empty) characters.
- Footer: "Esc/q to cancel and close".

**Progress display:**
- `"10/20"` — Files processed / total files.
- `"~5.21s remaining · 10/20"` — With time estimate.
- Time formatting: seconds if < 60s, "Xm Ys" otherwise.

**Input:** `Esc`, `q`, `Q` → call `onClose`.

### `CellComponent` System (`src/components/cells.ts`)

Factory for creating cell components used in `SortedTable`.

**Interface:**
```typescript
interface CellComponent {
  render(width: number, state?: CellState): string;
  invalidate(): void;
}
```

**Cell types:**

| Factory | Behavior |
|---------|----------|
| `cell.text(content)` | Static text, truncated to width. |
| `cell.header(content)` | Header text with optional sort indicator (`▲`/`▼`). |
| `cell.marquee(content, tui)` | Auto-scrolling text when focused and overflowing. Shows ellipsis when unfocused. |
| `cell.bar(fillPct, filledStyle, emptyStyle)` | Horizontal progress bar using ■ characters. |

### `renderBar()` (`src/components/shared/Bar.ts`)

```typescript
function renderBar(
  width: number,
  fillPct: number,
  filledStyle: (text: string) => string,
  emptyStyle: "transparent" | ((text: string) => string),
  filledChar?: string,
  emptyChar?: string,
): string
```

Renders a horizontal bar using repeated characters. `fillPct` is clamped to 0–100. Default characters: `■` for both filled and empty (styled differently).

### `GridRow` (`src/components/shared/GridRow.ts`)

**Implements:** `Component`

Distributes child components across the full width using percentage-based column widths.

**Constructor:** `GridRow(children: Component[], cols: number[])` where `cols` are percentages (e.g., `[33, 33, 34]`).

**Rendering:** Renders each child at its computed width, then interleaves lines horizontally. Pads shorter children with spaces.

---

## Tabs

### `Overview` (`src/tabs/Overview.ts`)

**Extends:** `Container`

The default tab showing high-level metrics and a cost chart.

**Layout (top to bottom):**
1. **KPI Cards** — 6 cards in a bordered box: Total Cost, Tokens, Messages, Sessions, Active Days, Avg/Day.
2. **Bar Chart** — "Cost overtime" chart showing daily or hourly spend.
3. **Top Cards** — 3 cards in a `GridRow` at `[33%, 33%, 34%]`:
   - Top Language (with language color border, line count footer).
   - Top Model (with provider color border, cost footer).
   - Top Project (with text border, cost footer).

**Bar chart height:** `min(18, maxHeight - kpiCardsHeight - topCardsHeight)`.

### `Languages` (`src/tabs/Languages.ts`)

**Extends:** `Container`

Displays languages ranked by lines written.

**Table columns:**
| Column | Width | Sort |
|--------|-------|------|
| Name | fill | — |
| Share % | 20 | — |
| Edits | 10 | — |
| Lines | 12 | desc |

**Empty state:** "No language data for this time range".

**Row construction:** Each language gets:
- `cell.marquee(language)` — Language name with auto-scroll.
- `cell.bar(barPct, langPalette.getColor(language))` — Colored progress bar.
- `cell.text(edits)` — Edit count (muted).
- `cell.text(lines)` — Line count (bold).

### `Models` (`src/tabs/Models.ts`)

**Extends:** `Container`

Displays models ranked by cost.

**Table columns:**
| Column | Width | Sort |
|--------|-------|------|
| Name | fill | — |
| Provider | 20 | — |
| Cost % | 14 | — |
| Calls | 10 | — |
| Cost | 10 | desc |

**Row construction:** Each model gets:
- `cell.marquee(formatModelName(model))` — Formatted model name with auto-scroll.
- `cell.text(provider)` — Provider name (muted).
- `cell.bar(barPct, modelPalette.getColor(provider))` — Provider-colored progress bar.
- `cell.text(calls)` — Call count (muted).
- `cell.text(cost)` — Cost (bold, or "Free" if 0).

### `Projects` (`src/tabs/Projects.ts`)

**Extends:** `Container`

Displays projects ranked by cost.

**Table columns:**
| Column | Width | Sort |
|--------|-------|------|
| Name | fill | — |
| Share % | 14 | — |
| Sessions | 10 | — |
| Cost | 10 | desc |

**Row construction:** Each project gets:
- `cell.marquee(project)` — Project name with auto-scroll.
- `cell.bar(barPct, identity)` — White progress bar.
- `cell.text(sessions)` — Session count (muted).
- `cell.text(cost)` — Cost (bold, or "Free" if 0).

### `Skills` (`src/tabs/Skills.ts`)

**Extends:** `Container`

Displays skill usage ranked by cost.

**Table columns:**
| Column | Width | Sort |
|--------|-------|------|
| Name | fill | — |
| Invocations | 12 | — |
| Sessions | 8 | — |
| Tokens | 14 | — |
| Cost | 16 | desc |

**Row construction:** Each skill gets:
- `cell.text(name)` — Skill name (static, no marquee).
- `cell.text(calls)` — Invocation count (muted).
- `cell.text(sessions)` — Session count (muted).
- `cell.text(tokens)` — Token count (muted).
- `cell.text(cost)` — Cost (bold, or `$0` if 0).

### `Usage` (`src/tabs/Usage.ts`)

**Extends:** `Container`

Displays token breakdown and tool usage.

**Layout:**
1. **Token section** — Bordered box with title "Tokens" and total count. Contains a `GridRow` with 4 `StatCard`s at `[25%, 25%, 25%, 25%]`:
   - Input tokens
   - Output tokens
   - Cache Read tokens
   - Cache Write tokens

2. **Tool section** — SortedTable with tool usage.

**Table columns:**
| Column | Width | Sort |
|--------|-------|------|
| Command | fill | — |
| Share % | 20 | — |
| Calls | 12 | desc |

**Tool name sanitization:** `stripAnsi()` is applied, then truncated to 120 characters.

---

## Configuration

### `package.json` — pi config

```json
{
  "pi": {
    "extensions": ["./src/index.ts"],
    "image": "https://github.com/mohndoe/pi-atlas/raw/main/media/screenshot.png"
  }
}
```

### `tsconfig.json`

- Target: ESNext
- Module: Preserve (bundler mode)
- Strict mode enabled
- `noUncheckedIndexedAccess: true`
- `noImplicitOverride: true`
- No emit (type-checking only)

### File Paths

| Path | Purpose |
|------|---------|
| `~/.pi/agent/sessions/` | Session log directory (read-only) |
| `~/.pi/pi-atlas-cache.json` | Cache file (read-write) |

---

## Tests

### Test Strategy

Tests use **Bun's built-in test runner** (`bun:test`). The test suite covers:

1. **Unit tests** for pure functions (parser, compute, format, cache).
2. **Component tests** for TUI rendering and input handling.
3. **Integration tests** for Dashboard → Tab → SortedTable input dispatch.

### Test Files

| File | Coverage |
|------|----------|
| `src/cache.test.ts` | Cache signature, read/write, validation, loadAggregate, progress reporting |
| `src/components/BarChart.test.ts` | Daily/hourly rendering, width constraints, labels, y-axis, caching |
| `src/components/Dashboard.test.ts` | All tabs, empty states, tab switching, range switching, escape/q close |
| `src/components/KpiCards.test.ts` | Rendering, formatting, width constraints |
| `src/components/LoadingView.test.ts` | Progress display, time formatting, input handling |
| `src/components/MarqueeText.test.ts` | Scrolling, wrapping, timer, reset |
| `src/components/RangeSelector.test.ts` | Rendering, value selection |
| `src/components/RankedBarList.test.ts` | Bar calculation, empty state, caching |
| `src/components/SortedTable.test.ts` | Rendering, scrolling, width resolution, cursor, sort indicators, marquee lifecycle |
| `src/components/TabBar.test.ts` | Rendering, navigation, invalidation |
| `src/components/cells.test.ts` | All cell types: text, header, marquee, bar |
| `src/components/shared/Bar.test.ts` | renderBar function |
| `src/components/__tests__/SortedTable.integration.test.ts` | Dashboard → Models → SortedTable arrow key integration |
| `src/tabs/Overview.test.ts` | KPI + chart rendering, empty state |
| `src/tabs/Languages.test.ts` | Table rendering, empty state, marquee lifecycle |
| `src/tabs/Models.test.ts` | Table rendering, empty state, marquee lifecycle |
| `src/tabs/Projects.test.ts` | Table rendering, empty state, marquee lifecycle |
| `src/tabs/Skills.test.ts` | Table rendering, empty state, sort order |
| `src/tabs/Usage.test.ts` | Token section + tool table, empty state, marquee lifecycle |

### Test Fixtures

**`src/components/components.fixtures.ts`:**
- `makeTheme()` — Pass-through theme (all styling returns text unchanged).
- `testPalette()` — Empty color palette.
- `makeMockTUI()` — Mock TUI with no-op `requestRender()`.
- `makeRangeSelector()` — Range selector initialized to "All time".

**`src/compute.fixtures.ts`:**
- `makeSummary()` — Returns a `StatsSummary` with sample data for testing.

### Running Tests

```bash
bun test                    # Run all tests
bun test --coverage         # With coverage report
```

---

## Security Considerations

See `docs/SECURITY-AUDIT.md` for the full security audit.

**Key findings:**
- **No network activity** — All processing is local.
- **No code execution** — No `eval`, `exec`, or dynamic imports.
- **No telemetry** — No analytics or tracking code.
- **Read-only data access** — Only reads session logs, writes only to cache file.
- **Cache file** — Contains aggregated statistics (costs, counts), not message content.

---

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PI_ATLAS_FORCE_CACHE` | `0` | Set to `1` to skip cache and re-parse all session logs. |
| `PI_ATLAS_SLOW_DELAY_MS` | `0` | Adds artificial delay (ms) per file during parsing. For testing progress UI. |

---

## Glossary

| Term | Definition |
|------|------------|
| **Session Log** | A `.jsonl` file in `~/.pi/agent/sessions/` containing pi session entries. |
| **SessionAgg** | Per-file aggregate of all session log entries. |
| **StatsSummary** | Time-range-filtered aggregate of all SessionAgg data. |
| **Time Range** | Filter applied to data: 1d, 7d, 30d, or All. |
| **Day Aggregate** | Pre-computed rollup of session data for a single calendar day. |
| **Signature** | SHA-256 hash of session directory structure for cache invalidation. |
| **Language Count** | Lines edited or written, measured by splitting on `\n`. |
| **Skill Invocation** | Activation of a skill via user tag or agent SKILL.md read. |
| **Active Skill Stack** | Set of skills currently active for cost attribution. Cleared per user message. |
