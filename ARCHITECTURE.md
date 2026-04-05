# Claude Code Architecture

> Claude Code v2.1.88 — Anthropic's CLI agent for software engineering tasks.
> ~1,884 TypeScript files, ~512K lines of code.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Entrypoints                          │
│  cli.tsx → Fast-path routing (--version, --daemon, etc.)│
│         → main.tsx → Commander.js parsing               │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                   UI Layer (React/Ink)                   │
│  REPL.tsx → PromptInput → Response rendering            │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                 Query Engine (Core Loop)                 │
│  query.ts → API calls, tool dispatch, context mgmt      │
└────────┬─────────────┬──────────────────┬───────────────┘
         │             │                  │
┌────────▼───┐ ┌───────▼────────┐ ┌──────▼──────────────┐
│  Tool      │ │  API Service   │ │  Context & State     │
│  System    │ │  Streaming     │ │  Compaction          │
│  45+ tools │ │  Multi-backend │ │  Session history     │
│  MCP tools │ │  Retry logic   │ │  Memory system       │
└────────────┘ └────────────────┘ └─────────────────────┘
```

---

## Component Details

Each component has a detailed deep-dive document linked below.

### [Entrypoints & CLI Bootstrap](docs/architecture/entrypoints-and-cli.md)

How Claude Code starts up, parses commands, and launches the interactive REPL. Covers the fast-path routing in `cli.tsx` (13 branches for common flags before full CLI loads), Commander.js command definition with 100+ options, global initialization sequence in `init.ts`, and the REPL React component mounting.

**Key files:** `src/entrypoints/cli.tsx`, `src/main.tsx`, `src/entrypoints/init.ts`, `src/screens/REPL.tsx`

---

### [Query Engine & Conversation Loop](docs/architecture/query-engine.md)

The core `while(true)` async generator that drives conversations. Covers the state machine with 7 continue/recovery sites, API request construction with task budgets and thinking config, streaming consumption, tool result feedback loops, multi-backend client support (Direct API, Bedrock, Vertex, Azure), retry strategy with error categorization, and both client-side and API-native context compression.

**Key files:** `src/query.ts`, `src/QueryEngine.ts`, `src/services/api/claude.ts`, `src/services/compact/`

---

### [Tool System](docs/architecture/tool-system.md)

How tools are defined, registered, and executed. Covers the `Tool<Input, Output, Progress>` interface with Zod schemas, the `buildTool()` factory with fail-closed defaults, tool registry with feature gating, the full execution pipeline (validation → hooks → permissions → execution → result processing), concurrency partitioning (up to 10 parallel read-only tools), and the `StreamingToolExecutor` that dispatches tools as they stream in.

**Key files:** `src/Tool.ts`, `src/tools.ts`, `src/services/tools/toolOrchestration.ts`, `src/tools/*/`

---

### [Agent & Multi-Agent Architecture](docs/architecture/agent-architecture.md)

Agent spawning, execution isolation, and multi-agent coordination. Covers 6 execution paths (sync, async, worktree, remote, fork, teammate), subagent context creation with state isolation via `AsyncLocalStorage`, agent definition loading from markdown frontmatter, coordinator mode with worker tool restrictions, inter-agent communication via `SendMessage`, and background agent lifecycle with task-notification.

**Key files:** `src/tools/AgentTool/`, `src/coordinator/coordinatorMode.ts`, `src/utils/swarm/`, `src/tasks/`

---

### [Permission & Security Model](docs/architecture/permissions-and-security.md)

Layered defense system for tool execution. Covers 6 permission modes (default through bypass), rule-based matching with `ToolName(content)` syntax, the main permission algorithm with bypass-immune safety checks, the two-stage ML classifier for auto mode, denial tracking with fallback to prompting (3 consecutive or 20 total), sandbox boundaries with filesystem/network isolation, and bash-specific AST-based permission analysis.

**Key files:** `src/utils/permissions/`, `src/types/permissions.ts`, `src/utils/sandbox/`

---

### [Configuration, Settings & Hooks](docs/architecture/configuration-and-hooks.md)

Settings loading from 6 sources with priority merging, and the hooks system with 24 event types. Covers the settings hierarchy (plugin → user → project → local → flag → policy), policy settings with "first source wins" resolution, 4 hook command types (shell, prompt, agent, HTTP), hook matching with exact/pipe/regex patterns, async hook detection, exit code semantics (0=success, 2=blocking), and context assembly from CLAUDE.md files.

**Key files:** `src/utils/settings/`, `src/utils/hooks.ts`, `src/schemas/hooks.ts`, `src/context.ts`

---

### [MCP Integration](docs/architecture/mcp-integration.md)

Model Context Protocol server connections, tool wrapping, and authentication. Covers 8 transport types (stdio, SSE, HTTP, WebSocket, IDE bridges, SDK, proxy), tool wrapping with qualified `mcp__server__tool` naming, OAuth with RFC 9728/8414 discovery and XAA cross-app access, configuration across 7 scopes (local through managed), policy enforcement with allowlist/denylist, and resource fetching.

**Key files:** `src/services/mcp/client.ts`, `src/services/mcp/config.ts`, `src/services/mcp/auth.ts`

---

### [Memory System](docs/architecture/memory-system.md)

Persistent file-based memory across conversations. Covers 4 memory types (user, feedback, project, reference), two-step save process (write file + update index), path resolution with security validation, MEMORY.md truncation (200 lines / 25KB), git root sharing for worktrees, assistant mode daily logs, and team memory support.

**Key files:** `src/memdir/paths.ts`, `src/memdir/memdir.ts`, `src/memdir/memoryTypes.ts`

---

### [IDE Integration & Terminal UI](docs/architecture/ide-and-ui.md)

IDE detection/connection and the custom terminal rendering engine. Covers support for VS Code, Cursor, Windsurf, and 15 JetBrains IDEs via process detection and lockfiles, the custom Ink terminal framework (~13K lines) with Yoga flexbox layout, React Fiber reconciliation, diff-based terminal updates, cell pooling, text selection, and 146 React components.

**Key files:** `src/utils/ide.ts`, `src/ink/`, `src/components/`

---

## Key User Journeys

### 1. Interactive Conversation

```
claude → cli.tsx → main.tsx → REPL.tsx → PromptInput
→ query() loop → API streaming → tool execution → response display
```

### 2. Tool Execution

```
API returns tool_use → findToolByName() → validateInput()
→ permission check (rules → hooks → classifier → dialog)
→ tool.call() → result → next API turn
```

### 3. Sub-Agent Spawning

```
Agent({ prompt, run_in_background }) → resolve definition
→ createSubagentContext() → runAgent() → query() independently
→ task-notification to parent on completion
```

### 4. Context Compression

```
Token count > 180K → API microcompact (clear tool results/uses)
Token count still high → client-side compaction (LLM summarization)
→ Continue with reduced context
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | TypeScript (ES2022, strict in parts) |
| Runtime | Node.js 18+ |
| UI | React + custom Ink terminal framework |
| CLI | Commander.js |
| Build | esbuild / Bun |
| API Client | `@anthropic-ai/sdk` |
| Layout | Yoga (flexbox in terminal) |
| Search | ripgrep (via GrepTool) |

---

## Project Statistics

| Metric | Value |
|--------|-------|
| Source Files | 1,884 TypeScript |
| Lines of Code | ~512,664 |
| Built-in Tools | 45+ |
| CLI Commands | 100+ |
| UI Components | 146 React/Ink |
| Custom Hooks | 87 |
| Utility Modules | 331 |
| Hook Events | 24 types |
| MCP Transports | 8 types |
| Permission Modes | 6 |
