# Chapter 12: Terminal UI and Rendering Engine

## 12.1 Overview and Purpose

Claude Code is an interactive AI coding assistant running in the terminal. Its UI layer needs to deliver a near-GUI interactive experience in a plain-text terminal environment: streaming message rendering, Vim keybinding editing, mouse text selection, search highlighting, and multi-layer overlay dialogs. These requirements far exceed the capability boundaries of traditional CLI tools.

Claude Code's terminal UI layer is built on three core technical decisions:

1. **Custom Ink Renderer** (`src/ink/`): Deep reworking of the React + Ink framework, introducing a double-buffered screen model, ANSI diff engine, hardware scroll acceleration, and IME-aware physical cursor positioning.
2. **Declarative Component Tree** (`src/screens/REPL.tsx`, `src/components/`): Using React components to describe UI structure; the rendering engine translates the virtual DOM into a terminal character matrix.
3. **Full Vim Mode** (`src/vim/`): An independent state machine implementation covering NORMAL/INSERT dual modes, the complete set of operators/motions/textObjects, as well as dot-repeat and registers.

The entire rendering pipeline's tempo is controlled by `FRAME_INTERVAL_MS = 16` (`src/ink/constants.ts:1`), giving a theoretical frame rate of 62.5fps, aligned with display refresh rates.

---

## 12.2 Theoretical Foundation

### Low-Level Principles of Terminal Rendering

Terminal rendering is essentially writing a byte stream to a file descriptor; the terminal emulator is responsible for parsing it and updating the screen. Key mechanisms include:

- **ANSI escape codes**: Control sequences starting with `\x1b[`, controlling cursor position (CSI CUP), color (SGR), scroll regions (DECSTBM), mouse tracking (DEC private modes), etc. Claude Code's `src/ink/termio/` directory encapsulates these sequences as named constants (`CURSOR_HOME`, `ENTER_ALT_SCREEN`, `ENABLE_MOUSE_TRACKING`, etc.).
- **termios / raw mode**: In raw mode, each keypress is delivered to the program immediately without line buffering or echo. Claude Code takes over stdin via `setRawMode(true)` at startup, restoring it via `Ink.detachForShutdown()` on exit (`ink.tsx:942-950`).
- **Alt Screen (`?1049h/?1049l`)**: An alternate screen buffer; entering it doesn't pollute the main screen's scroll history, and exiting restores the original state. In fullscreen mode (`isFullscreenEnvEnabled()`), the REPL is wrapped in an `<AlternateScreen>` component.

### Double Buffering Rendering

Claude Code uses front and back frame buffers (`frontFrame` / `backFrame`) to implement double buffering (`ink.tsx:196-197`):

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

Each frame renders as follows:
1. Renderer writes the React virtual DOM into `backFrame` (new frame)
2. LogUpdate diff engine compares `frontFrame` (old frame) with `backFrame`, generating a minimal patch sequence
3. The patch sequence is written to the terminal; `frontFrame` ↔ `backFrame` swap (`ink.tsx:594`)

This way the terminal only receives characters that actually changed, rather than fully repainting each frame.

### React in Non-DOM Environments (Custom Renderer)

React's renderer interface (`react-reconciler`) allows any host environment. Claude Code implements a Custom Renderer for the terminal in `src/ink/reconciler.ts` (512 lines):

