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

## Insights

- The tool execution wrapper handles three content types: text (passed through), binary (persisted to disk with MIME type), and structured (MCP `_meta` preserved). Large results are truncated with file references, same as built-in tools.
- Agent-specific MCP servers can be declared in agent frontmatter. The agent system initializes these at spawn time and cleans them up when the agent completes.
- IDE extensions (VS Code, JetBrains) can expose MCP servers via `sse-ide` or `ws-ide` transports. The IDE lockfile provides the port and auth token.
- `claudeai-proxy` transport rewrites URLs for Claude.ai's session-ingress, enabling cloud-hosted MCP connections.
- Description truncation at 2048 chars prevents a single MCP tool from consuming excessive context.

## Key Files

| File | Role |
|------|------|
| `src/services/mcp/client.ts` | Connection management, tool wrapping (~3300 lines) |
| `src/services/mcp/config.ts` | Config loading, policy enforcement, deduplication |
| `src/services/mcp/types.ts` | Transport types, connection states |
| `src/services/mcp/auth.ts` | OAuth flow, XAA cross-app access, token refresh |
| `src/services/mcp/mcpStringUtils.ts` | Qualified naming: `mcp__server__tool` |
| `src/services/mcp/normalization.ts` | Character normalization for API compatibility |
| `src/tools/MCPTool/MCPTool.ts` | Base MCP tool wrapper with UI rendering |
