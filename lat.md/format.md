# Format Utilities

The `format.ts` module provides pure formatting functions used throughout the tabs and components.

## Language Mapping

`formatLanguage(lang: string): string` maps lowercase file extension strings to display names. Covers 70+ languages (TypeScript, JavaScript, Python, Rust, Go, HTML, CSS, JSON, YAML, Markdown, SQL, Shell, etc.).

## Number Formatting

`formatNumber(n: number): string` formats with thousands separators (e.g., `1,234`). Handles zero, negative, and decimal inputs.

## Cost Formatting

`formatCost(cents: number): string` converts integer cents to dollar strings (e.g., `formatCost(1234)` → `$12.34`). Zero returns `$0.00`. Values under 10 cents show `$0.0X`.

## Model Name Formatting

`formatModelName(name: string): string` strips provider prefixes (e.g., `anthropic/` from `anthropic/claude-3-5-sonnet`) and returns the bare model ID. Unknown models are returned unchanged.

## Timestamp Formatting

`formatTimestamp(iso: string): string` converts ISO date strings to `MMM DD HH:mm` format (e.g., `Jul 06 14:30`).

## Relative Time

`formatRelativeTime(isoDate: string): string` computes the elapsed time from an ISO timestamp to now, expressed in minutes, hours, or days (e.g., `Updated 5 min ago`, `Updated 2 hr ago`).
