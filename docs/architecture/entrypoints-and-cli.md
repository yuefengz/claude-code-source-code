# Entrypoints & CLI Bootstrap

## Overview

Claude Code's startup is optimized for speed through aggressive fast-path routing. Before the full CLI (~785KB of Commander.js setup) is ever imported, `cli.tsx` checks 13 common flags and routes them to lightweight handlers. This means `claude --version` returns instantly without loading React, tools, or the API client.

The full startup path goes through: bootstrap (`cli.tsx`) → global init (`init.ts`) → workspace setup (`setup.ts`) → trust dialog → REPL mount (`REPL.tsx`).

## Architecture

```
cli.tsx
  ├─ --version         → instant return (zero imports)
  ├─ --daemon-worker   → lean worker process
  ├─ bridge/remote     → bridgeMain()
  ├─ ps/logs/attach    → background session handlers
  ├─ --tmux+worktree   → exec into tmux before full load
  └─ (normal)          → import main.tsx
       ↓
main.tsx (Commander.js, 100+ options)
  ↓ preAction hook
  ├─ init() — memoized global bootstrap
  ├─ initSinks() — analytics
  ├─ runMigrations()
  └─ loadRemoteManagedSettings()
  ↓
setup() — setCwd, hooks snapshot, worktree creation
  ↓
showSetupScreens() — trust dialog (interactive only)
  ↓
launchRepl() → REPL.tsx (React/Ink component)
  ├─ Merges tools (built-in + MCP + plugins)
  ├─ Merges commands (built-in + skills + plugins)
  └─ Enters query() loop on user input
```

## Design Decisions

**Fast-path routing before imports.** The CLI checks flags like `--version`, `--daemon-worker`, and `--bridge` *before* importing the full application. This avoids loading ~400KB of OpenTelemetry, React, and the tool registry for simple operations. Each fast-path is also feature-gated via `feature()` for dead-code elimination in external builds.

**Parallel prefetching at module load time.** Before any `await` in the startup path, two fire-and-forget operations launch: MDM policy reads (macOS `plutil` / Windows registry) and keychain credential reads. These run concurrently with module loading, saving ~65ms on macOS.

**Memoized init.** `init()` uses lodash `memoize()` so it runs exactly once per process, even if called from multiple code paths. It handles config validation, graceful shutdown setup, network preconnection, and telemetry — all before the trust dialog appears.

**Trust before telemetry.** Telemetry is only initialized *after* the user accepts the trust dialog. This ensures no data is collected until explicit consent, while still allowing the heavy OpenTelemetry import (~400KB) to be deferred.

**REPL as React component.** The main interactive loop is a React component (`REPL.tsx`, ~5000 lines) using a custom Ink terminal framework. This enables declarative UI composition — merging tools, commands, and MCP clients via hooks like `useMergedTools()` — rather than imperative terminal manipulation.

**CA certs before TLS.** Config validation applies CA certificates *before* the trust dialog and any TLS handshakes. This prevents Bun's BoringSSL from caching a TLS session without the custom CA, which would cause all subsequent API calls to fail in corporate proxy environments.

**preAction sequencing.** The Commander.js `preAction` hook (not a Claude Code hook) orchestrates startup in a specific order: await MDM/keychain prefetch completion → memoized `init()` → attach analytics sinks → run migrations → fire-and-forget remote settings load. Analytics sinks attach here rather than in `setup()` because subcommands that skip `setup()` would silently drop events.

**Lazy telemetry initialization.** Telemetry init waits for remote settings only for eligible users; otherwise it initializes eagerly. This prevents deadlock if `loadRemoteManagedSettings()` never completes (e.g., in Agent SDK tests where no remote server is available).

**Fire-and-forget prefetches.** `init()` launches ~5 non-awaited prefetches (OAuth, JetBrains detection, GitHub status, remote settings, API preconnect) that populate caches before they're needed. The API preconnect reuses the global connection pool only when no proxy or mTLS is configured — otherwise it would warm a connection the actual requests can't use.

**Versioned migrations.** Migrations are declarative and guarded by version number — `runMigrations()` checks `getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION` and only runs if stale. The version is saved atomically with a re-check to prevent races when multiple sessions start simultaneously.

**Workspace setup ordering.** `setup()` establishes cwd *before* capturing the hooks snapshot (hooks are loaded from the correct directory). For worktree mode, setup resolves to the canonical main repo (handles nested worktrees), switches cwd, creates the worktree, then re-reads settings and re-captures hooks from the worktree directory. A UDS messaging server for swarm teammates is started and awaited before hooks snapshot so `$CLAUDE_CODE_MESSAGING_SOCKET` is available in hook environments.

**VCS-agnostic worktrees.** Worktree creation delegates to a `WorktreeCreate` hook for non-git repositories, enabling support for other version control systems without core changes.

**Lean daemon workers.** Daemon worker subprocesses skip `enableConfigs` and `initSinks` at the startup layer — they lazily initialize what they need inside their `run()` function. This supports fast worker spawning where the supervisor manages lifecycle. Bridge mode, by contrast, requires auth validation *before* GrowthBook gate evaluation to avoid stale cached gate values on cold start.

**REPL conditional imports.** `REPL.tsx` uses feature-gated conditional `require()` at the component level (not top-level) for ant-only features (voice, frustration detection, coordinator mode). This enables build-time dead code elimination in external builds — features are gated at both compile time (`bun:bundle`) and runtime (GrowthBook).

## Insights

- The `preAction` hook in Commander.js runs before *every* subcommand, including `--help`. This is where global init happens, ensuring consistent state regardless of which command runs.
- `setup()` calls `setCwd()` before anything else — all subsequent file operations, command loading, and agent definitions resolve relative to this path. Worktree mode changes this to an isolated directory.
- The hooks configuration is snapshotted at startup via `captureHooksConfigSnapshot()` to prevent a malicious project from modifying hooks after the trust dialog has been accepted.
- Non-interactive mode (`-p`/`--print`) skips the trust dialog entirely, treating pipe input as implicit trust.
- The 13 fast-path branches include: `--version`, `--dump-system-prompt`, `--claude-in-chrome-mcp`, `--chrome-native-host`, `--computer-use-mcp`, `--daemon-worker`, remote/bridge/sync, daemon supervisor, `ps|logs|attach|kill`, `--worktree --tmux` exec, and help display. Each is feature-gated via `feature()` for dead-code elimination in external builds.
- The `--tmux+worktree` fast path execs into tmux *before* Commander loads, avoiding the full CLI setup cost entirely.

## Key Files

| File | Role |
|------|------|
| `src/entrypoints/cli.tsx` | Bootstrap with 13 fast-path branches |
| `src/main.tsx` | Commander.js definition, 100+ options, preAction sequencing, migrations |
| `src/entrypoints/init.ts` | Global init (memoized): config, CA certs, network, telemetry, prefetches |
| `src/setup.ts` | Workspace setup: setCwd, hooks, worktree, UDS messaging server |
| `src/replLauncher.tsx` | Thin wrapper: mounts `<App><REPL /></App>` |
| `src/screens/REPL.tsx` | Main interactive component (~5000 lines), conditional feature imports |
