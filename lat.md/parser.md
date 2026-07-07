# Parser

The parser (`src/parser.ts`) reads a single `.jsonl` session file and emits an array of `DayAgg` objects, one per calendar day represented in the file.

## Entry Types

Each line in a `.jsonl` session file is a JSON object with a `type` field. The parser handles 10 entry types:

| Type | Handler | Tracked Fields |
|------|---------|----------------|
| `session` | `parseSession()` | project, cwd, timestamp → day grouping |
| `userMessage` | `parseUserMessage()` | userMsgCount, skill stack push, language counts |
| `assistantMessage` | `parseAssistantMessage()` | assistantTokens, skill stack pop, language counts |
| `toolResult` | `parseToolResult()` | toolResults, language counts for written content |
| `modelChange` | `parseModelChange()` | modelChanges counter |
| `thinkingLevelChange` | `parseThinkingLevelChange()` | thinkingLevelCount per level |
| `compaction` | `parseCompaction()` | compactionCount, compactedTokens |
| `branchSummary` | skipped | — |
| `custom` | skipped | — |
| `customMessage` | skipped | — |
| `label` | skipped | — |
| `sessionInfo` | skipped | — |

## Language Counting

Language is determined by file extension in the `toolName` field (for tool results) or by explicit `language` fields in assistant messages. The parser splits `newText`/`content` by newline to count lines, not characters.

## Skill Tracking

Skills are tracked via two mechanisms:

1. **Explicit**: user messages containing `<skill name="X">` XML tags, detected by regex in `parseUserMessage()`.
2. **Implicit**: assistant tool calls to read files matching `/SKILL.md`, detected in `parseAssistantMessage()`.

Skills use a stack model — when a skill is activated, it is pushed onto the active stack; the stack is cleared at each new user message. Assistant message costs are attributed to all skills currently on the stack.

## Error Handling

Malformed JSON lines are counted in `totalCorrupt` and skipped silently. The parser does not throw — `console.warn` logs the first 200 characters of failed tool arguments.
