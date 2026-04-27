# Chapter 7: Command System

## 7.1 Overview and Positioning

Claude Code's command system is the core entry point for user interaction with the REPL. Every time a user types `/` in the input box, this system is triggered. It serves three roles:

1. **UI control layer**: directly operates terminal interface state without going through the LLM (e.g., `/clear`, `/theme`, `/vim`)
2. **Session management layer**: manages conversation history, context compression, and recovery (e.g., `/compact`, `/resume`, `/branch`)
3. **Capability extension layer**: delegates complex tasks to the model for execution through the Prompt expansion mechanism (e.g., `/review`, `/skills`)

The boundary design of the command system reflects clear separation of concerns: commands handle "triggering," tools (Tools) handle "execution," and the LLM handles "decision-making." A `/review` command doesn't directly call git; instead, it injects the review prompt into the conversation flow, letting the model drive the subsequent tool call chain.

---

## 7.2 Theoretical Foundations

### Command Pattern

The system's design closely aligns with the classic GoF Command pattern:

- **Command interface**: the `Command` union type (`PromptCommand | LocalCommand | LocalJSXCommand`), uniformly encapsulating requests
- **ConcreteCommand**: each `commands/<name>/index.ts` file is a concrete command implementation
- **Invoker**: REPL's `processSlashCommand` is responsible for dispatch and execution
- **Receiver**: `ToolUseContext` (conversation state), `AppState` (application state) are the objects being operated on

But Claude Code makes two key extensions to the classic pattern:

**Lazy loading**: commands are implemented with delayed loading via `load(): Promise<Module>` rather than instantiation at registration time. This distributes startup overhead across first invocations, which is significant for commands with heavy dependencies (like `/insights`'s 113KB HTML rendering module).

**Typed return values**: commands don't have void return values (void actions), but return structured results (`LocalCommandResult`), leaving the upper-level REPL to decide how to render, achieving decoupling of execution and presentation.

### Design Patterns for REPL Command Processing

Claude Code's REPL command processing follows two core principles:

**Immediate vs. Queued**: the `immediate?: boolean` field on command objects determines whether the command bypasses the message queue for immediate execution. UI operations like `/clear` and `/exit` need immediate response, while operations involving API calls like `/compact` enter the queue for ordered processing.

**Auth-gated availability**: unlike runtime feature flags (`isEnabled()`), the `availability` field takes effect at the command list filtering stage, ensuring unauthorized users don't even see specific commands (e.g., commands limited to claude.ai subscribers).

---

## 7.3 Command Registration Mechanism

### Registration Flow in commands.ts

The core registration logic is concentrated in `commands.ts` (754 lines), organized in four layers:

**First layer: Static built-in commands**

```typescript
// commands.ts:240-310 (core snippet)
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... ~60 built-in commands
])
```

The `COMMANDS` function is wrapped in `memoize` rather than a module-level constant, because some commands need to read configuration files at registration time, and the configuration isn't available during module initialization.

**Second layer: Feature-flag conditional commands**

```typescript
// commands.ts:68-112 (conditional import snippet)
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

These commands undergo dead code elimination via `bun:bundle`'s `feature()` function, directly pruning disabled commands at build time rather than runtime judgment.

**Third layer: Internal-only commands**

```typescript
// commands.ts:197-222 (INTERNAL_ONLY_COMMANDS)
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, ultraplan, subscribePr, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
].filter(Boolean)
```

These commands are only registered when `USER_TYPE === 'ant'` (Anthropic internal users) and not in demo mode — an isolation mechanism for internal tools and debug commands.

**Fourth layer: Dynamically loaded commands**

```typescript
// commands.ts:360-395 (loadAllCommands)
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

Skills, plugin commands, and workflow commands are all loaded asynchronously in parallel, arranged by priority: bundled skills have the highest priority, built-in commands have the lowest. This ensures user-defined commands can shadow (override) same-named built-in commands.

### Command Type Definitions

`types/command.ts` defines three mutually exclusive command types, forming the `Command` union type:

