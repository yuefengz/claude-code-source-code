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

## Insights

- The Yoga layout engine provides full CSS flexbox support in the terminal — `flexDirection`, `justifyContent`, `alignItems`, `flexWrap`, etc. This makes complex layouts (side-by-side diffs, nested progress indicators) straightforward.
- Text selection in the terminal (alt-screen mode) supports click, shift+arrow, and triple-click (select line). This is 917 lines of selection logic — surprisingly complex because terminal text doesn't have DOM-like selection APIs.
- JetBrains plugin detection searches platform-specific directories (macOS Library, Windows AppData, Linux .config) for the `claude-code-jetbrains-plugin` prefix. Results are cached to avoid repeated disk scans.
- The `REPL.tsx` component (~5000 lines) is the largest React component, handling message rendering, tool merging, query execution, transcript mode, search, and virtually everything the user sees.
- The renderer detects when the previous frame's screen buffer is "contaminated" (by selection changes, resize, or absolute-positioned element removal) and falls back to a full repaint rather than a diff-based update.

## Key Files

| File | Role |
|------|------|
| `src/utils/ide.ts` | IDE detection, process scanning, lockfile parsing |
| `src/utils/jetbrains.ts` | JetBrains plugin detection (cross-platform) |
| `src/commands/ide/ide.tsx` | `/ide` command: selection UI, auto-connect dialog |
| `src/ink/ink.tsx` | Main Ink renderer, event loop (~1720 lines) |
| `src/ink/dom.ts` | Virtual DOM: element types, yoga nodes, dirty tracking |
| `src/ink/renderer.ts` | Render pipeline: layout → output → screen |
| `src/ink/screen.ts` | Screen buffer with cell pooling (~1486 lines) |
| `src/ink/selection.ts` | Text selection logic (~917 lines) |
| `src/ink/terminal.ts` | Terminal I/O, diff-based updates |
| `src/components/App.tsx` | Root component with providers |
