# Chapter 1: Architecture Overview and Startup Flow

> Data source: Claude Code TypeScript source snapshot (2026-03-31)
> Source path (mini): `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 Overview and Positioning

**What Claude Code is:** Claude Code is an AI programming assistant running in the terminal, rendering an interactive TUI (Terminal User Interface) via React/Ink, driven by a REPL loop to call the Claude API for completing development tasks like code editing, command execution, and file operations.

### Tech Stack Overview

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Runtime | Bun (primary) / Node.js 18+ (compatible) | JavaScript runtime |
| Language | TypeScript | Strict typing across the entire project |
| UI Framework | React + Ink | Terminal TUI rendering |
| CLI Framework | Commander.js (`@commander-js/extra-typings`) | Command-line argument parsing |
| API Client | `@anthropic-ai/sdk` | Claude API calls |
| MCP Integration | `@modelcontextprotocol/sdk` | MCP server protocol |
| Feature Flags | GrowthBook + `bun:bundle` feature flags | A/B testing and DCE |
| Telemetry | OpenTelemetry (lazy-loaded ~400KB) | Metrics/logs/traces |
| Validation | Zod v4 | Runtime schema validation |

### Code Scale Statistics

- **Total lines**: 512,664 lines (`.ts` + `.tsx` files)
- **File count**: 1,884 TypeScript files
- **Top-level directories**: 35

LOC share by major directory:

```
utils/       180,472 lines  (35.2%)  — utility functions, permissions, auth, settings, etc.
components/   81,546 lines  (15.9%)  — React UI components
services/     53,680 lines  (10.5%)  — API, MCP, analytics, memory, etc.
tools/        50,828 lines  (9.9%)   — 30 tool implementations (Bash/File/Agent, etc.)
commands/     26,428 lines  (5.2%)   — slash command implementations
screens/       5,977 lines  (1.2%)   — REPL and other top-level screens
bootstrap/     ~5,000 lines  (1.0%)  — global state (state.ts at 1,758 lines)
entrypoints/   ~3,000 lines  (0.6%)  — CLI/SDK/MCP entry points
main.tsx       4,683 lines  (0.9%)   — main entry coordinator
setup.ts         477 lines  (0.1%)   — initialization setup
```

---

## 1.2 Theoretical Foundations

### Architectural Patterns for CLI Applications

Claude Code blends two classic CLI architectural patterns:

**REPL Loop (Read-Eval-Print Loop)**
Traditional REPLs read input, evaluate, and print output in a synchronous loop. Claude Code upgrades this to an asynchronous event-driven REPL: input is captured by React components, "evaluation" is a Claude API round-trip (including multiple rounds of tool calls), and output is rendered to the terminal through the React/Ink reconciler.

**Event-Driven Architecture**
Startup does not block waiting for all initialization to complete — MDM reads, Keychain prefetches, MCP connections, and plugin hook loading are all triggered in parallel in a fire-and-forget manner (see Section 1.4). This minimizes TTFR (Time To First Render), consistent with the Critical Rendering Path optimization philosophy in web applications.

### Design Philosophy of the Terminal UI Framework: React in Terminal

Ink ports React's component model, declarative state, and reconciliation mechanism to the terminal. Core ideas:

- **Virtual DOM → virtual terminal buffer**: each state change triggers a diff, only repainting changed character rows, avoiding flicker
- **Flexbox → terminal layout**: using the CSS Yoga engine to calculate column widths and line breaks, allowing terminal UIs to be described declaratively with JSX
- **Component reuse**: loading spinners, confirmation dialogs, diff displays, and other UI logic are encapsulated as testable React components

This means Claude Code's UI code shares the same cognitive framework as web frontend code, and the 81,546 lines in the `components/` directory can be understood using familiar React patterns.

### Theoretical Foundations of Plugin Architecture

Claude Code's plugin system is based on the Capability Registration Pattern:

- Tools, Commands, and Hooks are all registered in a global registry at startup
- Plugins extend the tool/command list via filesystem conventions (`~/.claude/plugins/`)
- `bun:bundle`'s `feature()` function performs Dead Code Elimination (DCE) at compile time, so experimental features don't appear in external build artifacts

---

## 1.3 Overall Architecture

### Layered Architecture (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                    Entry Layer                           │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (CLI interaction) (Commander.js routing) (MCP server mode)│
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Bootstrap Layer                          │
│    setup.ts      │    entrypoints/init.ts                 │
│  (session init)   │    bootstrap/state.ts                 │
│  (worktree/tmux)  │    (global state singleton)           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  UI Layer (Ink/React TUI)                 │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (main UI)          │  components/ (81K LOC)              │
│  replLauncher.tsx  │  (input/output/dialogs/loading)      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Engine Layer                             │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (session lifecycle)│  (API calls) │  (React state tree)  │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Tool Layer                               │
│  tools/ (30 tools, 50K LOC)                              │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Service Layer                            │
│  services/ (53K LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (MCP client)    (GrowthBook/OTel)         │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (language server) (session memory) (enterprise config)    │
└─────────────────────────────────────────────────────────┘
```