```typescript
// types/command.ts (core union type)
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| Type | Description | Typical Commands |
|------|-------------|-----------------|
| `PromptCommand` | Expands to prompt injected into conversation flow, executed by model | `/review`, `/skills`, all Skills |
| `LocalCommand` | Pure local synchronous execution, returns text result | `/compact`, `/context` |
| `LocalJSXCommand` | Renders Ink React UI component | `/model`, `/resume`, `/config` |

`CommandBase` is the set of shared base fields for all three:

```typescript
// types/command.ts (CommandBase core fields)
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // runtime feature flag check
  isHidden?: boolean                      // hidden from typeahead
  argumentHint?: string                   // parameter hint text
  whenToUse?: string                      // model invocation scenario description
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // bypass queue for immediate execution
  isSensitive?: boolean                   // sanitize arguments from history
}
```

### Command Classification (Built-in vs. Plugin vs. User-defined)

```
Command source hierarchy (highest to lowest priority)
├── bundledSkills        # Built-in Skills packaged with Claude Code
├── builtinPluginSkills  # Skills provided by enabled built-in plugins
├── skillDirCommands     # Skills in user's .claude/skills/ directory
├── workflowCommands     # Workflow script commands (feature: WORKFLOW_SCRIPTS)
├── pluginCommands       # Commands registered by third-party plugins
├── pluginSkills         # Skills provided by third-party plugins
└── COMMANDS()           # Hard-coded built-in commands (lowest priority)
```

---

## 7.4 Complete Command Catalog

The following is organized based on `commands.ts`'s `ls` output and registration list.

### Session Management

| Command | Description |
|---------|-------------|
| `/compact [instructions]` | Compress conversation history, free context window |
| `/resume` | Select and restore conversation from history session list |
| `/branch [title]` | Fork a new session from current conversation |
| `/rewind` | Rewind to a historical node in the conversation |
| `/clear` | Clear current conversation records |
| `/session` | Display current session info |
| `/rename` | Rename current session |
| `/summary` | Generate current conversation summary (internal command) |
| `/export` | Export conversation content |
| `/copy` | Copy last message to clipboard |

### Development Tools

| Command | Description |
|---------|-------------|
| `/review [PR#]` | Local code review (calls `gh pr diff`) |
| `/ultrareview [PR#]` | Cloud-based deep code review (10-20 min, bughunter-driven) |
| `/commit` | Commit code changes (internal command) |
| `/commit-push-pr` | Commit + Push + create PR (internal command) |
| `/diff` | View current git diff |
| `/init` | Initialize project (generate CLAUDE.md) |
| `/add-dir` | Add extra working directory |
| `/hooks` | Manage event hook configuration |
| `/files` | List files tracked in session |
| `/pr_comments` | View PR comments |
| `/issue` | Create/view GitHub Issue (internal command) |
| `/autofix-pr` | Auto-fix issues in PR (internal command) |

### Configuration

| Command | Description |
|---------|-------------|
| `/model [name]` | Switch conversation model (with interactive selector) |
| `/config` | View/modify configuration items |
| `/theme` | Switch terminal theme |
| `/vim` | Toggle vim input mode |
| `/keybindings` | Manage keybinding shortcuts |
| `/permissions` | View/modify tool permissions |
| `/privacy-settings` | Manage privacy settings |
| `/output-style` | Set output format preference |
| `/effort` | Set response effort level |
| `/fast` | Toggle fast mode |
| `/plan` | Toggle Plan mode (plan only, don't execute) |
| `/sandbox-toggle` | Toggle sandbox mode |

### Debug and Diagnostics

| Command | Description |
|---------|-------------|
| `/doctor` | Diagnose configuration and environment issues |
| `/cost` | Display current session token consumption and fees |
| `/context` | Display context window usage details (by category in tables) |
| `/stats` | Display usage statistics |
| `/usage` | Display API usage info |
| `/insights` | Generate historical session usage analysis report (lazy-loads 113KB module) |
| `/heapdump` | Generate memory heap snapshot (for debugging) |
| `/debug-tool-call` | Debug tool calls (internal command) |
| `/perf-issue` | Record performance issues (internal command) |
| `/ant-trace` | Anthropic internal tracing (internal command) |

### Identity and Services

| Command | Description |
|---------|-------------|
| `/login` | Log in to Claude.ai account |
| `/logout` | Log out |
| `/upgrade` | Upgrade to higher plan |
| `/install-github-app` | Install GitHub App |
| `/install-slack-app` | Install Slack App |
| `/ide` | IDE integration management |
| `/terminalSetup` | Terminal integration configuration |
| `/mobile` | Display mobile connection QR code |
| `/chrome` | Chrome extension management |
| `/desktop` | Desktop app management |

### Advanced Features

| Command | Description |
|---------|-------------|
| `/mcp` | MCP server management (list/start/restart) |
| `/skills` | Skills management (list/install/update) |
| `/tasks` | Background task management |
| `/agents` | Sub-agent management |
| `/memory` | Project memory file management (CLAUDE.md) |
| `/plan` | Enter planning mode |
| `/thinkback` | Review model's thinking process |
| `/thinkback-play` | Play thinking review animation |
| `/advisor` | AI advisor mode |
| `/plugin` | Plugin management |
| `/reload-plugins` | Reload plugins |
| `/passes` | Multi-round review passes management |
| `/feedback` | Send feedback to Anthropic |
| `/btw` | Add annotation message |
| `/tag` | Tag conversation |
| `/stickers` | Display stickers (easter egg feature) |

Feature-flag conditional commands (invisible by default):

| Command | Feature Flag | Description |
|---------|-------------|-------------|
| `/ultraplan` | `ULTRAPLAN` | Cloud-based super planning (long async) |
| `/voice` | `VOICE_MODE` | Voice input mode |
| `/bridge` | `BRIDGE_MODE` | Remote control bridge mode |
| `/workflows` | `WORKFLOW_SCRIPTS` | Script workflow commands |
| `/peers` | `UDS_INBOX` | Peer session communication |
| `/fork` | `FORK_SUBAGENT` | Explicitly create sub-agent |
| `/buddy` | `BUDDY` | Buddy collaboration mode |

---

## 7.5 Command Execution Flow

### Complete Path from User Input "/" to Command Execution

```
User types "/compact some instructions"
        │
        ▼
    REPL input handler
    detects "/" prefix
        │
        ▼
    getCommands(cwd)                    ← aggregate command list from all sources
    findCommand("compact", commands)     ← find by name / aliases
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← check auth type gate
    isCommandEnabled(cmd)               ← check feature flag / isEnabled()
        │
        ├── check cmd.immediate          ← true: bypass queue for immediate execution
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        returns LocalCommandResult
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        render React component to terminal
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   returns ContentBlockParam[]
                                   inject into conversation flow → trigger model reasoning
```

### Command Argument Parsing

The command system does not have a built-in unified argument parsing framework — this is a deliberate design choice. Each command handles its `args: string` parameter independently, maintaining great flexibility:

- `/compact` uses `args.trim()` directly as custom compression instructions
- `/review` uses `/^\d+$/.test(prNumber)` to determine if it's a PR number
- `/model` goes to `SetModelAndClose` directly when args are provided, renders interactive `ModelPickerWrapper` when there are no args
- `/resume` supports session ID (UUID), custom title, or opens a list selector when no parameters

This design avoids complexity of a unified parsing layer, at the cost of each command needing to handle its own edge cases.

### Command Output Rendering

The three types of `LocalCommandResult` correspond to different rendering paths:

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // render as text message
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // trigger context replacement logic
  | { type: 'skip' }                      // render nothing
```

`LocalJSXCommand` passes results to REPL via `onDone()` callback:

```typescript
// types/command.ts (LocalJSXCommandOnDone)
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // message display mode
    shouldQuery?: boolean                   // whether to immediately trigger model query
    metaMessages?: string[]                 // model-visible but user-invisible messages
    nextInput?: string                      // auto-fill next input
    submitNextInput?: boolean               // whether to auto-submit
  },
) => void
```

`display: 'system'` shows in system message style (gray italic), `display: 'user'` shows as a normal user message, `display: 'skip'` shows nothing.

---

## 7.6 In-Depth Analysis of Representative Commands

### /compact Command Implementation Details

`/compact` is one of the logically most complex commands in the command system, handling the core responsibility of conversation history compression.

**Execution decision tree** (`commands/compact/compact.ts`):

```
/compact [instructions]
    │
    ├── Custom instructions present?
    │   └── No instructions → trySessionMemoryCompaction()   ← try Session Memory compression first
    │                          success returns, fastest path
    │
    ├── isReactiveOnlyMode() ?
    │   └── Yes → compactViaReactive()               ← reactive compression path (new architecture)
    │               parallel: executePreCompactHooks + getCacheSharingParams
    │               call: reactiveCompactOnPromptTooLong()
    │
    └── No → traditional compression path
              microcompactMessages()                  ← micro-compress first to reduce tokens
              compactConversation()                   ← main compression (summary generation)
              setLastSummarizedMessageId(undefined)   ← reset tracking pointer
