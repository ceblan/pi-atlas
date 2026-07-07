# Dashboard

The `Dashboard` class (`src/components/Dashboard.ts`) is the top-level TUI component. It extends `BorderBox` and owns the entire component tree — tab bar, range selector, and six tab content components.

## Responsibilities

The Dashboard class owns layout, keyboard routing, and range/tab state.

- **Layout**: Computes content height accounting for header, footer, dividers, and tab bar chrome (3 rows fixed).
- **Keyboard routing**: All keyboard input flows through `Dashboard.handleInput()`, which dispatches to the appropriate handler based on the active tab and key pressed.
- **Range cycling**: `r` key updates the `RangeSelector` and triggers tab invalidation.
- **Tab switching**: `←`/`→`/`h`/`l` keys route to the `TabBar`; `↑`/`↓`/`j`/`k` keys route to the active table tab (with j→↓ and k→↑ translation).

## Overlay Mode

In terminals ≥60 columns wide and ≥20 rows tall, the dashboard renders as a centered 50%-width overlay with rounded borders. Otherwise it fills the full terminal.

## Dependency Injection

`Dashboard` receives a `Map<TimeRange, StatsSummary>` in its constructor — all summaries for all four ranges are pre-computed before the dashboard opens. Switching ranges simply selects a different entry from this map, avoiding recomputation.

## Footer

The footer displays the last data update timestamp (e.g., "Updated 2 min ago") styled as muted italic text, aligned right.
