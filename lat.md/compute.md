# Compute

The compute module (`src/compute.ts`) exposes a single pure function: `summarize()`. It filters `DayAgg[]` entries by time range, merges them into a single accumulator, sorts the resulting maps, and builds a time series for the Overview bar chart.

## Algorithm

`summarize()` performs four steps in order: time filtering, accumulation, sorting, and time series construction.

## `summarize()`

Type signature of the core summarize function:

\`\`\`typescript
function summarize(
  allDays: DayAgg[],
  range: TimeRange,
  now: Date = new Date(),
): StatsSummary
```

## TimeRange Options

| Range | Description |
|-------|-------------|
| `1d` | Today only |
| `7d` | Last 7 calendar days |
| `30d` | Last 30 calendar days |
| `all` | All available history |

## Time Series Construction

Daily cost arrays are built from the filtered `DayAgg[]` for the `BarChart`. X-axis adapts: day-of-week for 7d, calendar day every 5th for 30d, month labels when month changes for `all`.

## Output: `StatsSummary`

A `StatsSummary` contains all six tab datasets pre-computed for a given range: KPI values, ranked top-lists, daily time series, and raw sorted tables.