```

Key design point: before compression, must call `getMessagesAfterCompactBoundary(messages)` to filter out messages the REPL has kept for UI scrollback — these should not appear in the summary.

After successful compression, the cleanup sequence is fixed:
1. `setLastSummarizedMessageId(undefined)` — reset message pointer
2. `suppressCompactWarning()` — suppress "context near exhaustion" warning
3. `getUserContext.cache.clear?.()` — clear user context cache
4. `runPostCompactCleanup()` — trigger post-compress hooks

**The Reactive Compact path** uses parallel optimization:

```typescript
// compact.ts:compactViaReactive (core parallel section)
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // execute pre-compress hooks (may start subprocesses)
  getCacheSharingParams(context, messages),  // build system prompt (traverse all tools)
])
```

The two are mutually independent; parallel execution significantly reduces wait time.

### /model Command Model Switching Logic

`/model` is `local-jsx` type, rendering an interactive selector via a React component.

**Two execution paths**:

- **With parameters** (`/model claude-sonnet-4-6`): renders `SetModelAndClose` component; `useEffect` executes model validation asynchronously; completes immediately via `onDone()`
- **No parameters** (`/model`): renders `ModelPickerWrapper` component, shows the complete `ModelPicker` interactive interface

**Model switching state update**:

```typescript
// model.tsx:handleSelect (core state update)
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // clear session-level temporary override
}))
```

**Model validation hierarchy** (fastest to slowest):
1. Check `isModelAllowed(model)` — organization restriction whitelist
2. Check `isOpus1mUnavailable(model)` — 1M context privilege check
3. Check `isKnownAlias(model)` — known alias passes directly (skips API validation)
4. `validateModel(model)` — call API to validate custom model name

Fast Mode and model switching are linked: if the new model doesn't support Fast Mode, it's automatically disabled; if supported and already enabled, "Fast mode ON" is shown in the confirmation message.

### /review Command Code Review Flow

`/review` demonstrates typical usage of the `PromptCommand` type — driving a complete review flow with a concise prompt template:

```typescript
// review.ts:LOCAL_REVIEW_PROMPT (complete prompt template)
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

