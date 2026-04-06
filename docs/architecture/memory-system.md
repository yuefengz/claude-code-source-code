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

**Write routing is prompt-driven.** There is no code logic deciding where a memory gets written — the model follows instructions injected by `buildMemoryLines()`. The rules are:

- **Always two writes (normal mode):** Every save requires (1) writing or updating an individual memory file with frontmatter (e.g., `feedback_testing.md`), then (2) adding or updating a one-line pointer in `MEMORY.md`. Content never goes directly into the index — `MEMORY.md` only holds short pointers (`- [Title](file.md) — one-line hook`).
- **New file vs. existing file:** The model is instructed: "Do not write duplicate memories. First check if there is an existing memory you can update before writing a new one." Since `MEMORY.md` is already in the model's context, it scans the index to judge whether an existing entry covers the same topic, then either updates that file or creates a new one. There is no deduplication logic in code.
- **Assistant mode (KAIROS) is different:** In long-lived sessions, the model skips both steps and instead appends timestamped bullets to a daily log file (`logs/YYYY/MM/YYYY-MM-DD.md`). A separate nightly `/dream` skill distills those logs into topic files and `MEMORY.md`. The model is explicitly told not to edit `MEMORY.md` directly in this mode.

**No complex retrieval.** Memory retrieval is deliberately simple — there is no embedding search, vector store, or ranking. `MEMORY.md` is read synchronously via `readFileSync` and injected verbatim into the system prompt. When the model needs more detail, it uses the standard `Read` tool to open individual memory files. For cross-file search, it uses `Grep` against the memory directory — the same tool used for code search. The complexity lives in the prompting (type taxonomy, staleness caveats, trust guidelines), not in any retrieval mechanism.

**Hard truncation.** MEMORY.md is capped at 200 lines / 25KB. This prevents memory from consuming the context budget. A warning is appended if truncated, signaling the user to prune.

**Project settings excluded from path override.** The `autoMemoryDirectory` setting can be set in user, local, or policy settings — but NOT in project settings. This prevents a malicious repo from redirecting memory writes to sensitive paths like `~/.ssh/`.

**Git root canonicalization.** Git worktrees share the same memory directory as their main repo, via `findCanonicalGitRoot()`. This means `claude` in a worktree sees the same memories as `claude` in the main checkout.

**Stale memory guidance.** The system prompt explicitly warns that memories can become outdated. Before acting on a memory that references specific files, functions, or flags, the model is instructed to verify current state first.

**Staleness injection at read time.** Beyond prompt-level warnings, `memoryAge.ts` injects per-file staleness caveats when a memory is read. For memories >1 day old, the model sees: *"This memory is N days old. Memories are point-in-time observations, not live state — claims about code behavior or file:line citations may be outdated. Verify against current code before asserting as fact."* Memories from today or yesterday get no caveat (adding one would be noise). This was motivated by user reports of stale file:line citations being asserted as fact — the citation made the stale claim sound *more* authoritative, not less.

**Extract-memories background agent.** A background agent (`isExtractModeActive()`, gated by feature flag `tengu_passport_quail`) runs alongside the main conversation to catch memories the main agent missed. When the main agent writes memories itself, the background agent skips that range (via `hasMemoryWritesSince`); when it doesn't, the background agent extracts from the conversation. This dual-write design means the main agent's prompt always has full save instructions regardless of whether background extraction is active.

**Memory file scanning.** `memoryScan.ts` provides a single-pass scan shared by query-time recall and the extraction agent. It reads all `.md` files in the memory directory (excluding `MEMORY.md`), extracts frontmatter from only the first 30 lines per file, and sorts by mtime (newest-first), capped at 200 files. The read-then-sort approach (vs. stat-sort-read) halves syscalls for the common case. The scan produces a manifest format (`[type] filename (ISO timestamp): description`) used by both recall selection and extraction prompts.

**Prompt caching.** Memory prompts are built once per session via `systemPromptSection('memory', ...)` and cached until `/clear` or `/compact`. In KAIROS mode, the daily log path is described as a pattern (`YYYY/MM/YYYY-MM-DD.md`) rather than a literal date so the cached prompt doesn't need invalidation at midnight — the model derives the current date from a `date_change` attachment appended at midnight rollover.