### Module Dependency Overview

```
main.tsx
  ├── entrypoints/init.ts       (memoized, initialized only once)
  ├── entrypoints/cli.tsx       (Commander sub-command routing)
  ├── bootstrap/state.ts        (global state, circular deps strictly forbidden)
  ├── setup.ts                  (called on each session)
  ├── QueryEngine.ts            (headless/SDK path)
  ├── replLauncher.tsx          (interactive path)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (MCP tool/resource loading)
```

**The special status of `bootstrap/state.ts`**: The code contains an explicit comment `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`, and there's an ESLint rule `custom-rules/bootstrap-isolation` preventing this file from being imported by non-leaf modules, guarding against circular dependencies.

### Three Entry Points Compared

| Entry | File | Trigger | Characteristics |
|-------|------|---------|-----------------|
| CLI interactive | `entrypoints/cli.tsx` | `claude` command | Full REPL + React TUI |
| SDK headless | `QueryEngine.ts` | `-p` flag / SDK API | No UI, single or streaming output |
| MCP server | `entrypoints/mcp.ts` | `claude --mcp` | Exposes tool set as MCP server |

---

## 1.4 Startup Flow in Detail

### main.tsx Complete Startup Sequence

`main.tsx`'s 4,683 lines are not executed sequentially — the import side effects at the top of the file are a carefully orchestrated parallel warm-up sequence.

**Phase 0: Module loading (import side effects, ~135ms)**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. performance baseline start

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. parallel: MDM subprocess (plutil/reg query)

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. parallel: macOS Keychain prefetch (OAuth + API key)

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // all imports complete
```

Comments precisely explain the benefit of these three parallel operations: MDM reads save ~135ms of module evaluation time, Keychain prefetch saves ~65ms of sequential sync spawn. This is the core trick of Claude Code's startup optimization: **leveraging ES module's static analysis to run I/O-intensive operations during module graph evaluation**.

**Phase 1: Commander routing (synchronous)**

In `entrypoints/cli.tsx`, Commander.js parses argv and dispatches to different execution paths based on sub-commands (`chat`, `api`, `mcp`, `resume`, etc.) or flags:

```typescript
// entrypoints/cli.tsx (simplified structure)
async function main(): Promise<void> {
  // Fast path: --version with zero imports
  // Normal path: await init() → setup() → branch execution
}
```

**Phase 2: init() initialization (memoized, executed only once)**

The `init` function in `entrypoints/init.ts` is wrapped with `memoize`, ensuring multiple calls only initialize once:

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // activate configuration system
  applySafeConfigEnvironmentVariables()  // safe env vars before trusted conversation
  applyExtraCACertsFromConfig()     // set CA certs before any TLS connection
  setupGracefulShutdown()           // register exit cleanup hooks
  // lazy loading: OpenTelemetry (~400KB) + gRPC (~700KB)
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // async cache
  detectCurrentRepository()          // GitHub repo detection
  preconnectAnthropicApi()           // TCP+TLS preconnect (~100-200ms overlap)
  configureGlobalMTLS()
  configureGlobalAgents()            // proxy configuration
})
```

**Phase 3: setup() session initialization (called on each session)**