The command itself has only 4 key lines of code; everything else is done by the model — this is exactly the design philosophy of `PromptCommand`: **command defines WHAT, model decides HOW**.

In contrast, `/ultrareview` (`local-jsx` type) executes a completely different path:

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← check free quota / Extra Usage balance
    │   ├── Team/Enterprise → pass directly
    │   ├── has free quota → pass, with hint
    │   └── quota exhausted → show overage confirmation dialog
    │
    └── launchRemoteReview()
        ├── PR mode → teleportToRemote(branchName: "refs/pull/N/head")
        └── branch mode → git merge-base → git diff check → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → return task URL, model notifies user
```

`/ultrareview` "teleports" the code review task to run in the cloud; after registering `RemoteAgentTask` locally, returns immediately and receives results via polling — an async task delegation model, completely different from the synchronous execution model of local commands.

---

## 7.7 Boundary Between Commands and Skills

### Similarities and Differences

| Dimension | Command | Skill |
|-----------|---------|-------|
| Definition | TypeScript code, hard-coded logic | Markdown file, frontmatter + prompt content |
| Loading time | Statically registered at startup (built-in) or async loaded (plugin) | Scanned from filesystem at runtime |
| Execution type | `local` / `local-jsx` / `prompt` | Only `prompt` (expands to prompt) |
| Model invocable | Most built-in commands prevent model invocation (`source: 'builtin'`) | Designed to support model invocation via SkillTool |
| User visibility | All commands appear in `/` typeahead | Depends on `userInvocable` and `hasUserSpecifiedDescription` |
| Context awareness | Full application state via `ToolUseContext` | Can only use prompt content, no direct state access |
| Source identifier | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### Considerations Behind Design Choices

**Why not use Markdown Skills for built-in commands?**

Built-in commands need to access application state (`AppState`), call Node.js APIs (file system, crypto), render React components — capabilities far beyond what prompt templates can express. `/compact` needs to call 4 different compression strategies; `/model` needs to render an interactive UI; `/resume` needs to read/write session files. All of these must be code.

**SkillTool's filtering logic** reveals the precise delineation:

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← built-in commands are excluded
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`** is the core rule: built-in commands are explicitly excluded from the model-invocable list. This prevents the model from bypassing permission checks via SkillTool to directly manipulate session state.

