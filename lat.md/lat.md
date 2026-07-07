# lat.md

pi-atlas knowledge graph — architecture, design decisions, and test specs.

## Sections

- [[overview]] — TUI dashboard extension for pi; reads session logs and renders a six-tab interactive dashboard
- [[parser]] — JSONL session log parser; handles 10 entry types, tracks skills and languages
- [[cache]] — SHA-256-gated caching layer; avoids re-parsing session files on every /atlas invocation
- [[compute]] — pure `summarize()` function; filters, merges, and sorts DayAgg arrays into StatsSummary
- [[format]] — formatting utilities; costs, numbers, language names, model names, timestamps
- [[dashboard]] — top-level TUI component; owns the tab tree, keyboard routing, and overlay mode
- [[components]] — reusable TUI building blocks; BarChart, SortedTable, GridRow, and more
- [[tabs]] — six dashboard tab content components; Overview, Languages, Models, Projects, Skills, Usage
- [[navigation]] — keyboard shortcuts; arrow keys and vim-style h/j/k/l bindings
- [[last-commit]] — git commit hash tracker for the documentator subagent

---

_This directory is managed by lat.md. Run `lat --help` for commands._
