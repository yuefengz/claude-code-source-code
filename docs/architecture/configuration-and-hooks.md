# Configuration, Settings & Hooks

## Overview

Claude Code loads settings from 6 sources with priority merging (plugin < user < project < local < flag < policy). The hooks system provides 24 event types for extending behavior — from pre-tool-use validation to session lifecycle events — with 4 execution backends (shell, LLM prompt, agent, HTTP webhook).

Context for the system prompt is assembled from multiple sources: CLAUDE.md files (walked up from CWD), git status, memory, and settings.

## Architecture

**Settings loading:**
```
Plugin settings (lowest)
  ↓ merge
User settings (~/.claude/settings.json)
  ↓ merge
Project settings (.claude/settings.json)
  ↓ merge
Local settings (.claude/settings.local.json)
  ↓ merge
Flag settings (--settings CLI / SDK inline)
  ↓ merge
Policy settings (managed-settings.json, MDM, remote API)  ← highest
```

**Hook execution:**
```
Event fires (e.g., PreToolUse)
  ↓
Match hooks by event + optional matcher pattern
  ↓
For each match:
  ├─ Evaluate "if" condition (permission rule syntax)
  ├─ Dispatch: command | prompt | agent | http | callback
  ├─ Detect async (config flag or {"async":true} output)
  └─ Process result:
       ├─ Exit 0 → success
       ├─ Exit 2 → blocking error (shown to model)
       └─ Other → non-blocking (stderr to user)
```

## Design Decisions

**Later sources override, arrays deduplicate.** The merging strategy uses lodash `mergeWith` — scalar values from higher-priority sources win, while arrays are concatenated and deduplicated. This means a project can *add* permission rules without replacing the user's rules. Setting a value to `undefined` signals deletion from record fields.

**Policy first-source-wins.** Within policy settings, there's a sub-hierarchy: remote API > MDM (admin-only) > file-based > registry (user-writable). The *first* policy source that provides a value wins, preventing user-writable registries from overriding admin policy.

**Drop-in file pattern.** `managed-settings.d/*.json` files are loaded alphabetically and merged on top of the base `managed-settings.json`. This follows the systemd/sudoers convention, making it easy for deployment tools to add policy fragments without modifying a single file.

**Trust-gated hooks.** All hooks are skipped if the trust dialog hasn't been accepted yet. This is defense-in-depth: a malicious project can't execute hooks just because a user opened a terminal in the project directory.

**Exit code semantics.** Hook exit codes have specific meaning: 0 = success (output context-dependent), 2 = blocking error that's shown to the model (the model must address it), anything else = non-blocking (stderr shown to user but not to model). This gives hooks fine-grained control over whether issues are "soft" (informational) or "hard" (must be fixed).

**Async hook detection.** Hooks can declare async behavior in two ways: via config (`async: true`) or dynamically by emitting `{"async":true}` as their first stdout line. The dynamic approach lets hooks decide at runtime whether they need to go async. `asyncRewake` hooks run in the background and wake the model when they exit with code 2.

**Hooks snapshot at startup.** The hooks configuration is captured at startup time via `captureHooksConfigSnapshot()`. This prevents a pre-tool-use hook from modifying the hooks config to escalate its own privileges.

## Insights

- CLAUDE.md files are discovered by walking up from CWD — a monorepo root's CLAUDE.md applies to all subdirectories. Content is cached for the conversation duration and also provided to the auto-mode classifier as context.
- Git status is a snapshot taken at conversation start and not updated during the conversation. The system prompt explicitly notes this so the model knows to run `git status` for current state.
- Hook matchers support three patterns: exact match (`Write`), pipe-separated (`Write|Edit`), and regex (`^Write.*`). Legacy tool names are automatically included in matching.
- The `allowManagedHooksOnly` policy setting prevents user/project hooks from running — only admin-deployed hooks execute. This is the enterprise lockdown path.
- Settings caching uses two levels: per-source parsed cache and session-level merged cache, both invalidated by `resetSettingsCache()` after writes.

## Key Files

| File | Role |
|------|------|
| `src/utils/settings/settings.ts` | Loading pipeline, merging, caching |
| `src/utils/settings/types.ts` | Full settings schema |
| `src/schemas/hooks.ts` | Hook command type schemas (command/prompt/agent/http) |
| `src/types/hooks.ts` | 24 event types, result types |
| `src/utils/hooks.ts` | Hook execution engine (~3700 lines) |
| `src/utils/hooks/hooksSettings.ts` | Hook matching, source management |
| `src/context.ts` | System prompt context assembly (CLAUDE.md, git, memory) |
