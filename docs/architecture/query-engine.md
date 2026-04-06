# Query Engine & Conversation Loop

## Overview

The query engine is the heart of Claude Code — an async generator (`queryLoop()`) that iteratively calls the Claude API, executes tools, manages context, and recovers from errors. It runs as a `while(true)` loop where each iteration makes one API call, processes any tool calls in the response, and decides whether to continue or terminate.

The engine supports multiple API backends (Direct, Bedrock, Vertex, Azure), automatic retry with exponential backoff, and two layers of context compression when conversations grow long.

## Architecture

```
REPL / SDK caller
  ↓
QueryEngine.submitMessage()
  ↓
query() async generator
  ↓
queryLoop() — while(true):
  │
  ├─ 1. Context management
  │     ├─ Proactive autocompact (if tokens > threshold)
  │     └─ API microcompact edits (server-side)
  │
  ├─ 2. API call (queryModel + withRetry)
  │     ├─ Build params: model, tools, thinking, budget, speed
  │     ├─ Stream response: text, tool_use, thinking blocks
  │     └─ Execute tools as they stream in
  │
  ├─ 3. Decision: continue or terminate?
  │     ├─ Tool results pending → loop (feed results back)
  │     ├─ Prompt-too-long error → compact and retry
  │     ├─ Output limit hit → escalate (8K→64K) or auto-continue
  │     ├─ Stop hook blocks → inject error and retry
  │     └─ No more work → return terminal result
  │
  └─ Yield events to caller (messages, stream deltas, boundaries)
```

The loop carries a `State` object across iterations containing messages, tool context, compaction tracking, output token recovery count, and turn count. There are 7 distinct "continue" sites where the loop decides to iterate again rather than terminate.

## Design Decisions

**Async generator pattern.** The query loop is an `async function*` that yields events as they occur. This lets callers (REPL, SDK, agents) consume results incrementally without buffering entire responses. It also naturally handles the iterative tool-call-and-continue pattern.

**Streaming tool execution.** Tools aren't executed after the full API response arrives — they begin executing as `tool_use` blocks stream in via `StreamingToolExecutor`. This overlaps API streaming with tool execution, reducing end-to-end latency for multi-tool responses.

**Two-layer context compression.** When conversations grow long:
1. **API-native microcompaction** — sends `context_management.edits` instructing the server to clear old tool results/uses (trigger: 180K input tokens, target: keep last 40K)
2. **Client-side compaction** — uses an LLM call to summarize old messages, replacing them with compressed summaries

The API layer is cheaper and preserves more context; the client layer is a fallback when API-native isn't sufficient.

**Task budget pacing.** The API receives `task_budget: { total, remaining }` so the model can pace itself across an agentic loop. The `remaining` field is decremented across compaction boundaries, ensuring the model doesn't "forget" it has already used most of its budget.

**Auto-continue with diminishing returns detection.** The +500K token budget feature lets conversations auto-continue past normal limits. But after 3+ continuations, if output delta drops below 500 tokens for two consecutive checks, the engine stops — preventing infinite loops where the model produces negligible output.

**Multi-backend abstraction.** `getAnthropicClient()` routes to the correct SDK class based on environment variables. All backends share the same streaming interface, so the query loop doesn't know which provider it's talking to.

**Seven continue sites.** The `while(true)` loop has 7 distinct points where it decides to iterate rather than terminate:
1. **collapse_drain_retry** — context collapse recovery after 413/media errors
2. **reactive_compact_retry** — reactive compaction for prompt-too-long recovery
3. **max_output_tokens_escalate** — single clean escalation from 8K→64K before multi-turn recovery
4. **max_output_tokens_recovery** — multi-turn recovery loop (max 3 attempts)
5. **stop_hook_blocking** — stop hook injected blocking errors trigger retry
6. **token_budget_continuation** — auto-continue when <90% of budget consumed
7. **next_turn** — tool execution loop at turn end

