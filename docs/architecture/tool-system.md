# Tool System

## Overview

Claude Code has 45+ built-in tools (file operations, shell commands, search, web access, agents, etc.) plus dynamically loaded MCP tools. Every tool implements a common `Tool<Input, Output, Progress>` interface with Zod schema validation, permission checks, and progress reporting.

Tools are executed through a pipeline: validation → hooks → permissions → execution → result processing. Multiple read-only tools can run in parallel (up to 10), while mutating tools run sequentially. A streaming executor dispatches tools as they arrive from the API, overlapping network and computation.

## Architecture

```
Tool Definition (Tool.ts)
  │
  ▼
Tool Registry (tools.ts)
  ├─ getAllBaseTools() — 60+ tools with feature gating
  ├─ getTools() — filtered by mode (simple/full/coordinator)
  └─ assembleToolPool() — merge built-in + MCP, sorted for cache stability
  │
  ▼
Tool Execution (toolExecution.ts)
  ├─ validateInput() — Zod schema
  ├─ backfillObservableInput() — normalize for hooks
  ├─ runPreToolUseHooks() — may override permission/input
  ├─ resolvePermission() — hooks + classifier + dialog
  ├─ tool.call() — actual execution with progress
  ├─ processToolResultBlock() — persist large results
  └─ runPostToolUseHooks()
  │
  ▼
Orchestration (toolOrchestration.ts / StreamingToolExecutor.ts)
  ├─ Partition into concurrent/sequential batches
  ├─ Execute batches with concurrency limit
  └─ Yield results in order
```

## Design Decisions

**Fail-closed defaults.** The `buildTool()` factory sets `isConcurrencySafe: false` and `isReadOnly: false` by default. New tools are conservatively treated as mutating and sequential unless explicitly marked otherwise. This prevents accidental parallel execution of tools that could interfere with each other.

**Streaming execution.** `StreamingToolExecutor` starts executing tools as `tool_use` blocks arrive from the API stream, rather than waiting for the complete response. This is critical for multi-tool turns — while the API is still generating the second tool call, the first is already running.

**Bash-only error cascade.** When a Bash command fails, sibling tools are aborted (via a shared abort controller). But when a file read or grep fails, siblings continue. The rationale: Bash failures often indicate environment problems that would affect other tools, while search failures are typically isolated.

**Sorted tool pool.** Tools are sorted by name before being sent to the API. This ensures the tool description prefix in the system prompt stays stable across turns, maximizing prompt cache hit rates.

**Pagination by default.** Grep and Glob cap results at 250 entries by default (`head_limit`). This prevents a single search from consuming the entire context window. Models can pass `head_limit=0` to disable the cap when they genuinely need all results.

**Context modifiers.** Tools can return a `contextModifier` closure that mutates the `ToolUseContext` after execution. For concurrent batches, these are queued and applied after the batch completes (to avoid races). For sequential execution, they apply immediately.

## Insights

- The tool registry supports **deferred loading** via `ToolSearchTool` — not all 60+ tool schemas are sent to the API on every request. Less-used tools are loaded on demand, reducing token count.
- **Permission matchers** are tool-specific. Bash parses commands to AST and matches each subcommand independently (`Bash(git *)` matches `git push` but not `gitmoji`). File tools expand paths to absolute form before matching.
- Large tool results (>30K chars for Bash) are persisted to disk as files, with only a preview sent to the API. This prevents a single command output from dominating the context.
- The `isSearchOrReadCommand()` method lets the UI collapse search results — Grep, Glob, and read-only Bash commands get a compact display treatment.

## Key Files

| File | Role |
|------|------|
| `src/Tool.ts` | `Tool` interface, `buildTool()` factory, `ToolUseContext` type |
| `src/tools.ts` | Registry: `getAllBaseTools()`, `getTools()`, `assembleToolPool()` |
| `src/services/tools/toolOrchestration.ts` | Batch partitioning, concurrent/sequential execution |
| `src/services/tools/toolExecution.ts` | Full pipeline: validation → hooks → permissions → execution |
| `src/services/tools/StreamingToolExecutor.ts` | Streaming dispatch with abort cascade |
| `src/tools/BashTool/` | Shell execution with AST analysis and sandbox |
| `src/tools/FileEditTool/` | File mutation with mtime conflict detection |
| `src/tools/GrepTool/` | ripgrep wrapper with pagination |