**Eval-driven prompt engineering.** The prompt sections in `memoryTypes.ts` contain inline eval annotations documenting measurable improvements from prompt positioning:
- The "Before recommending from memory" section (H1) went from 0/2 → 3/3 when moved from a bullet under "When to access" to its own section — the model needed it as a separate "action cue at the decision point" rather than buried in a list.
- The "ignore memory" bullet (H6) was added after observing the model would cite memory while claiming to ignore it — treating "ignore" as "acknowledge then override" rather than "don't reference at all."
- The "what NOT to save" gate (H2) prevents ephemeral data noise: when users ask to save a PR list or activity summary, the prompt asks "what was surprising or non-obvious?" instead of saving verbatim.
- Known gap: verification doesn't cover slash-command claims (0/3 on the `/fork` case — slash commands aren't files or functions in the model's ontology).

**Team memory security: symlink containment.** Team memory (`teamMemPaths.ts`) implements multi-layer defense against symlink attacks beyond basic path validation:
- `realpathDeepestExisting()` walks up the directory tree to the deepest existing ancestor, resolves symlinks there, then rejoins the remaining path. Catches dangling symlinks, symlink loops (ELOOP), and filesystem permission errors.
- `isRealPathWithinTeamDir()` compares canonical filesystem paths after symlink resolution, with separator-aware prefix matching to prevent `team-evil/` matching `team/`.
- `validateTeamMemKey()` runs two-phase validation: fast string-level containment check via `path.resolve()`, then symlink resolution. Also defends against URL-encoded traversal (`%2e%2e%2f`) and Unicode normalization attacks (fullwidth `．．／` normalizing to `../`).

**Cowork/SDK integration.** The memory system supports external orchestrators via two mechanisms: `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` redirects memory to a space-scoped mount (avoiding per-session cwd pollution), and `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` injects additional policy text into all memory prompt variants (e.g., per-workspace memory conventions).

## Insights

- Memories are explicitly scoped to information that *can't be derived from the code or git history*. Architecture, file paths, and recent changes should be found by reading the codebase, not recalled from memory. This keeps memories focused and prevents staleness.
- The "what NOT to save" list is as important as the types: no code patterns, no git history summaries, no debugging recipes, no ephemeral task details. These exclusions apply even when the user explicitly asks to save.
- In assistant mode (KAIROS), daily logs provide an append-only journal that's distilled into topic files by a nightly `/dream` skill. This separates raw capture from curated knowledge.
- Team memory (TEAMMEM feature) adds a shared directory where teammates can read/write common memories while keeping private memories local. The system prompt adjusts to show both contexts.
- Memory path validation rejects relative paths, root/near-root paths, Windows drives, UNC paths, and null bytes — defense against path traversal attacks.
- Prompt section positioning has measurable impact on model behavior — the same text achieves different eval scores depending on whether it's a standalone section vs. a bullet in a list. This makes the prompt layout itself a load-bearing design decision.
- The memory system is designed for defense-in-depth: path validation, symlink resolution, Unicode normalization, URL-decoding, and project-settings exclusion form overlapping layers rather than relying on any single check.
- Models are poor at date arithmetic — `memoryAge.ts` converts raw mtime to "N days ago" because a human-readable string triggers staleness reasoning in a way that an ISO timestamp does not.
- The dual-write architecture (main agent + background extraction agent) ensures memory capture even when the main agent is too focused on the task to notice save-worthy information.

## Key Files

| File | Role |
|------|------|
| `src/memdir/paths.ts` | Path resolution, validation, git root sharing |
| `src/memdir/memdir.ts` | Prompt building, truncation, memory loading |
| `src/memdir/memoryTypes.ts` | Type taxonomy (user, feedback, project, reference) |
| `src/memdir/memoryAge.ts` | Per-file staleness caveat injection |
| `src/memdir/memoryScan.ts` | Single-pass directory scan, frontmatter extraction, manifest formatting |
| `src/memdir/teamMemPaths.ts` | Team memory paths, symlink containment, key validation |
| `src/memdir/teamMemPrompts.ts` | Combined (private + team) prompt building with scope tags |
