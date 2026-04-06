# MCP (Model Context Protocol) Integration

## Overview

Claude Code connects to external MCP servers that provide additional tools, resources, and prompts. It supports 8 transport types (from local stdio processes to remote WebSocket servers), wraps MCP tools into the internal `Tool` interface with qualified names (`mcp__server__tool`), and handles OAuth authentication with RFC 9728/8414 discovery.

MCP configuration comes from 7 scopes (local, user, project, dynamic, enterprise, Claude.ai, managed) with policy-level allowlist/denylist enforcement.

## Architecture

```
MCP Server Configs (.mcp.json, settings, policy)
  ↓
MCPConnectionManager
  ├─ connectToServer() per config
  │   ├─ stdio: spawn local process
  │   ├─ sse/http: HTTP connection
  │   ├─ ws: WebSocket
  │   ├─ sse-ide/ws-ide: IDE bridge
  │   └─ sdk: in-process
  │
  ├─ fetchToolsForClient() → Tool[] (wrapped)
  │   ├─ List tools via tools/list RPC
  │   ├─ Create wrapper with qualified name
  │   ├─ Apply annotations (readOnly, destructive)
  │   └─ Truncate descriptions (2048 char limit)
  │
  └─ fetchResourcesForClient() → Resources
      fetchCommandsForClient() → Prompts
```

**Tool integration:**
```
Built-in tools (sorted by name)
  + MCP tools (sorted by name, filtered by deny rules)
  → assembleToolPool() → deduplicate (built-in wins)
  → Sent to API as tool schemas
```

## Design Decisions

**Qualified naming.** MCP tools use `mcp__servername__toolname` format with aggressive character normalization (`[^a-zA-Z0-9_-]` → `_`). This ensures no collision with built-in tools and satisfies API's `^[a-zA-Z0-9_-]{1,64}$` pattern requirement.

**Built-in tools win.** When `assembleToolPool()` deduplicates, built-in tools take precedence over MCP tools with the same name. This prevents an MCP server from shadowing critical tools like `Bash` or `Read`.

**Session recovery.** MCP connections can go stale (server restart, network issues). When a `McpSessionExpiredError` occurs during tool execution, the client automatically reconnects and retries the call. This is invisible to the model.

**OAuth discovery chain.** For MCP servers requiring authentication, the client tries three discovery methods in order: RFC 9728 probe on the MCP server → RFC 8414 metadata on a configured URL → fallback. Non-standard error codes (Slack's `invalid_refresh_token`, `expired_refresh_token`) are normalized to standard `invalid_grant`.

**LRU tool caching.** `fetchToolsForClient()` uses an LRU cache (max 20 servers) to avoid repeated RPC calls. This is important because tool lists are needed on every API call for schema building.

**Policy enforcement at connection time.** Deny rules are checked before connections are established, not after. An enterprise denylist can prevent connections to specific servers by name, command, or URL pattern. Denylist takes precedence over allowlist.

**Error resilience via SDK transport gap bridging.** The MCP SDK's transport layer calls `onerror` on connection failures but does *not* call `onclose`, which Claude Code depends on for triggering reconnection. The implementation bridges this by tracking 3 consecutive terminal errors (`MAX_ERRORS_BEFORE_RECONNECT`) and manually triggering `close()` to reject pending tool calls and clear the memoization cache. A `hasTriggeredClose` guard prevents re-entry — `close()` aborts in-flight streams which may fire `onerror` again.

**Stdio process termination escalation.** For stdio servers, `StdioClientTransport.close()` only sends an abort signal. Many MCP servers (especially Docker containers) require explicit signal escalation: SIGINT (100ms wait) → SIGTERM (400ms wait) → SIGKILL, with process existence checks via `process.kill(pid, 0)`. A 600ms failsafe timeout keeps the CLI responsive, trading thorough graceful shutdown for user experience.

**HTTP session expiry detection.** The implementation detects HTTP 404 + JSON-RPC error code -32001 (MCP session not found) and calls `closeTransportAndRejectPending('session expired')`, which clears the connection memoization cache and forces a fresh session ID on reconnect. This is critical for remote HTTP/SSE transports where sessions are ephemeral.

**Four-cache invalidation on disconnect.** When `onclose` fires, four separate caches are invalidated: `connectToServer`, `fetchToolsForClient`, `fetchResourcesForClient`, `fetchCommandsForClient`. This prevents "reconnect fetch stale data" bugs — resources/tools cached before connection drop would persist otherwise.

**Resource tool deduplication.** Resource tools (`ListMcpResourcesTool`, `ReadMcpResourceTool`) are added only once per MCP client set, not per server. The implementation checks if any connected server already provides these tools before adding them, preventing duplicate "list resources" entries when multiple servers expose resources.

**Large output file persistence.** When `ENABLE_MCP_LARGE_OUTPUT_FILES` is set, large MCP outputs are persisted to disk with instructions for reading rather than truncated in-memory. Falls back to truncation if output contains images (persisting images as JSON defeats compression logic).

**Configurable connection timeout.** Connection timeout defaults to 30 seconds but is configurable via the `MCP_TIMEOUT` environment variable, used in `Promise.race()` timeout logic.

## Insights

- The tool execution wrapper handles three content types: text (passed through), binary (persisted to disk with MIME type), and structured (MCP `_meta` preserved). Large results are truncated with file references, same as built-in tools.
- Agent-specific MCP servers can be declared in agent frontmatter. The agent system initializes these at spawn time and cleans them up when the agent completes.
- IDE extensions (VS Code, JetBrains) can expose MCP servers via `sse-ide` or `ws-ide` transports. The IDE lockfile provides the port and auth token.
- `claudeai-proxy` transport rewrites URLs for Claude.ai's session-ingress, enabling cloud-hosted MCP connections.
- Description truncation at 2048 chars prevents a single MCP tool from consuming excessive context. Server instructions are similarly truncated, with original/truncated byte counts logged for diagnostics.
- Qualified naming (`mcp__serverName__toolName`) accommodates server names containing `__` by joining all parts after the server name prefix, though this creates edge-case ambiguity.

## Key Files

| File | Role |
|------|------|
| `src/services/mcp/client.ts` | Connection management, tool wrapping, error resilience (~3300 lines) |
| `src/services/mcp/config.ts` | Config loading, policy enforcement, deduplication |
| `src/services/mcp/types.ts` | Transport types, connection states |
| `src/services/mcp/auth.ts` | OAuth flow, XAA cross-app access, token refresh, session expiry |
| `src/services/mcp/mcpStringUtils.ts` | Qualified naming: `mcp__server__tool` |
| `src/services/mcp/normalization.ts` | Character normalization for API compatibility |
| `src/tools/MCPTool/MCPTool.ts` | Base MCP tool wrapper with UI rendering |
