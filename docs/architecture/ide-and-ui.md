# IDE Integration & Terminal UI

## Overview

Claude Code integrates with IDEs (VS Code, Cursor, Windsurf, 15 JetBrains IDEs) via process detection and lockfile-based connections. The terminal UI is powered by a custom Ink framework (~13K lines) — a React-based rendering engine that uses Yoga flexbox, diff-based terminal updates, and cell pooling for efficient rendering.

## Architecture

**IDE integration:**
```
IDE process running
  ↓
Detection: process name matching → ancestor PID walk → lockfile scan
  ↓
.claude-ide.lock: { pid, ideName, transport, port, authToken }
  ↓
Connection: WebSocket or SSE based on lockfile transport
  ↓
Features: visual diffs, file picker, MCP server exposure
```

**Terminal UI stack:**
```
React Components (146 components, JSX)
  ↓
React Fiber Reconciler (custom, not react-dom)
  ↓
Virtual DOM (ink-box, ink-text, ink-link, ink-progress, ink-raw-ansi)
  ↓
Yoga Layout Engine (flexbox in terminal)
  ↓
Screen Buffer (cell pooling: char, style, hyperlink pools)
  ↓
Terminal I/O (diff-based updates, Kitty keyboard protocol)
```

## Design Decisions

**Custom Ink framework.** Rather than using the upstream Ink.js library, Claude Code maintains a full custom implementation (~13K lines). This allows deep integration with features like virtual scrolling, text selection, mouse tracking, and focus management that the upstream library doesn't support or supports differently.

**Diff-based terminal updates.** `writeDiffToTerminal()` compares the current screen buffer against the previous frame and only writes changed cells. This is dramatically faster than full-screen rewrites, especially for incremental changes like streaming text or progress updates.

**Cell pooling.** The screen buffer reuses `CharPool`, `StylePool`, and `HyperlinkPool` objects across frames. Since most cells don't change between renders, this avoids allocation pressure and GC pauses in the hot render loop.

**Process-based IDE detection.** IDE detection uses platform-specific process keyword matching and ancestor PID chain walking. This is more reliable than environment variables (which may not be set in all terminal contexts) and works across terminal multiplexers.

**Lockfile protocol.** IDEs write a `.claude-ide.lock` file with connection details (transport, port, auth token). Claude Code reads this to establish a bidirectional channel. The lockfile approach is simpler than service discovery and works offline.

**React component architecture.** The entire UI is React — from the root `App.tsx` (with `AppStateProvider`, `StatsProvider`, `FpsMetricsProvider`) down to individual tool result renders. This enables declarative composition, memoization, and the familiar hooks pattern (`useMergedTools`, `useCanUseTool`, `useAutoMode`, etc.) for state management.

**Virtual scrolling with quantized re-renders.** The virtual scroll system (`useVirtualScroll.ts`) uses a `SCROLL_QUANTUM` (40 rows) to batch wheel events. Scroll ticks that don't cross a quantum bin trigger NO React commit — only Ink's imperative `forceRender()` reads the live `scrollTop` from the DOM node. This decouples React rendering from scroll smoothness. On resize, cached heights are scaled by `oldCols/newCols` instead of cleared (a cold clear would re-measure ~190 items at ~600ms), and renders freeze for 2 frames to prevent mount/unmount churn while Yoga heights are stale.

**Terminal-specific scroll drain rates.** The renderer uses different scroll drain strategies per terminal:
- *xterm.js (VS Code)*: Pending ≤5 drains entirely in one frame (instant click response). Higher pending uses fixed steps (2-3 rows) for smooth fast flicks. Excess beyond 30 is snapped to prevent coasting.
- *Native terminals*: Uses `floor(pending * 3/4)`, capped at `innerHeight-1`, so the hardware scroll fast path (DECSTBM + SU/SD) fires and the renderer can emit a blit+shift instead of rewriting the whole viewport.

**Stack-based focus management.** The focus system (`focus.ts`) uses a stack (max 32 entries) with deduplication. When a node is removed, the stack is filtered to only in-tree entries, and focus restores to the most recent still-mounted element. The FocusManager owns no tree reference — callers walk `parentNode` to find it, like the browser's `getRootNode()`.

