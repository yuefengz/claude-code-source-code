# Memory System

## Overview

Claude Code has a persistent, file-based memory system that carries context across conversations. Memories are stored as individual markdown files with YAML frontmatter, indexed by a `MEMORY.md` file that's loaded into every conversation's system prompt.

The system supports 4 memory types (user, feedback, project, reference), project-scoped auto-memory directories, and optional team-shared memory.

## Architecture

```
~/.claude/
├── MEMORY.md                              ← User-level memory (always private)
└── projects/
    └── <sanitized-git-root>/
        └── memory/
            ├── MEMORY.md                  ← Project memory index (loaded into prompt)
            ├── user_role.md               ← Individual memories
            ├── feedback_testing.md
            └── logs/                      ← Daily logs (assistant mode)
                └── YYYY/MM/YYYY-MM-DD.md
```

**Memory lifecycle:**
```
Claude notices relevant info during conversation
  ↓
Step 1: Write memory file (e.g., feedback_testing.md) with frontmatter
Step 2: Add one-line entry to MEMORY.md index
  ↓
Next conversation: MEMORY.md loaded into system prompt
  ↓
Claude reads relevant memory files when needed
  ↓
Stale memories verified against current code before acting on them
```

## Design Decisions

**Two-step save.** Saving a memory requires two writes: the memory file and the MEMORY.md index. This ensures every memory is both discoverable (via index) and self-describing (via frontmatter). The index is a one-liner per memory — fast to scan but pointing to rich detail.

**Type taxonomy.** Four memory types guide what to save and how to use it. Each type has distinct save triggers and usage patterns, preventing the memory from becoming a dumping ground:

- **user** — Who the user is (role, preferences, expertise) → tailor responses
  - *Save triggers:* User reveals their role ("I'm a data scientist"), expertise level ("first time touching React"), responsibilities, or domain knowledge. These signals are often embedded casually in requests rather than stated explicitly.
  - *Usage pattern:* Consulted when the model needs to calibrate its response — e.g., choosing the right level of detail in an explanation, framing frontend concepts in backend terms for a backend engineer, or deciding whether to explain a basic concept or skip ahead.
  - *Scope:* Always private. Never shared in team memory since it describes the individual, not the project.

- **feedback** — What approach to take (do/don't, with why) → avoid repeating mistakes
  - *Save triggers:* Two distinct signal types — **corrections** (explicit: "don't do X", "stop doing Y", "no, not that way") and **confirmations** (implicit: "yes exactly", "perfect", or silently accepting a non-obvious choice). Corrections are easy to spot; confirmations require active attention. Both are saved with the *why* to enable judgment in edge cases.
  - *Usage pattern:* Applied proactively during task execution. Before making an approach decision (testing strategy, PR structure, code style), check feedback memories for relevant guidance. The body structure (rule → Why → How to apply) supports contextual application rather than blind rule-following.
  - *Scope:* Private by default. Only promoted to team when the guidance is a project-wide convention (e.g., "always use real databases in integration tests"), not a personal style preference (e.g., "don't summarize at the end of responses").

- **project** — What's happening (goals, deadlines, decisions) → understand context
  - *Save triggers:* User communicates who is doing what, why, or by when — merge freezes, initiative motivations, architectural decisions with external drivers (legal, compliance, performance targets). Relative dates are always converted to absolute dates at save time ("Thursday" → "2026-03-05") so the memory stays interpretable across conversations.
  - *Usage pattern:* Provides the "why behind the what" for current requests. When a user asks to modify code, project memories reveal whether the change is driven by tech debt cleanup vs. compliance requirements, which affects scope decisions. These memories decay fastest — always verify they're still current before acting on them.
  - *Scope:* Strongly biased toward team, since project state affects all contributors.

- **reference** — Where to find things (external systems) → know where to look
  - *Save triggers:* User mentions external resources and their purpose — issue trackers ("bugs are in Linear project INGEST"), dashboards ("the oncall Grafana board is at X"), Slack channels, documentation wikis, CI/CD URLs, or any out-of-repo system that's relevant to the project.
  - *Usage pattern:* Consulted when the user references an external system or when the model suspects relevant information lives outside the codebase. These memories act as a directory of external resources — they store *where* to look, not the information itself.
  - *Scope:* Usually team, since external system locations are shared knowledge.

**Hard truncation.** MEMORY.md is capped at 200 lines / 25KB. This prevents memory from consuming the context budget. A warning is appended if truncated, signaling the user to prune.

**Project settings excluded from path override.** The `autoMemoryDirectory` setting can be set in user, local, or policy settings — but NOT in project settings. This prevents a malicious repo from redirecting memory writes to sensitive paths like `~/.ssh/`.

**Git root canonicalization.** Git worktrees share the same memory directory as their main repo, via `findCanonicalGitRoot()`. This means `claude` in a worktree sees the same memories as `claude` in the main checkout.

**Stale memory guidance.** The system prompt explicitly warns that memories can become outdated. Before acting on a memory that references specific files, functions, or flags, the model is instructed to verify current state first.

## Insights

- Memories are explicitly scoped to information that *can't be derived from the code or git history*. Architecture, file paths, and recent changes should be found by reading the codebase, not recalled from memory. This keeps memories focused and prevents staleness.
- The "what NOT to save" list is as important as the types: no code patterns, no git history summaries, no debugging recipes, no ephemeral task details. These exclusions apply even when the user explicitly asks to save.
- In assistant mode (KAIROS), daily logs provide an append-only journal that's distilled into topic files by a nightly `/dream` skill. This separates raw capture from curated knowledge.
- Team memory (TEAMMEM feature) adds a shared directory where teammates can read/write common memories while keeping private memories local. The system prompt adjusts to show both contexts.
- Memory path validation rejects relative paths, root/near-root paths, Windows drives, UNC paths, and null bytes — defense against path traversal attacks.

## Key Files

| File | Role |
|------|------|
| `src/memdir/paths.ts` | Path resolution, validation, git root sharing |
| `src/memdir/memdir.ts` | Prompt building, truncation, memory loading |
| `src/memdir/memoryTypes.ts` | Type taxonomy (user, feedback, project, reference) |