- The host element type is the custom `dom.DOMElement` (`src/ink/dom.ts`)
- The layout engine uses Yoga (Facebook's Flexbox implementation), driven in the `onComputeLayout` callback (`ink.tsx:239-258`)
- React Concurrent Root mode (`ConcurrentRoot`) allows interruptible rendering, avoiding long blocking of the main thread

---

## 12.3 Ink Custom Renderer

### Claude Code's Customizations to Ink

`src/ink/ink.tsx` (1,722 lines) is the core controller of the entire UI layer, defining the `Ink` class. This is not direct use of the original Ink package but a deeply customized version, with core enhancements including:

**Double buffering + front/back frame management**: The original Ink outputs strings directly; the custom version introduces Frame objects and a `Screen` matrix.

**Performance monitoring**: Each frame records the duration of renderer, diff, optimize, and write phases, as well as Yoga layout metrics (visited/measured/cacheHits), exposed via the `onFrame` callback (`ink.tsx:772-788`):

```typescript
// ink.tsx:772-788
this.options.onFrame?.({
  durationMs: performance.now() - renderStart,
  phases: {
    renderer: rendererMs,
    diff: diffMs,
    optimize: optimizeMs,
    write: writeMs,
    patches: diff.length,
    yoga: yogaMs,
    commit: commitMs,
    yogaVisited: yc.visited,
    yogaMeasured: yc.measured,
    yogaCacheHits: yc.cacheHits,
    yogaLive: yc.live,
  },
  flickers,
});
```

**External TUI handoff**: Provides `enterAlternateScreen()` / `exitAlternateScreen()` methods, suspending Ink's control, closing the kitty keyboard protocol, allowing external editors (vim, nano) to run normally, then fully restoring on return (`ink.tsx:357-419`).

**SIGCONT self-healing**: `handleResume` handles process recovery from suspension (fg command); main screen resets buffers, alt screen re-enters and clears (`ink.tsx:280-301`).

### Double Buffering + Diff Update Mechanism

`LogUpdate.render()` in `src/ink/log-update.ts` is the core of the diff engine. It accepts `prev: Frame` and `next: Frame`, returning a set of `Diff` patch operations:

```
Diff patch types:
- { type: 'stdout', content: string }    — write directly to terminal
- { type: 'cursorMove', x, y }           — relative cursor movement
- { type: 'cursorTo', x, y }             — absolute cursor positioning
- { type: 'styleStr', str }              — SGR style switch
- { type: 'hyperlink', uri }             — OSC 8 hyperlink
- { type: 'cursorHide' | 'cursorShow' }  — cursor visibility
- { type: 'clear', count }               — erase characters
- { type: 'clearTerminal', reason }      — full screen repaint (triggers flicker)
```

The diff engine is optimized in `src/ink/optimizer.ts` in a single pass (`optimize(diff)`), merging adjacent `cursorMove`, collapsing consecutive `cursorTo`, concatenating adjacent `styleStr`, canceling cursor hide/show pairs, reducing actual write volume.

### Int32Array Screen Buffer

`src/ink/screen.ts` defines the `Screen` type, using `Int32Array` rather than object arrays to store screen cells, eliminating GC pressure (`screen.ts:356-370`):

```typescript
// screen.ts:356-370 (comment)
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen` allocates an `ArrayBuffer`, simultaneously mounting two views (`Int32Array` and `BigInt64Array`); the latter is used by `resetScreen` for bulk zeroing (`screen.ts:472-490`):

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes per cell
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

Characters and styles are interned through **shared Pool objects** (`CharPool`, `StylePool`, `HyperlinkPool`), sharing IDs across Screens, allowing `blitRegion` to copy integer IDs directly without re-interning, and `diffEach` to use integer comparisons instead of string comparisons (`screen.ts:17-31`).

`StylePool.intern()`'s bit-0 encoding indicates visibility (`screen.ts:148-161`): odd IDs indicate the style is visible on spaces (background color, reverse video, underline), allowing the renderer to skip invisible spaces with a single bit mask, avoiding unnecessary writes.

### Hardware Scrolling (DECSTBM)

DECSTBM hardware scroll optimization is implemented in `log-update.ts` (`log-update.ts:149-185`):

When `ScrollBox`'s `scrollTop` changes in alt-screen mode, instead of rewriting the entire scroll region, it sends `CSI top;bot r` (set scroll region) + `CSI n S` (scroll up n lines), letting terminal hardware accelerate the content shift, and only the newly scrolled-in lines need to be supplemented. Simultaneously, `shiftRows()` is applied to `prev.screen` to simulate scrolling, so subsequent diff loops naturally find different rows rather than fully rewriting.

This optimization is only enabled when `decstbmSafe=true` — when the external terminal doesn't support DEC 2026 (BSU/ESU atomic operations), terminals like tmux render intermediate states (after scrolling but before new rows are supplemented), causing visible flicker, so it falls back to full diff (`log-update.ts:158-167`).

### 16,384-Character Cache

The `Output` class in `src/ink/output.ts` (the character measurement and output engine inside the renderer) maintains a `charCache` caching the mapping from characters to measured widths. When the cache exceeds 16,384 entries, it is simply cleared and rebuilt (`output.ts:204`):

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

Similarly, `src/ink/line-width-cache.ts` maintains a line-width cache (maximum 4,096 entries) for reusing `stringWidth` calls for the many unchanged lines during streaming output (`line-width-cache.ts`):

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### Performance Optimization Strategies

| Strategy | Location | Effect |
|----------|----------|--------|
| Render scheduling microtask delay | `ink.tsx:212-216` | scheduleRender uses `queueMicrotask`, ensuring useLayoutEffect completes before rendering, avoiding single-frame cursor position lag |
| Throttle 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`, stable frame rate |
| Drain timer 4x frame rate | `ink.tsx:757-758` | scroll continuous drain uses `FRAME_INTERVAL_MS >> 2` (~4ms), maximizing scroll smoothness |
| Pool periodic reset | `ink.tsx:600-603` | Reset char/hyperlink pools every 5 minutes, preventing memory growth in long sessions |
| Damage rectangle | `screen.ts:382` | Each frame only diffs the actually written rectangular region, not the full screen |
| Alt-screen cursor anchoring | `ink.tsx:578-591` | Before each frame diff, use `ALT_SCREEN_ANCHOR_CURSOR` to reset the virtual cursor to (0,0), preventing cursor drift in tmux and similar terminals |

---

## 12.4 REPL Main Interface

### REPL.tsx Component Structure

`src/screens/REPL.tsx` (5,005 lines) is Claude Code's main interface component and the largest single file in the entire project. Its top-level imports span over 150 lines, covering session management, permission systems, MCP connections, Swarm multi-agent, voice integration, and nearly all other functional modules.

The REPL render tree structure (simplified):

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen (fullscreen mode)
    │   └── Box (main layout)
    │       ├── AnimatedTerminalTitle (terminal tab title)
    │       ├── FullscreenLayout
    │       │   ├── Messages (message list)
    │       │   ├── VirtualMessageList (virtual scrolling)
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── various Callout / Survey / Dialog
    │       └── PromptInput (input area)
    └── DevBar (ant-only debug bar)
```

Fullscreen mode is determined by a conditional branch on whether to wrap `<AlternateScreen>` (`REPL.tsx:4998-5001`):

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### Message Rendering Flow

Message state is held by React `useState<MessageType[]>`, with `handleMessageFromStream` processing streaming SSE events to append/update messages. The message list is rendered to the `<Messages>` or `<VirtualMessageList>` components; the latter uses virtual scrolling to only render the visible area, avoiding large numbers of DOM nodes in long conversations.

REPL contains multiple optimizations to avoid unnecessary re-renders: `AnimatedTerminalTitle` is extracted as a leaf node to prevent the 960ms animation tick from re-rendering the entire REPL tree (comment in `REPL.tsx` notes: `Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`).

`useDeferredValue(searchQuery)` is used to delay search computation; `React.memo` caches the `PromptInput` subtree, preventing frequent renders triggered by character input from propagating upward.

### Input Processing Pipeline

User input enters Ink's `parse-keypress.ts` from stdin raw mode, parsed into `ParsedKey` objects, distributed via FocusManager to the current focus node's `useInput` callback. `GlobalKeybindingHandlers` and `CommandKeybindingHandlers` provide global shortcut registration mechanisms with priority over component-level `useInput`.

Input ultimately arrives at the `PromptInput` component, routing to `VimTextInput` or `TextInput` based on the current mode (Vim/Standard).

---

## 12.5 Vim Mode

### Vim Mode Implementation Architecture

The `src/vim/` directory (5 files, ~1,513 lines) implements a complete Vim key layer. The architecture design is extremely clean: **types as documentation**, with all state machine transitions encoded as TypeScript types, and TypeScript's exhaustive checking guaranteeing complete state handling.

File responsibilities:
- `types.ts`: State machine type definitions + constant sets
- `transitions.ts`: Transition functions (main entry `transition()`)
- `motions.ts`: Cursor movement pure functions (`resolveMotion()`)
- `operators.ts`: Operation execution functions (delete/change/yank, etc.)
- `textObjects.ts`: Text object boundary calculation

### Supported Operations (motions, operators, textObjects)

**Operators**: `d` (delete), `c` (change), `y` (yank) — defined in `types.ts:OPERATORS`.

**Simple Motions** (`types.ts:SIMPLE_MOTIONS`):
- Basic movement: `h`/`l`/`j`/`k`
- Word movement: `w`/`b`/`e`/`W`/`B`/`E`
- Line position: `0`/`^`/`$`

**Find Motions**: `f`/`F`/`t`/`T` (character search), `;`/`,` repeat/reverse.

**Text Objects** (`types.ts:TEXT_OBJ_TYPES`): `w`/`W` (word), `"`/`'`/`` ` `` (quotes), `(`/`)`/`b` (parentheses), `[`/`]` (brackets), `{`/`}`/`B` (braces), `<`/`>` (angle brackets).

**Line-level operations**: `dd`/`cc`/`yy`, `D`/`C`/`Y`, `o`/`O` (open line), `J` (join), `>>`/`<<` (indent).

**Other commands**: `r` (replace char), `~` (toggle case), `x` (delete char), `p`/`P` (paste), `u` (undo), `G`/`gg` (jump to line), `.` (dot-repeat).

Special handling: `cw`/`cW` changes to word end rather than the start of the next word (classic vim behavior, `operators.ts:91-102`); Image ref chips are recognized as atomic units, so word movement doesn't stop inside a chip (`operators.ts:333-337`).

### State Machine and Mode Switching

`VimState` is the top-level type (`types.ts:52-55`):

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

INSERT mode records inserted text for dot-repeat; NORMAL mode holds the `CommandState` sub-state machine.

`CommandState` has 11 states (`types.ts:62-75`), covering idle, count accumulation, operator pending, operatorCount, operatorFind, operatorTextObj, find, g-prefix, operatorG, replace, and indent. The state diagram is recorded in ASCII art form at the beginning of `types.ts` (`types.ts:1-26`):

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

The `transition()` function (`transitions.ts:55-73`) dispatches to dedicated handler functions based on the current `CommandState` type, each returning `{ next?: CommandState; execute?: () => void }`. When `execute` is present, it runs immediately; when `next` is present, it advances the state; when neither is present, it returns to idle.

`PersistentState` (`types.ts:80-87`) persists across commands: `lastChange` (dot-repeat recording), `lastFind` (`;`/`,` repeat), `register` (clipboard), `registerIsLinewise` (line-level paste judgment).

Count multiplication is implemented in `fromOperatorCount` (`transitions.ts`): `effectiveCount = state.count * motionCount`, supporting `3d2w` = delete 6 words. Count is capped at `MAX_VIM_COUNT = 10000` (`types.ts:96`) to prevent malicious input.

---

## 12.6 Core Component Analysis

### PromptInput Input Component

`src/components/PromptInput/PromptInput.tsx` (2,338 lines) is one of Claude Code's highest-density components, handling nearly all input-related interaction logic.

**Props scale**: The `Props` type definition spans approximately 60 lines, containing input value, mode, Vim state, stash stack, submit callbacks, permission context, MCP clients, IDE selection, voice integration, and more.

**External input injection**: `insertTextRef` exposes a `{ insert, setInputWithCursor, cursorOffset }` interface, allowing voice recognition (STT) to insert text at the cursor position rather than replacing the entire input (`PromptInput.tsx:180-200`).

**Cursor management**: Independently tracks `cursorOffset` state, distinguishing internal changes (via `trackAndSetInput` wrapping `onInputChange`) from external injection (voice input); external changes move the cursor to the end (`PromptInput.tsx:164-176`).

**Mode routing**: Renders `<VimTextInput>` or `<TextInput>` based on `isVimModeEnabled()`, both sharing the `BaseTextInputProps` interface.

**Bottom layout constants** (`PromptInput.tsx`):
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
The bottom input slot has a maximum height of 50%, reserving 5 lines for the footer, borders, and status hints.

### Settings Interface

`src/components/Settings/Config.tsx` (1,821 lines) implements the settings interface for the `/config` command.

**Search-driven navigation**: Config defaults to search mode (`isSearchMode = true`), allowing users to type directly to filter settings items without using arrow keys to scroll through the full list. `maxVisible = Math.max(5, paneCap - 10)` dynamically calculates the number of visible items based on terminal height.

**Settings categorization**: The `settingsItems: Setting[]` array uniformly defines all configuration items, each being `boolean` (toggle), `enum` (enum selection), or `managedEnum` (custom component-managed).

**Change tracking and restoration**: The `changes` state records all modifications; the `isDirty` ref is used for Escape key judgment on whether to write to disk on revert, avoiding meaningless disk writes when opening and immediately closing (`Config.tsx`).

**Submenu system**: `showSubmenu` state controls the expansion of submenus for Theme, Model, TeammateModel, ExternalIncludes, OutputStyle, Language, EnableAutoUpdates, etc., communicating with the parent Settings component via `setTabsHidden` to hide the Tab header.

**Multi-source configuration architecture**: Settings sources are divided into `localSettings` (project-level) and `userSettings` (user-level), read/written via `getSettingsForSource`/`updateSettingsForSource`; Escape restoration uses initial snapshots per source to restore rather than reading the merged global view (note in `Config.tsx` explains this is to support `undefined`-delete-key semantics).

### Log Selector

`src/components/LogSelector.tsx` (1,574 lines) implements the session history selection interface (`/resume` command).

**Tree display**: Via `<TreeSelect<LogTreeNode>>`, forked sessions are displayed in a parent-child tree, with expand/collapse managed by `expandedGroupSessionIds`.

**Three view modes**: `viewMode` can be `"list"` (default list), `"search"` (search), or `"preview"` (session preview).

**Fuse.js fuzzy search**: Uses `FUSE_THRESHOLD = 0.3` fuzzy matching, combined with `DATE_TIE_THRESHOLD_MS = 60 * 1000` (results within 1 minute are sorted by relevance, otherwise by time).

**Search result snippet preview**: `extractSnippet()` takes `SNIPPET_CONTEXT_CHARS = 50` characters before and after the match position, rendering context with `chalk.dim` and matched words in highlight color (`LogSelector.tsx`).

**Deep search constants**:
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
(Currently `isDeepSearchEnabled = false`, pending feature)

**React Compiler optimization**: `import { c as _c } from "react/compiler-runtime"` at the top of the file indicates it has been compiled with React Compiler; `_c(247)` allocates a memo cache with 247 slots, with extensive conditional memoization to avoid subtree re-renders.

---

## 12.7 Design Decision Analysis

### Why a Custom Ink Renderer Rather Than the Original

The original Ink serializes the React tree to a string and uses `log-update` for incremental writing, designed for simple CLI tools. Claude Code's requirements exceed this model's limits:

1. **Mouse selection**: Requires screen-level cell coordinate mapping (hit-test); the string model cannot support this.
2. **Search highlighting**: Requires overlaying reverse-color cells on an already-rendered frame; the cell matrix must be operated on directly.
3. **DECSTBM hardware scrolling**: Requires maintaining consistency between prev/next frames in the scroll region; string diff cannot simulate the effect of terminal hardware scrolling.
4. **IME / a11y cursor positioning**: Requires knowing the precise physical screen coordinates of the input cursor, positioning the physical terminal cursor correctly (CJK input method preedit text renders at the physical cursor position).
5. **Alt-screen atomicity (BSU/ESU)**: Requires wrapping erase + paint in `\x1b[?2026h`/`\x1b[?2026l` to prevent external terminals like tmux from rendering intermediate states.

### Performance Considerations of Double Buffering

Terminal output is I/O-intensive; `write()` system calls have fixed overhead. The double-buffering diff strategy reduces per-frame write volume from O(screen size) to O(changed region). Measured comments show: drain-only frames (DECSTBM scroll only, no React commit) produce ~10 patches and ~200 bytes of output (`ink.tsx:756`), while full repaints require several thousand bytes.

The `prevFrameContaminated` flag is a correctness safety net: when operations like selection highlighting overlays, `resetFramesForAltScreen()`, or `forceRedraw()` modify frontFrame content, the next frame must do a full diff rather than relying on incremental blit optimization (`ink.tsx:743`).

### Motivation for Vim Mode Design

Claude Code's core user base consists of heavy Vim users — for them, Vim keybindings in the prompt box are not a nice-to-have but a basic requirement. The pure state machine implementation (no dependencies, no side effects) makes Vim logic fully unit-testable and decoupled from the terminal rendering layer.

The comment at the beginning of `types.ts` reveals the design philosophy: **types as documentation**. The `CommandState` union type describes the state machine more precisely than any comment — TypeScript's switch exhaustive checking guarantees every state is handled, and missing a new state will cause a compile error (`types.ts:1`).

---

## 12.8 Transferable Patterns

The following design patterns can be borrowed for other terminal applications or UI frameworks:

**1. Shared Pool + Integer ID Screen Buffer**
Store integer IDs for characters/styles in `Int32Array` rather than strings or objects. Cross-frame shared Pools reduce diffs to integer comparisons, completely eliminating GC pressure. Applicable to any rectangular matrix scenario requiring high-frequency diffing.

**2. Types as State Machine**
Define state machines using TypeScript union types; each state carries exactly the data it needs; state transition functions use exhaustive switch. The `{next?, execute?}` two-key return value of `transition()` + `TransitionResult` is extremely concise and can be directly transplanted to other key handling scenarios.

**3. Damage Rectangle-Driven Incremental Diff**
Record the actually written rectangular region (`damage`) during rendering; diff only scans this region. Applicable to any frame buffer system — not just terminals; games UI and embedded displays can also use this pattern.

**4. Double Buffering + prevFrameContaminated Safety Net**
When certain operations don't go through the normal rendering path (directly modifying the front frame), use a bool flag to force a full diff on the next frame, rather than breaking the double-buffering invariant. More efficient than always doing full repaints, more correct than allowing dirty data.

**5. Render Microtask Delay**
The `scheduleRender = throttle(() => queueMicrotask(onRender), 16)` pattern ensures React layout effects fully commit before rendering, avoiding single-frame cursor position lag. Equally applicable in other React + custom renderer scenarios.

**6. Line Width Cache (Streaming Scenario Specific Optimization)**
Computing `stringWidth` for large numbers of unchanged lines during streaming output is a hotspot. `line-width-cache.ts`'s simple Map + clear-when-full strategy solves `~50x` computation overhead in 20 lines of code (from the original comment), and can be directly transplanted to any streaming text rendering scenario.

---

## 12.9 Source Index

| File | Lines | Core Responsibility |
|------|-------|-------------------|
| `src/ink/ink.tsx` | 1,722 | Ink class: frame management, render scheduling, SIGCONT, alt-screen handoff, mouse selection, search highlighting |
| `src/ink/screen.ts` | 1,300+ | Screen type, Int32Array cell layout, CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate: prev/next frame diff engine, DECSTBM hardware scrolling |
| `src/ink/optimizer.ts` | ~100 | Diff single-pass optimizer |
| `src/ink/reconciler.ts` | 512 | React Custom Renderer host implementation |
| `src/ink/render-node-to-output.ts` | 1,462 | Render DOM nodes to Screen |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | Line width LRU cache (4,096 entries) |
| `src/ink/output.ts` | ~300 | Output renderer, character cache (16,384 entries) |
| `src/vim/types.ts` | ~150 | VimState, CommandState, PersistentState type definitions and constants |
| `src/vim/transitions.ts` | ~350 | `transition()` main entry, 11 state transition functions |
| `src/vim/motions.ts` | ~80 | `resolveMotion()` cursor movement pure functions |
| `src/vim/operators.ts` | ~500 | delete/change/yank/x/r/~/J/paste/indent/openLine execution functions |
| `src/vim/textObjects.ts` | ~220 | Word/quote/bracket text object boundary finding |
| `src/screens/REPL.tsx` | 5,005 | Main REPL interface, session management, message streaming, fullscreen/normal mode switching |
| `src/components/PromptInput/PromptInput.tsx` | 2,338 | Input component, Vim/Standard mode routing, cursor management, STT injection |
| `src/components/Settings/Config.tsx` | 1,821 | Settings interface, search navigation, multi-source config read/write, Escape restore |
| `src/components/LogSelector.tsx` | 1,574 | Session history selection, Fuse.js search, tree-shaped fork display |

**Key Constants Reference**:
- `FRAME_INTERVAL_MS = 16` (`constants.ts:1`)
- `MAX_CACHE_SIZE = 4096` (`line-width-cache.ts`)
- `charCache` limit `16384` (`output.ts:204`)
- `MAX_VIM_COUNT = 10000` (`vim/types.ts:96`)