```typescript
// setup.ts — critical step sequence
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. UDS messaging server (swarm/ant mode)
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Terminal backup check (iTerm2/Terminal.app)
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — must be before all code that depends on cwd
  setCwd(cwd)
  // 4. Hooks config snapshot (must be after setCwd)
  captureHooksConfigSnapshot()
  // 5. Worktree creation (if --worktree)
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. Background task registration (SessionMemory, context collapse)
  if (!isBareMode()) initSessionMemory()
  // 7. Plugin prefetch (parallel, non-blocking)
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. Analytics sink activation + first telemetry event
  initSinks()
  logEvent('tengu_started', {})
  // 9. Release notes check (interactive mode)
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**Phase 4: REPL rendering**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // lazy load UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

Finally Ink takes over the terminal, the React component tree begins rendering, and the REPL is ready.

### Parallel Prefetch Strategy

Claude Code's startup optimization follows the "**trigger as early as possible, wait as late as possible**" principle:

| Operation | Trigger point | Wait point |
|-----------|--------------|------------|
| MDM subprocess (`plutil/reg query`) | `main.tsx` line 1 import side effect | Before `applySafeConfigEnvironmentVariables()` call |
| Keychain prefetch (OAuth + API key) | `main.tsx` line 3 import side effect | `ensureKeychainPrefetchCompleted()` |
| Claude API TCP preconnect | `preconnectAnthropicApi()` inside `init()` | Connection automatically reused on first API request |
| Plugin hooks loading | fire-and-forget inside `setup()` | `processSessionStartHooks()` before rendering |
| MCP configs reading | `getClaudeCodeMcpConfigs()` kick-off | `getMcpToolsCommandsAndResources()` in interactive mode |

### Lazy Loading Mechanism

Claude Code explicitly lazy-loads large modules on the startup critical path:

```typescript
// entrypoints/init.ts — OpenTelemetry lazy loading comment
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

Additionally, `replLauncher.tsx` only `import`s App and REPL components at the last moment, preventing the React tree from being evaluated before Commander routing is complete.

`bun:bundle`'s `feature()` function implements compile-time DCE:

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

In external builds, these code blocks are completely eliminated, reducing bundle size.

### setup.ts Initialization Steps in Detail

`setup.ts`'s 477 lines revolve around the following key constraints:

1. **`setCwd()` must be called first**: all subsequent operations (hooks, settings, plugin loading) depend on the correct cwd
2. **Hooks snapshot must come after `setCwd()`**: ensures `.claude/settings.json` is read from the correct directory
3. **Worktree creation before `getCommands()`**: otherwise the `/eject` command is unavailable
4. **`initSinks()` after all background task registration**: ensures the analytics event queue is ready

`--bare` mode (scripted/SDK headless calls) skips many interactive features: terminal backup checks, plugin hook prefetch, commit attribution, team memory watcher, etc., minimizing startup overhead for script calls.

### bootstrap/state.ts State Construction

`state.ts` (1,758 lines) maintains the global singleton state for the entire session. The core `State` type covers:

```typescript
// bootstrap/state.ts (State type definition, partial)
type State = {
  originalCwd: string
  projectRoot: string          // stable project root, worktree doesn't change it
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // telemetry counters
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // log/trace providers
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... ~60 fields total
}
```

**Design constraint**: The comment `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` is an architectural guard. The ESLint rule `custom-rules/bootstrap-isolation` prevents state.ts from being imported by modules that would cause circular dependencies. All state is accessed through setter/getter functions; mutable objects are never directly exposed.

---

## 1.5 Entry Point Analysis

### CLI Entry (interactive mode)

`entrypoints/cli.tsx` is the most complex entry point, handling all user-facing feature routing:

**Startup path**:
1. Commander.js parses argv → identifies sub-commands or flags
2. `await init()` initialization (memoized)
3. Handle MCP configs, enterprise policy, Chrome integration
4. `await setup(cwd, permissionMode, ...)` session initialization
5. Branch based on mode:
   - **Interactive mode**: `showSetupScreens()` → `launchRepl()` → React TUI
   - **Print mode (`-p`)**: `runHeadless()` → `QueryEngine` → stdout
   - **Resume mode**: `loadConversationForResume()` → restore historical session
   - **Teleport mode**: remote session takeover

**Key CLI options** (partial):

| Flag | Function |
|------|---------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | dynamic MCP server configuration |
| `--worktree` | create git worktree isolation |
| `--tmux` | run in tmux session |
| `--model` | override main loop model |
| `--resume` | restore historical session |

### SDK Entry (programmatic API)

When called via the `-p` flag or SDK programmatic API, bypasses the React TUI and goes directly into `QueryEngine.ts`:

- `isNonInteractiveSession = true`
- Skips all UI rendering (Ink)
- Streams structured output to stdout via `SDKMessage` types
- Supports `SDKStatus`, `SDKPermissionDenial`, `SDKCompactBoundaryMessage`, and other structured outputs

SDK mode also has exclusive beta features: `entrypoints/sdk/coreSchemas.ts` defines structured JSON input/output schemas, and `entrypoints/agentSdkTypes.ts` defines SDK-specific types like `HookEvent` and `ModelUsage`.

### MCP Entry (MCP server mode)

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools: expose all Claude Code tools as MCP tools
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool: proxy execution to the corresponding Tool implementation
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

MCP mode exposes Claude Code's entire tool set (BashTool, FileReadTool, GrepTool, etc.) back to external MCP clients, implementing "Claude Code as MCP server."

### Shared Logic Across Three Entry Points

Regardless of which entry point is used, all share:
- `bootstrap/state.ts` global state
- `entrypoints/init.ts` initialization (memoized to execute only once)
- `Tool.ts` tool registry
- All services under `services/` (API client, permission system, etc.)
- Hooks lifecycle system

The difference lies in whether React TUI is rendered and the output format (interactive text vs. structured JSON).

---

## 1.6 Design Decision Analysis

### Why Bun Instead of Node.js

Observable Bun usage characteristics from the code:

1. **`bun:bundle`'s `feature()` function**: This is a Bun-specific compile-time feature flag mechanism that supports Dead Code Elimination. Used extensively in `main.tsx` (COORDINATOR_MODE, KAIROS, CHICAGO_MCP, UDS_INBOX, etc.); external builds completely eliminate this experimental code.

2. **Bun's WebView API** (conditional reference): `typeof Bun !== 'undefined' && 'WebView' in Bun`, indicating some features depend on Bun-specific APIs.

3. **Bun's single-file executable**: Comments mention `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`, indicating the release artifact is a Bun-compiled single-file executable.

4. **Performance**: Bun's startup speed and module loading speed are significantly faster than Node.js, which is critical for CLI tool TTFR.

