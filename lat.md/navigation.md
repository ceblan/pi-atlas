# Keyboard Navigation

pi-atlas supports dual keyboard schemes: standard arrow keys and vim-style h/j/k/l bindings. This was added in v0.2.0 to improve efficiency for users familiar with vim navigation.

## Tab Navigation

| Key | Action |
|-----|--------|
| `←` / `h` | Move to previous tab |
| `→` / `l` | Move to next tab |

Both keys are handled by `TabBar.handleInput()` in [[src/components/TabBar.ts]]. The `h`/`l` bindings were added in commit `5b7812d` alongside the `j`/`k` scroll bindings.

## Scroll Navigation

| Key | Action |
|-----|--------|
| `↑` / `k` | Scroll up in table tabs |
| `↓` / `j` | Scroll down in table tabs |

The `Dashboard` component detects `j`/`k` keys in its `handleInput()` method and translates them to ANSI escape sequences (`\x1b[B` for down/j, `\x1b[A` for up/k) before dispatching to the active tab. This is necessary because child components (SortedTable, etc.) listen for raw arrow-key escape sequences.

## Range Selector

| Key | Action |
|-----|--------|
| `r` | Cycle time range: 1d → 7d → 30d → All → 1d |

## General

| Key | Action |
|-----|--------|
| `Esc` / `q` | Close the dashboard |

## Controls Hint

The footer bar shows the current key bindings: `Esc/q close  ←→/h·l tabs  r range  ↑↓/j·k scroll`. Updated in commit `5b7812d` to include the vim bindings.
