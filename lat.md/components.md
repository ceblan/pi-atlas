# Components

The UI layer is built from a set of composable TUI components in `src/components/`.

## Container Components

| Component | File | Purpose |
|-----------|------|---------|
| `BorderBox` | `@local/pi-tui-extras` | Box border with optional title/footer |
| `Container` | `@earendil-works/pi-tui` | Base flex container for children |

## Dashboard-Level Components

Components at the top of the dashboard hierarchy.

| Component | File | Purpose |
|-----------|------|---------|
| `Dashboard` | `Dashboard.ts` | Top-level container, keyboard routing |
| `Header` | `Header.ts` | Framed range selector with title |
| `TabBar` | `TabBar.ts` | Horizontal tab labels with active indicator |
| `RangeSelector` | `RangeSelector.ts` | Cycles through 1d/7d/30d/All |
| `LoadingView` | `LoadingView.ts` | Progress bar during session parsing |

## Overview-Specific Components

Used only on the Overview tab.

| Component | File | Purpose |
|-----------|------|---------|
| `KpiCards` | `KpiCards.ts` | 6-card grid (2 rows × 3 columns) |
| `BarChart` | `BarChart.ts` | ASCII bar chart with auto-scaled Y-axis |
| `RankedBarList` | `RankedBarList.ts` | Ranked items with UsageRow per entry |

## Table Components

Interactive table rendering components.

| Component | File | Purpose |
|-----------|------|---------|
| `SortedTable` | `SortedTable.ts` | Scrollable table with cursor, sort, marquee |
| `MarqueeText` | `MarqueeText.ts` | Auto-scrolling text at 150ms/char |
| `cells.ts` | `cells.ts` | Factory functions for CellComponent |

## Shared Layout Components

Reusable layout primitives.

| Component | File | Purpose |
|-----------|------|---------|
| `Bar` | `shared/Bar.ts` | Renders ■ block characters |
| `GridRow` | `shared/GridRow.ts` | Percentage-based column distribution |
| `StatCard` | `StatCard.ts` | Two-line label + value card |
| `UsageRow` | `UsageRow.ts` | Two-line ranked bar row |
