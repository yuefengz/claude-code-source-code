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

## Insights

- The `preAction` hook in Commander.js runs before *every* subcommand, including `--help`. This is where global init happens, ensuring consistent state regardless of which command runs.
- `setup()` calls `setCwd()` before anything else — all subsequent file operations, command loading, and agent definitions resolve relative to this path. Worktree mode changes this to an isolated directory.
- The hooks configuration is snapshotted at startup via `captureHooksConfigSnapshot()` to prevent a malicious project from modifying hooks after the trust dialog has been accepted.
- Non-interactive mode (`-p`/`--print`) skips the trust dialog entirely, treating pipe input as implicit trust.

## Key Files

| File | Role |
|------|------|
| `src/entrypoints/cli.tsx` | Bootstrap with 13 fast-path branches |
| `src/main.tsx` | Commander.js definition, 100+ options, default action handler |
| `src/entrypoints/init.ts` | Global init (memoized): config, network, telemetry |
| `src/setup.ts` | Workspace setup: setCwd, hooks, worktree |
| `src/replLauncher.tsx` | Thin wrapper: mounts `<App><REPL /></App>` |
| `src/screens/REPL.tsx` | Main interactive component (~5000 lines) |
