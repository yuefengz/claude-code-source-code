# Agent & Multi-Agent Architecture

## Overview

Claude Code can spawn sub-agents to handle complex tasks in parallel. Each agent is an independent conversation loop (`query()`) with its own context, tools, and permissions. Agents can run synchronously (blocking the parent), asynchronously (background tasks), or in isolated environments (git worktrees, remote CCR).

The system supports three coordination patterns: simple delegation (one-off agents), coordinator mode (orchestrator + restricted workers), and swarm mode (peer teammates with mailbox communication).

## Architecture

```
Parent (REPL or Agent)
  │
  ▼
AgentTool.call()
  ├─ Resolve definition (built-in, custom .md, plugin, policy)
  ├─ Create isolated context (createSubagentContext)
  ├─ Initialize agent-specific MCP servers
  └─ Dispatch to execution path:
      │
      ├─ Sync: query() in parent's thread → block until done
      ├─ Async: registerAsyncAgent() → background task → notification
      ├─ Worktree: git worktree + async → isolated branch
      ├─ Remote: CCR environment → always async
      └─ Fork: cache-sharing clone → identical prompt prefix
```

**Coordinator Mode:**
```
Coordinator (full tools)
  ├─ Agent({ subagent_type: "worker" }) → Worker 1 (restricted tools)
  ├─ Agent({ ... }) → Worker 2
  └─ SendMessage({ to: "Worker 1" }) → continue existing worker
```

## Design Decisions

**Fail-closed state isolation.** By default, sub-agents can't mutate the parent's `AppState` — `setAppState` is a no-op. This prevents background agents from accidentally changing permission modes, UI state, or other shared state. The one exception: `setAppStateForTasks` always reaches root, so bash kill signals propagate correctly.

**Linked abort controllers.** Each agent gets a *child* abort controller linked to the parent's. If the parent aborts, all children abort too. But children can be independently aborted without affecting the parent. This enables clean cancellation hierarchies.

**Sidechain transcripts.** Every agent writes its messages to a sidechain transcript file. This enables resumption — `resumeAgent()` reconstructs the conversation from the stored transcript, cleaning up incomplete tool calls and orphaned thinking blocks. This is essential for background agents that may be continued via `SendMessage`.

**Cache-sharing forks.** Fork agents reconstruct the parent's system prompt at fork time, producing an identical prompt prefix. This maximizes prompt cache hits — the forked agent reuses the same cached tokens as the parent, avoiding redundant computation.

**Worker tool restrictions.** In coordinator mode, workers get a minimal tool set (Bash, Read, Edit, Write, Skill) and explicitly *cannot* spawn other workers or use coordination tools. This prevents recursive delegation loops and keeps the coordinator as the single point of control.

**AsyncLocalStorage for analytics isolation.** Multiple agents may run concurrently in the same Node.js process. `AsyncLocalStorage` provides per-async-chain context, so telemetry events correctly attribute to the right agent without threading context parameters through every function call.

## Insights

- Agent definitions are markdown files with YAML frontmatter. The frontmatter specifies model, tools, permissions, MCP servers, hooks, and memory settings. The markdown body becomes the agent's system prompt. This makes agents easily shareable and version-controllable.
- Resolution priority for agent types: `builtIn → plugin → user → project → flag → managed`. Later sources override earlier ones, so a project can customize a built-in agent's behavior.
- Background agents communicate with their parent via **task-notification** XML messages injected into the parent's conversation. The parent sees a structured summary (status, output, usage) rather than the agent's full transcript.
- Worktree agents persist their worktree path in metadata. On resume, the path's mtime is bumped to prevent a stale-worktree cleanup job from removing it mid-conversation.
- The `SendMessage` tool supports broadcast (`to: "*"`) and structured messages (shutdown requests, plan approvals) for coordinated multi-agent workflows.

## Key Files

| File | Role |
|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | Input schema, dispatch to execution paths |
| `src/tools/AgentTool/runAgent.ts` | Agent execution loop, context setup, hooks, skills |
| `src/tools/AgentTool/resumeAgent.ts` | Checkpoint-based resumption |
| `src/tools/AgentTool/loadAgentsDir.ts` | Definition loading from .md/.json files |
| `src/tools/AgentTool/agentToolUtils.ts` | Async lifecycle, progress tracking, result finalization |
| `src/coordinator/coordinatorMode.ts` | Coordinator system prompt, worker restrictions |
| `src/tools/SendMessageTool/` | Inter-agent messaging (mailbox, broadcast, structured) |
| `src/utils/forkedAgent.ts` | `createSubagentContext()`, state cloning |
| `src/utils/agentContext.ts` | AsyncLocalStorage-based analytics isolation |
| `src/tasks/LocalAgentTask/` | Background task state machine |