**Remote Safe Commands (REMOTE_SAFE_COMMANDS)** further refines this boundary:

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

Only 20 commands are available in `--remote` mode — these commands don't depend on the local filesystem, git, or IDE; they're pure TUI state operations, safe to execute in remote bridge sessions.

---

## 7.8 Design Decision Analysis

**Decision 1: Three command types instead of a unified interface**

The three-way split of `local` / `local-jsx` / `prompt` may seem to add complexity, but each type solves a different core problem:
- `local` handles operations with side effects but no UI (needs to return structured data)
- `local-jsx` handles operations needing interactive interfaces (relies on Ink rendering tree)
- `prompt` handles operations that can be delegated to the model (lowest coupling)

Forcing a single interface would either require all commands to handle React rendering (unnecessary dependency) or lose type safety.

**Decision 2: memoize by cwd instead of global singleton**

`loadAllCommands = memoize(async (cwd: string) => ...)` uses working directory as the cache key, meaning different Claude Code instances in different directories have independent command caches. This supports monorepos and multi-project scenarios where each directory has an independent set of Skills.

**Decision 3: No unified argument parsing**

This is an intentional "loose design." A unified parsing framework (like commander.js) would force each command to declare complete parameter schemas, meaningless for commands like `/compact` that accept "free text instructions." Keeping the raw string lets commands decide how to parse, trading consistency for flexibility.

**Decision 4: Two-layer gating with Availability vs. isEnabled**

Two-layer gating solves visibility at different lifecycles:
- `availability` filters at command list construction time, results cached; suitable for static auth type checks
- `isEnabled()` re-evaluated on every `getCommands()` call (not cached); suitable for dynamic feature flag checks

Comments specifically explain why `isEnabled()` is not memoized: after `/login` executes, auth state changes and must immediately reflect in the command list.

**Decision 5: Not managing internal commands as separate packages**

`INTERNAL_ONLY_COMMANDS` directly controls visibility via environment variable `USER_TYPE === 'ant'` rather than through separate npm packages. This simplifies build complexity, at the cost of requiring dead code elimination to prune this code in external builds (the `filter(Boolean)` also works for `null` conditional commands).

---

## 7.9 Transferable Patterns

### Pattern 1: Three-Way Command Type Split

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**Applicable scenarios**: any command system that needs to simultaneously support "pure logic," "UI interaction," and "LLM delegation." The boundaries between the three are very clear; can be directly ported to other REPL/CLI frameworks.

**Core value**: the type system enforces separation of concerns; no runtime isinstance checks needed.