A critical detail: `hasAttemptedReactiveCompact` is preserved across `stop_hook_blocking` to prevent infinite loops (CC-1180), but reset on `next_turn`.

**Output limit escalation.** Default max output is 32K but capped to 8K (`CAPPED_DEFAULT_MAX_TOKENS`) for slot-reservation optimization — BQ p99 analysis shows <1% of requests hit this limit. Those get one clean retry escalating to 64K (gated by `tengu_otk_slot_v1`), then fall back to a multi-turn recovery loop. This avoids expensive recompaction when a single retry suffices.

**Withheld errors for SDK callers.** `max_output_tokens` errors are withheld from streaming (`isWithheldMaxOutputTokens`) until recovery is exhausted. This prevents SDK callers (cowork, desktop) from terminating mid-recovery when the engine may still succeed.

**Fast mode cooldown mechanics.** The beta header is latched session-stable (cache-safe), but the `speed='fast'` parameter stays dynamic — cooldown suppresses actual fast mode without changing the cache key. Cooldown triggers on `rate_limit` or `overloaded` errors, with a `resetAt` timestamp. If the org disables fast mode entirely, the user's setting is permanently cleared (distinct from temporary cooldown).

**API microcompact: thinking preservation.** When thinking is enabled and not redacted, all thinking blocks are preserved *unless* `clearAllThinking` fires (>1h idle, cache miss) — then only the last turn's thinking is kept (minimum 1, the required minimum). This balances context savings with model continuity.

**Tool clearing selectivity.** API-native tool clearing distinguishes read ops from write ops: shell, glob, grep, file-read, web-fetch, and web-search results are clearable, but file-edit, file-write, and notebook-edit are preserved. Write operations contain state that the model may need to reference.

**Task budget carryover across compaction.** Consumed tokens before compaction are subtracted from the remaining budget, preventing over-usage when a compacted conversation resumes with a "fresh" token count.

## Insights

- The retry strategy distinguishes between transient errors (429/529 → backoff) and auth errors (401 → refresh credentials and get a fresh client). Fast mode adds a wrinkle: short `retry-after` headers keep fast mode active, but long delays trigger a permanent cooldown.
- Prompt caching uses `cache_control: { type: 'ephemeral', ttl: '1h' }` markers. Beta headers like `afkHeaderLatched` use sticky-on latches — once activated in a session, they stay on for stability.
- The stream idle watchdog aborts connections that stall for 90+ seconds with no chunks, preventing hung connections from blocking the conversation.
- `paramsFromContext()` is a closure that rebuilds API params on each retry attempt, allowing dynamic adjustments (like model fallback or thinking config changes) between retries.
- The `clear_at_least` parameter in API microcompact computes the minimum tokens to drop to guarantee hitting the target, rather than relying on the server to guess an appropriate amount.
- Diminishing returns detection (≥3 continuations + last 2 deltas <500 tokens) prevents infinite loops where the model produces negligible output per continuation.

## Key Files

| File | Role |
|------|------|
| `src/query.ts` | Main `queryLoop()` state machine (~1300 lines), 7 continue sites |
| `src/QueryEngine.ts` | SDK-facing wrapper, message management |
| `src/services/api/claude.ts` | API streaming client, request construction (~3000 lines) |
| `src/services/api/client.ts` | Multi-backend client factory (Direct, Bedrock, Vertex, Azure) |
| `src/services/api/withRetry.ts` | Retry strategy, error categorization |
| `src/services/compact/compact.ts` | Client-side compaction via LLM summarization |
| `src/services/compact/apiMicrocompact.ts` | API-native context editing, thinking preservation, tool clearing |
| `src/query/tokenBudget.ts` | Auto-continue budget tracking & diminishing returns |
| `src/utils/context.ts` | Output token defaults, capping, escalation thresholds |
| `src/utils/fastMode.ts` | Fast mode state, cooldown mechanics, org status |
