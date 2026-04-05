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

**Type taxonomy.** Four memory types guide what to save and how to use it:
- **user** — Who the user is (role, preferences, expertise) → tailor responses
- **feedback** — What approach to take (do/don't, with why) → avoid repeating mistakes
- **project** — What's happening (goals, deadlines, decisions) → understand context
- **reference** — Where to find things (external systems) → know where to look

Each type has different save triggers and usage patterns, preventing the memory from becoming a dumping ground.

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