**SSH-safe terminal capability detection.** `TERM_PROGRAM` isn't forwarded over SSH, so the fallback sends a CSI `>0q` (XTVERSION) query through the pty — this reaches the *client* terminal and returns via stdin, detecting xterm.js in VS Code even over SSH. Only whitelisted terminals (iTerm, kitty, WezTerm, Ghostty, tmux) use the Kitty keyboard protocol, because some terminals (notably xterm.js over SSH) honor the enable sequence but emit codepoints the parser doesn't handle. DEC 2026 synchronized output is skipped in tmux because tmux parses every byte but doesn't implement it — the BSU/ESU pass through but tmux already broke atomicity by chunking.

**Lockfile validation and stale cleanup.** IDE lockfiles are validated by checking if the PID is running (`process.kill(pid, 0)`) and if the port responds (socket connection test with 500ms timeout). WSL uses port connectivity as the authoritative signal since PIDs are unreliable across subsystems. All `.lock` files are stat'd in parallel (not serial), sorted by mtime (newest first).

**Unicode normalization for IDE paths.** Workspace-folder comparison normalizes to NFC (composed Unicode) because macOS returns NFD paths but IDEs report NFC — without this, accented and CJK characters in paths fail to match.

**Selection styling via bit encoding.** Styles visible on space characters (background, inverse, underline) get odd IDs; foreground-only styles get even IDs. The renderer skips invisible spaces with a single bitmask check instead of per-cell introspection. Search match highlighting filters both foreground AND background from the base style before applying yellow-via-inverse, because with explicit user-prompt backgrounds, inverse swaps colors unpredictably across terminals.

**Deferred cursor for IME support.** `useDeclaredCursor` writes are deferred to a microtask after `useLayoutEffect` commits, so IME preedit appears inline at the text caret without a one-keystroke lag.

## Insights

- The Yoga layout engine provides full CSS flexbox support in the terminal — `flexDirection`, `justifyContent`, `alignItems`, `flexWrap`, etc. This makes complex layouts (side-by-side diffs, nested progress indicators) straightforward.
- Text selection in the terminal (alt-screen mode) supports click, shift+arrow, and triple-click (select line). This is 917 lines of selection logic — surprisingly complex because terminal text doesn't have DOM-like selection APIs.
- JetBrains plugin detection searches platform-specific directories (macOS Library, Windows AppData, Linux .config) for the `claude-code-jetbrains-plugin` prefix. Results are cached to avoid repeated disk scans.
- The `REPL.tsx` component (~5000 lines) is the largest React component, handling message rendering, tool merging, query execution, transcript mode, search, and virtually everything the user sees.
- The renderer detects when the previous frame's screen buffer is "contaminated" (by selection changes, resize, or absolute-positioned element removal) and falls back to a full repaint rather than a diff-based update.
- When sticky-scroll fires, the renderer consumes `followScroll` post-render to translate the text selection by `-delta` so the highlight stays anchored to scrolling text, not the viewport.
- Render-time scroll clamping (`scrollClampMin/Max`) prevents blank screens when scroll input outpaces React's async commits — the clamp holds the viewport at the edge of mounted content until React catches up.

## Key Files

| File | Role |
|------|------|
| `src/utils/ide.ts` | IDE detection, process scanning, lockfile validation |
| `src/utils/jetbrains.ts` | JetBrains plugin detection (cross-platform) |
| `src/commands/ide/ide.tsx` | `/ide` command: selection UI, auto-connect dialog |
| `src/ink/ink.tsx` | Main Ink renderer, event loop (~1720 lines) |
| `src/ink/dom.ts` | Virtual DOM: element types, yoga nodes, dirty tracking |
| `src/ink/renderer.ts` | Render pipeline: layout → output → screen |
| `src/ink/render-node-to-output.ts` | Terminal-specific scroll drain rates |
| `src/ink/screen.ts` | Screen buffer with cell pooling, bit-encoded styles (~1486 lines) |
| `src/ink/selection.ts` | Text selection logic (~917 lines) |
| `src/ink/focus.ts` | Stack-based focus management (max 32) |
| `src/ink/terminal.ts` | Terminal I/O, capability detection, diff-based updates |
| `src/hooks/useVirtualScroll.ts` | Quantized virtual scrolling with height scaling |
| `src/components/App.tsx` | Root component with providers |