### Pattern 2: Lazy Loading + memoize by cwd

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

**Applicable scenarios**: CLI tools with many commands (>50) or some commands with heavy module dependencies.

**Implementation notes**: the memoize key must include all factors affecting the command set (here it's cwd); cache invalidation timing must correspond to actual state changes (here it's `clearCommandsCache()`).

### Pattern 3: Multi-Source Command Aggregation + Priority Sorting

```typescript
return [
  ...bundledSkills,       // highest priority (can override same-named built-in commands)
  ...pluginCommands,
  ...COMMANDS(),          // lowest priority (can be overridden)
]
```

**Applicable scenarios**: CLI tools supporting plugin ecosystems, where third-party extensions need to be able to override (override) built-in behavior.

**Notes**: `findCommand` returns the first match in the list, so array order is priority order; this needs to be clearly documented in the design.

### Pattern 4: Auth-gated Command Visibility

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai': if (isClaudeAISubscriber()) return true; break
      case 'console':   if (!isUsing3PServices() && ...) return true; break
    }
  }
  return false
}
```

**Applicable scenarios**: SaaS products needing to show different feature sets to users at different subscription tiers.

**Key design**: intercept at the command list filtering stage rather than reporting errors at execution — users don't see commands they can't use, reducing cognitive load.

### Pattern 5: Bridge Safe / Remote Safe Whitelist

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([session, exit, clear, ...])
```

**Applicable scenarios**: command systems needing to run in restricted environments (remote sessions, sandboxes, mobile bridges).

**Implementation thinking**: safer than a blacklist — new commands are not available in restricted environments by default and must be explicitly added to the whitelist. This prevents sensitive commands from accidentally being exposed in wrong environments due to oversight.

---

## 7.10 Source Index

| File Path | LOC | Content |
|-----------|-----|---------|
| `src/commands.ts` | 754 | All entry logic for command registration, aggregation, filtering, lookup |
| `src/types/command.ts` | ~250 | Command union type definition, CommandBase, detailed subtype declarations |
| `src/commands/compact/compact.ts` | 287 | /compact three-path implementation (session memory / reactive / traditional) |
| `src/commands/model/model.tsx` | 296 | /model interactive selector + direct-set two paths, React Compiler output |
| `src/commands/review.ts` | ~50 | /review (prompt type) and /ultrareview (local-jsx type) entry |
| `src/commands/review/reviewRemote.ts` | 316 | /ultrareview remote launch logic: teleport, overage gate, task registration |
| `src/commands/resume/resume.tsx` | 274 | /resume session list selector UI |
| `src/commands/branch/branch.ts` | 296 | /branch conversation forking: JSONL copy, sessionId rewrite, conflict handling |
| `src/commands/context/context-noninteractive.ts` | 325 | /context non-interactive path: categorized token stats, Markdown table rendering |
| `src/skills/loadSkillsDir.ts` | — | Skills directory scanning and dynamic loading logic |
| `src/skills/bundledSkills.ts` | — | Built-in Skills packaged with product registration |
| `src/plugins/builtinPlugins.ts` | — | Skill command extraction for built-in plugins |
| `src/utils/plugins/loadPluginCommands.ts` | — | Third-party plugin command loading and caching |

**Key function index**:

| Function | File | Purpose |
|----------|------|---------|
| `getCommands(cwd)` | commands.ts | Return all commands available to current user (main entry) |
| `findCommand(name, commands)` | commands.ts | Find command by name/alias |
| `meetsAvailabilityRequirement(cmd)` | commands.ts | Auth type gate check |
| `getSkillToolCommands(cwd)` | commands.ts | Return model-invocable Skill command set |
| `getSlashCommandToolSkills(cwd)` | commands.ts | Return user-triggerable Skill set via / |
| `isBridgeSafeCommand(cmd)` | commands.ts | Determine if command can execute in bridge mode |
| `formatDescriptionWithSource(cmd)` | commands.ts | Description formatting with source annotation in UI |
| `clearCommandsCache()` | commands.ts | Clear all command caches (including Skills and plugins) |