Node.js 18+ compatibility is maintained (there's a Node version check in `setup.ts`) to support non-Bun environments (CI, enterprise-managed machines).

### Why React/Ink for Terminal UI

The `components/` directory's 81,546 lines of code indicate extremely high UI complexity. If written by hand with raw ANSI control codes, maintenance costs would be unmanageable. The React/Ink choice brings:

1. **Declarative UI**: streaming output, tool execution status, permission confirmation dialogs, etc. can all be driven by React state rather than imperative cursor control
2. **Component isolation**: `screens/REPL.tsx` only needs to handle overall layout; each sub-feature (input box, message list, tool progress) is self-encapsulated
3. **Hot-reload friendly**: debuggable with standard React DevTools thinking during development
4. **Testability**: React components can have unit tests written with `@testing-library/react`, without depending on a real terminal

### Performance Optimization Philosophy of Parallel Prefetch

Claude Code's startup optimization has a clear priority model: **TTFR (Time To First Render) comes first, not "all initialization complete"**.

Specifically:
- Keychain reads (~65ms) are triggered on the first line of import side effects, not when the API key is needed
- MCP server connections happen in the background in parallel; REPL rendering doesn't wait (users see the interface before MCP connections complete)
- Release notes, GrowthBook configuration, plugin hooks are all fire-and-forget

The trade-off is the need to carefully manage race conditions for "consumed before prefetch completes," precisely controlled through await points like `ensureKeychainPrefetchCompleted()`.

### Lazy Loading vs. Preloading Tradeoffs

| Strategy | Target | Rationale |
|----------|--------|-----------|
| Preloading (import side effects) | MDM subprocess, Keychain | I/O-intensive, earlier is better |
| Lazy loading (`await import()`) | OpenTelemetry (~400KB), gRPC (~700KB), React TUI components | Module evaluation is expensive, not on critical path |
| Conditional loading (`feature()` DCE) | COORDINATOR_MODE, KAIROS, CHICAGO_MCP | Experimental features, not needed by external users |
| `setImmediate()` delay | commit attribution hook | Avoid blocking event loop during setup() microtask window |

This layered strategy ensures Claude Code only does "the minimum work needed to display the interface" at startup.

---

## 1.7 Transferable Patterns

### Universal Startup Optimization Pattern

Claude Code's startup sequence demonstrates a reusable three-layer optimization framework of "**parallel warm-up + lazy loading + DCE**":

**Pattern 1: Use ES module side effects for I/O warm-up**
```typescript
// Insert fire-and-forget I/O between import statements
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // trigger immediately, no await
import { SomethingElse } from './other.js'  // parallel loading
```
Applicable to: any initialization data that "must be read but reading is slow" (config files, credentials, network preconnects).

**Pattern 2: memoize single initialization**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
Applicable to: initialization logic shared by multiple entry points, preventing repeated execution.

**Pattern 3: `--bare` mode layering**
Scripted/API calls don't need UI, terminal checks, analytics, etc.; use `isBareMode()` to quickly skip these, keeping headless call overhead low.

**Pattern 4: State isolation**
`bootstrap/state.ts` as a strict leaf module (no circular dependencies), accessed via setters/getters, enforced by ESLint rules. This allows the state module to be safely imported anywhere.

### What Doramagic CLI Can Learn

Based on the above analysis, Doramagic CLI can adopt the following patterns in its architectural design:

1. **Separate startup critical paths**: strictly separate "must complete before rendering" and "can complete after rendering" initialization; annotate the rationale with comments (following the style of Claude Code's `// ~65ms on every macOS startup` comments)

2. **Global state singleton + accessor pattern**: reference `bootstrap/state.ts`, maintain session state in a strict leaf module, avoid state scattered everywhere

3. **`memoize` initialization functions**: regardless of which entry point calls, ensure initialization executes only once

4. **Three mode separation**: interactive (React TUI) / headless (-p flag) / server (MCP), sharing the underlying tool and service layers

5. **Feature flag + DCE**: wrap experimental features with feature flags, automatically eliminated at release time

---

## 1.8 Source Index

| File | LOC | Key Content |
|------|-----|------------|
| `main.tsx` | 4,683 | Main entry, Commander routing, state initialization, interactive/headless branching |
| `setup.ts` | 477 | Session initialization: cwd, hooks, worktree, plugin prefetch |
| `bootstrap/state.ts` | 1,758 | Global state singleton, `State` type definition, all getters/setters |
| `entrypoints/init.ts` | ~400 | memoized global initialization: config, mTLS, proxy, OTel lazy loading |
| `entrypoints/cli.tsx` | ~2,000 | Commander.js routing, interactive/print/resume/teleport branching |
| `entrypoints/mcp.ts` | ~200 | MCP server mode, exposes tool set |
| `entrypoints/sdk/coreSchemas.ts` | - | SDK mode structured input/output schema |
| `entrypoints/agentSdkTypes.ts` | - | SDK-specific types (HookEvent, ModelUsage, etc.) |
| `replLauncher.tsx` | ~30 | Lazy-load App + REPL, start React TUI |
| `QueryEngine.ts` | ~1,500 | Session lifecycle management, headless path core |
| `Tool.ts` | - | Tool interface definition (inputSchema, call, prompt, etc.) |
| `tools/` | 50,828 | 30 tool implementations (BashTool/FileEditTool/AgentTool, etc.) |
| `services/api/` | - | Claude API calls, retry, usage statistics |
| `services/mcp/client.ts` | - | MCP client connection management |
| `utils/startupProfiler.ts` | - | `profileCheckpoint()` performance instrumentation |
| `utils/secureStorage/keychainPrefetch.ts` | - | macOS Keychain parallel prefetch |
| `utils/settings/mdm/rawRead.ts` | - | MDM configuration parallel read |

### Key Code Locations

- **Parallel warm-up start**: `main.tsx:12-20` (3 import side effects)
- **memoized initialization**: `entrypoints/init.ts:57` (`export const init = memoize(...)`)
- **Global state type**: `bootstrap/state.ts:30-200` (`type State = {...}`)
- **MCP server definition**: `entrypoints/mcp.ts:42` (`startMCPServer`)
- **REPL rendering entry**: `replLauncher.tsx:14` (`launchRepl`)
- **Tool interface**: `Tool.ts:1-30` (`ToolInputJSONSchema`, `ToolUseContext`)
- **setup critical order**: `setup.ts:77-230` (setCwd → captureHooksConfigSnapshot → worktree → background jobs)

---

*Chapter word count: ~9,800 characters | Source snapshot date: 2026-03-31*
