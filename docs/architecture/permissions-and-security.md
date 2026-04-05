# Permission & Security Model

## Overview

Claude Code uses a layered defense system to control tool execution. Every tool call passes through a pipeline: rule-based checks → mode checks → ML classifier (auto mode) → interactive prompt. Some checks are "bypass-immune" — they fire even in `bypassPermissions` mode, protecting sensitive paths like `.git/` and `.claude/`.

The system supports 6 permission modes (from fully interactive to fully autonomous), pattern-based rules (`Bash(git *)`, `Write(/etc/*)`), and a two-stage ML classifier that evaluates tool safety in auto mode.

## Architecture

```
Tool call request
  │
  ▼
Step 1: Rule-based checks (BYPASS-IMMUNE)
  ├─ Deny rules → block
  ├─ Ask rules → prompt (even in bypass mode)
  ├─ Tool's own checkPermissions() → may deny/ask
  └─ Safety checks (.git/, .claude/, shell configs) → always prompt
  │
  ▼
Step 2: Mode checks
  ├─ bypassPermissions → allow
  └─ Always-allow rules → allow
  │
  ▼
Step 3: Auto mode (if enabled)
  ├─ Allowlisted tools (Read, Grep, etc.) → auto-approve
  ├─ Two-stage ML classifier → approve or block
  └─ Denial limits (3 consecutive / 20 total) → fallback to prompt
  │
  ▼
Step 4: Interactive prompt or headless decision
  ├─ Interactive: PermissionRequest dialog with Allow/Deny/Always options
  ├─ Headless: PermissionRequest hooks → auto-deny if no hook approves
  └─ dontAsk mode: convert ask → deny
```

## Design Decisions

**Bypass-immune safety checks.** Even in `bypassPermissions` mode, writes to `.git/`, `.claude/`, `.vscode/`, and shell config files always trigger a prompt. This prevents an agent from modifying its own instructions (`.claude/`) or corrupting the repository (`.git/`). The rationale: these paths have outsized blast radius and the cost of prompting is low.

**Two-stage classifier.** The auto mode classifier runs in two stages: Stage 1 is a fast check (64 tokens, stop at `</block>`) that approves obvious safe actions. Stage 2 only runs if Stage 1 blocks, using chain-of-thought reasoning to avoid false positives. This keeps most approvals fast while allowing nuanced decisions for ambiguous cases.

**Denial tracking with fallback.** In auto mode, if the classifier blocks 3 consecutive actions or 20 total in a session, the system falls back to interactive prompting. This prevents the user from being locked out of their own workflow by an overly cautious classifier. Counters reset on successful tool execution.

**Transcript safety.** The classifier only sees user text and assistant `tool_use` blocks — assistant text blocks are deliberately excluded. This prevents prompt injection where a malicious CLAUDE.md or tool output could craft assistant text that tricks the classifier into approving dangerous actions.

**Rule syntax with escaping.** Permission rules use `ToolName(content)` format with backslash escaping for literal parentheses. Rules from 7 sources (user, project, local, flag, policy, CLI, session) are merged, with policy settings having highest priority.

**Sandbox boundaries.** The optional sandbox restricts filesystem access to the working directory + Claude temp directory. Git bare repo files (HEAD, objects, refs) are always denied to prevent sandbox escape. Settings.json files are always denied to prevent self-modification.

## Insights

- Bash permissions are uniquely complex: commands are parsed to AST, compound commands split into subcommands (capped at 50), and each subcommand evaluated independently. This is why `Bash(git push)` correctly matches `git status && git push` but not `echo "git push"`.
- The `acceptEdits` mode auto-approves file writes within the working directory but still prompts for bash commands. This is a middle ground for users who trust file edits but want oversight on shell execution.
- The speculative bash classifier starts running *while the permission dialog is open*. If it finishes with high confidence before the user responds, the dialog auto-dismisses. This reduces perceived latency without reducing safety.
- The `permissionPromptTool` decision reason indicates an external tool (not the built-in permission system) made the decision. This is used by MCP servers that handle their own auth.

## Key Files

| File | Role |
|------|------|
| `src/types/permissions.ts` | Permission types, modes, rules, decisions |
| `src/utils/permissions/permissions.ts` | Main algorithm: `hasPermissionsToUseTool()` |
| `src/utils/permissions/PermissionMode.ts` | Mode definitions and display properties |
| `src/utils/permissions/permissionRuleParser.ts` | Rule parsing with escaping |
| `src/utils/permissions/yoloClassifier.ts` | Two-stage ML classifier (~1500 lines) |
| `src/utils/permissions/denialTracking.ts` | Denial counter and fallback logic |
| `src/hooks/useCanUseTool.tsx` | React hook for interactive permission dialogs |
| `src/utils/sandbox/sandbox-adapter.ts` | Sandbox filesystem/network boundaries |
| `src/tools/BashTool/bashPermissions.ts` | AST-based bash command analysis |
