# Chapter 3: Tool System

## 3.1 Overview and Positioning

Claude Code's tool system is the entire product's execution layer. The LLM is responsible for reasoning and decision-making, but the actual side effects — reading files, running commands, searching code, accessing the network — are all accomplished through the tool system. The tool system is the only channel between LLM intent and the real world.

In terms of scale, this is a quite large subsystem:
- The source snapshot shows tool directories totaling **40+ subdirectories**, covering file operations, code execution, Agent coordination, MCP integration, task management, and other categories
- The core abstraction file `Tool.ts` has 792 lines, the tool registration file `tools.ts` has 389 lines, and the tool execution engine `services/tools/toolExecution.ts` has 1,745 lines
- The tool result storage module `utils/toolResultStorage.ts` has 1,040 lines, independently handling token budget issues

This scale reveals one fact: **the tool system is not an accessory to Claude Code, but its core engineering asset**. The reliability, security, and extensibility of the entire product are largely determined by the design quality of the tool system.

The competitive analysis (cc-notebook) has no dedicated tool system chapter — an obvious analytical blind spot. This chapter fills that gap.

---

## 3.2 Theoretical Foundations

### Self-Describing Tools Pattern

In traditional API calls, the caller needs to know the interface specification in advance. Claude Code's tool system adopts a different design philosophy: **each tool self-describes its capabilities, input format, and usage constraints**.

This is embodied in several core fields of the `Tool` type:

```typescript
// Tool.ts:300-310 (simplified)
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // 3-10 word capability summary for ToolSearch keyword matching
  description(input, options): Promise<string>   // dynamically generated description
  prompt(options): Promise<string>               // tool's complete system prompt
  inputSchema: Input           // Zod schema, serving as both documentation and validator
  outputSchema?: z.ZodType
  // ...
}
```

`description()` and `prompt()` are async methods, meaning the tool's self-description can be **dynamically generated** — adjusting prompt content based on current permission context, installed tools, and environment state. This is not static documentation, but context-aware description generated at runtime.

### Plugin Architecture and Dependency Injection

The tool system is essentially a plugin architecture. Each tool is built with the `buildTool()` factory function, implementing the unified `Tool` interface while being completely decoupled from each other. Adding a new tool only requires:

1. Creating a tool directory (e.g., `tools/MyTool/`)
2. Implementing the `ToolDef` interface
3. Registering in `getAllBaseTools()` in `tools.ts`

Tools themselves don't depend on each other (circular dependencies are broken via lazy require), but all depend on `ToolUseContext` — a context object that runs through the entire execution chain, containing permission state, message history, application state, etc.

```typescript
// Tool.ts:167-172 (simplified)
export type ToolUseContext = {
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  contentReplacementState?: ContentReplacementState
  // ...
}
```

The design of `ToolUseContext` is classic dependency injection: all external dependencies needed during tool execution are passed in through context; tools themselves are stateless pure-functional components. This makes testing, isolation, and sub-agent execution all feasible.

### The Role of Function Calling in LLMs

Claude Code follows Anthropic API's Function Calling protocol. The LLM can output `tool_use` blocks during reasoning, specifying the tool name and parameters to call; execution results are returned to the LLM as `tool_result` blocks, serving as input for the next round of reasoning.

The key constraint of this loop is: **tool definitions (name + input schema) must be sent to the LLM in the system prompt**, consuming precious context tokens. As tool count grows beyond a threshold (experiments show ~40-60 tools), this overhead becomes non-negligible — directly motivating the ToolSearch lazy loading mechanism described in Section 3.6.

---

## 3.3 Architecture and Data Structures

### buildTool() Unified Abstraction

`buildTool()` is the core factory function of the tool system, defined at `Tool.ts:756-769`:

```typescript
// Tool.ts:756-769
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

It does one thing: merges the user-provided `ToolDef` (allowing optional fields to be omitted) with `TOOL_DEFAULTS` (safe default values), returning a complete `Tool`.

The defaults (`Tool.ts:729-742`) embody a **fail-closed** design philosophy:

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // default: not concurrency-safe
  isReadOnly: (_input?) => false,            // default: assume writes
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // default allow, handled by general permission system
  toAutoClassifierInput: (_input?) => '',    // default: skip safety classifier
  userFacingName: (_input?) => '',
}
```

Worth noting: `isConcurrencySafe` defaults to `false` — meaning the system prefers serial execution of two tools rather than risking concurrent execution of operations that may have side effects. Only tools that explicitly declare `isConcurrencySafe: () => true` (like GrepTool, GlobTool, and other read-only tools) will be scheduled in parallel.

### Tool Core Type Definition

The methods of the `Tool` interface can be divided into several functional domains (`Tool.ts:297-580`):

**Execution domain**
- `call(args, context, canUseTool, parentMessage, onProgress)` — core execution method, returns `Promise<ToolResult<Output>>`
- `validateInput(input, context)` — pre-execution validation, returns `ValidationResult`
- `checkPermissions(input, context)` — permission check, independent from the general permission system

**Description domain** (tool self-description capability)
- `description(input, options)` — brief description, used in the API's tools list
- `prompt(options)` — complete system prompt, telling the model how to use this tool
- `searchHint` — 3-10 word capability summary, specifically for ToolSearch keyword matching

**Rendering domain** (React components, REPL mode only)
- `renderToolUseMessage(input, options)` — UI when tool call starts
- `renderToolResultMessage(content, progressMessages, options)` — UI for tool results
- `renderToolUseProgressMessage(progressMessages, options)` — progress UI during execution
- `renderToolUseRejectedMessage(input, options)` — UI when rejected

**Metadata domain**
- `isConcurrencySafe(input)` — declares whether concurrent execution is safe
- `isReadOnly(input)` — declares whether read-only (affects permission judgment)
- `isDestructive(input)` — declares whether irreversible (deletion, overwrite, send)
- `shouldDefer` — whether to lazy-load (loaded on demand by ToolSearch)
- `alwaysLoad` — always loaded into prompt (not deferred)
- `maxResultSizeChars` — threshold for persisting tool results to disk

`ToolResult<T>`'s structure (`Tool.ts:289-298`) is also worth noting:

```typescript
// Tool.ts:289-298
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

`contextModifier` allows tools to modify the context after execution (but the comment explicitly notes: **only non-concurrency-safe tools will execute contextModifier** — an important concurrency safety constraint).

### Tool Registration and Discovery Mechanism

`getAllBaseTools()` in `tools.ts` is the single source of truth for tool registration (`tools.ts:108-186`). This function returns all built-in tools available in the current environment, controlling tool availability through multiple conditional layers:

**Environment conditions** (process.env):
- `USER_TYPE === 'ant'` — Anthropic internal tools (ConfigTool, TungstenTool, REPLTool)
- `NODE_ENV === 'test'` — testing tools (TestingPermissionTool)
- `ENABLE_LSP_TOOL` — LSP integration tool
- `CLAUDE_CODE_VERIFY_PLAN` — plan verification tool

**Feature Flag conditions** (`feature()` from `bun:bundle`):
- `PROACTIVE` / `KAIROS` — SleepTool (proactive behavior)
- `AGENT_TRIGGERS` — ScheduleCronTool and other scheduled task tools
- `COORDINATOR_MODE` — coordinator mode related tools
- `WEB_BROWSER_TOOL` — browser tool
- `WORKFLOW_SCRIPTS` — workflow tool

**Runtime conditions**:
- `isToolSearchEnabledOptimistic()` — whether to include ToolSearchTool
- `isTodoV2Enabled()` — whether to include task management tool set
- `isAgentSwarmsEnabled()` — whether to include team collaboration tools
- `hasEmbeddedSearchTools()` — if bfs/ugrep is built in, don't add GlobTool/GrepTool

Tool **deduplication and ordering** (`assembleToolPool()`, `tools.ts:218-248`) uses a carefully designed strategy: built-in and MCP tools are sorted separately then concatenated, with built-in tools as prefix and MCP tools appended at the end. This is to maintain system prompt stability (prompt cache stability) — Anthropic's servers set cache breakpoints at fixed positions; if built-in and MCP tools were mixed in ordering, any new MCP tool would break the cache.

---

## 3.4 Tool Catalog

Based on the `tools/` directory structure and `tools.ts` registration logic, here is the complete tool list:

### File Operations

| Tool | Directory | Function | Concurrency Safe |
|------|-----------|----------|-----------------|
| FileReadTool | `FileReadTool/` | Read files, supports PDF/images/Notebooks, paginated reading | Yes |
| FileEditTool | `FileEditTool/` | Precise string replacement, supports replace_all | No |
| FileWriteTool | `FileWriteTool/` | Write/create files | No |
| GlobTool | `GlobTool/` | Find files by glob pattern | Yes |
| GrepTool | `GrepTool/` | ripgrep regex content search | Yes |
| NotebookEditTool | `NotebookEditTool/` | Jupyter Notebook cell editing | No |

### Code Execution

| Tool | Directory | Function | Notes |
|------|-----------|----------|-------|
| BashTool | `BashTool/` | Shell command execution, supports background tasks, sandbox | Core tool |
| PowerShellTool | `PowerShellTool/` | Windows PowerShell execution | Conditionally enabled |
| REPLTool | `REPLTool/` | REPL execution in isolated VM environment | Ant internal |

### Agent Orchestration

| Tool | Directory | Function |
|------|-----------|---------|
| AgentTool | `AgentTool/` | Launch subagents, supports parallel execution |
| SendMessageTool | `SendMessageTool/` | Send messages to other Agents |
| TeamCreateTool | `TeamCreateTool/` | Create Agent teams |
| TeamDeleteTool | `TeamDeleteTool/` | Delete Agent teams |
| TaskCreateTool | `TaskCreateTool/` | Create background tasks |
| TaskGetTool | `TaskGetTool/` | Get task status |
| TaskUpdateTool | `TaskUpdateTool/` | Update task status |
| TaskListTool | `TaskListTool/` | List all tasks |
| TaskStopTool | `TaskStopTool/` | Stop tasks |
| TaskOutputTool | `TaskOutputTool/` | Get task output |

### Context and Tool Discovery

| Tool | Directory | Function |
|------|-----------|---------|
| SkillTool | `SkillTool/` | Load and execute Skills (~/.claude/skills/) |
| ToolSearchTool | `ToolSearchTool/` | Search deferred tools |
| MCPTool (dynamically generated) | `MCPTool/` | MCP server tools (dynamically registered at runtime) |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | List MCP resources |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | Read MCP resources |
| LSPTool | `LSPTool/` | LSP language server integration |

### Planning and State

| Tool | Directory | Function |
|------|-----------|---------|
| EnterPlanModeTool | `EnterPlanModeTool/` | Enter plan mode (read-only, no execution) |
| ExitPlanModeTool | `ExitPlanModeTool/` | Exit plan mode |
| EnterWorktreeTool | `EnterWorktreeTool/` | Enter git worktree isolated environment |
| ExitWorktreeTool | `ExitWorktreeTool/` | Exit worktree environment |
| TodoWriteTool | `TodoWriteTool/` | Write Todo list (displayed in sidebar) |
| BriefTool | `BriefTool/` | Generate session summary |

### Network Access

| Tool | Directory | Function |
|------|-----------|---------|
| WebFetchTool | `WebFetchTool/` | HTTP fetch, HTML→Markdown conversion, domain safety check |
| WebSearchTool | `WebSearchTool/` | Web search |

### System and Scheduling

| Tool | Directory | Function | Condition |
|------|-----------|----------|-----------|
| ConfigTool | `ConfigTool/` | Read/write config | Ant internal |
| SleepTool | `SleepTool/` | Wait (proactive mode) | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | Synthetic output (special purpose) | — |
| ScheduleCronTool | `ScheduleCronTool/` | Create/delete/list scheduled tasks | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | Remote triggers | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | Ask user a question (interactive) | — |

---

## 3.5 Tool Execution Flow

### Complete Flow from LLM tool_use to Tool Execution

The entry point for tool execution is `runToolUse()` in `services/tools/toolExecution.ts` (`toolExecution.ts:298-428`), which is an async generator:

```
LLM outputs tool_use block
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — find tool, supports aliases (backward compat for renamed tools)
    ↓
abortController.signal.aborted? → return CANCEL_MESSAGE
    ↓
streamedCheckPermissionsAndCallTool() [returns AsyncIterable]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — Zod type validation
  2. tool.validateInput(input, context)  — tool custom validation
  3. runPreToolUseHooks()                — execute PreToolUse hooks
  4. canUseTool()                        — permission check (may show UI confirmation)
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — persist large results
  7. runPostToolUseHooks()               — execute PostToolUse hooks
    ↓
yield MessageUpdateLazy (containing tool_result)
    ↓
next LLM reasoning round
```

An important backward-compatible design (`toolExecution.ts:350-360`): when a tool is renamed, the old name is kept as `aliases`. When a tool isn't found in `options.tools`, the system also checks `getAllBaseTools()` for alias matches — ensuring old tool names in historical transcripts can still execute.

### Streaming Tool Execution

Tool execution is streamed via `Stream<MessageUpdateLazy>` (`toolExecution.ts:500-535`):

```typescript
// toolExecution.ts:500-535 (simplified)
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // progress messages
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // final results
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

The significance of streaming design: the UI can display progress in real time while the tool is still executing (e.g., BashTool's live output, AgentTool's sub-agent progress). Progress messages and final results are passed through the same `Stream` pipeline, simplifying consumer code.

### Concurrent Tool Execution

Claude Code supports the LLM outputting multiple `tool_use` blocks in a single response and executing them in parallel. The prerequisite for concurrency is: **all tools declare `isConcurrencySafe: () => true`**.

During parallel execution, `contextModifier` won't execute (as the `ToolResult` comment says: "contextModifier is only honored for tools that aren't concurrency safe"). This is an important safety constraint: operations that modify global context cannot be performed in a concurrent environment.

Typical concurrency-safe tools: GrepTool, GlobTool, FileReadTool (all declare `isConcurrencySafe: () => true`).

---

## 3.6 ToolSearch — Lazy Loading Mechanism

### Why ToolSearch Is Needed (Prompt Bloat Problem)

Each tool's definition (name + JSON Schema + description) consumes tokens when sent to the LLM. When tool count exceeds a threshold (experiments show ~40-60 tools), the problems that arise are:

1. **Rising token costs**: every API call carries a large amount of tool definitions
2. **Attention dilution**: the LLM may give less attention to each tool when facing dozens of them
3. **Risk of prompt cache invalidation**: changes to the tool list (e.g., MCP tools dynamically joining) invalidate the cache

ToolSearch's solution is **on-demand loading**: most tools are marked with `shouldDefer: true`, not sending the full schema in the initial prompt; they're only loaded after being discovered through search.

### Registration and Discovery of Deferred Tools

Tools declare whether to lazy-load via the `shouldDefer` field (`Tool.ts:456-462`):

```typescript
// Tool.ts:456-462
readonly shouldDefer?: boolean

/**
 * When true, this tool is never deferred — its full schema appears in the
 * initial prompt even when ToolSearch is enabled. For MCP tools, set via
 * `_meta['anthropic/alwaysLoad']`. Use for tools the model must see on
 * turn 1 without a ToolSearch round-trip.
 */
readonly alwaysLoad?: boolean
```

The `isDeferredTool()` function (defined in `tools/ToolSearchTool/prompt.ts`) determines whether a tool should be deferred: tools with `shouldDefer: true` and without `alwaysLoad: true` are marked as deferred.

ToolSearchTool itself **is never deferred** — it must be available in the first round; otherwise other tools can't be discovered.

### On-Demand Loading Implementation

ToolSearchTool's `call()` method (`ToolSearchTool.ts:221-302`) supports two query modes:

**Direct selection mode** (`select:` prefix):
```
query: "select:NotebookEdit"     → directly returns NotebookEditTool
query: "select:Read,Edit,Grep"   → batch select multiple tools
```

**Keyword search mode**:
```
query: "jupyter notebook"        → keyword matching, returns NotebookEditTool, etc.
query: "mcp__github"             → MCP server prefix matching
```

Search scoring algorithm (`ToolSearchTool.ts:155-198`):

```
Exact match in tool name portion (MCP): +12 points
Exact match in tool name portion (normal): +10 points
Partial keyword match in tool name (MCP): +6 points
Partial keyword match in tool name (normal): +5 points
Full tool name match fallback: +3 points
searchHint word-boundary match: +4 points (carefully crafted capability summary, strong signal)
Description text word-boundary match: +2 points
```

The `searchHint` field's weight (+4 points) is higher than description text (+2 points), encouraging tool developers to provide precise capability summaries. For example, GrepTool's `searchHint: 'search file contents with regex (ripgrep)'`, FileEditTool's `searchHint: 'modify file contents in place'`.

Search results are returned to the LLM via `tool_reference` blocks (`ToolSearchTool.ts:330-352`), a special extension of the Anthropic API that tells the server "please inject the full schema for these tools into the current conversation's tool list."

---

## 3.7 Tool Result Storage

### Disk Storage Strategy

Tool execution results can be very large (e.g., reading a 10MB log file, running a command that outputs a lot). Placing large results directly in message history wastes tokens and inflates the context for subsequent requests.

`utils/toolResultStorage.ts` implements an **on-demand persistence** strategy:

1. Calculate result size (`contentSize()`)
2. Compare against the tool's `maxResultSizeChars` threshold (resolved via `getPersistenceThreshold()`)
3. Results exceeding the threshold are written to `~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`
4. Replaced with a message containing the file path + preview

```typescript
// toolResultStorage.ts:168-177
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  let message = `${PERSISTED_OUTPUT_TAG}\n`
  message += `Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}\n\n`
  message += `Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):\n`
  message += result.preview
  message += result.hasMore ? '\n...\n' : '\n'
  message += PERSISTED_OUTPUT_CLOSING_TAG
  return message
}
```

`PREVIEW_SIZE_BYTES = 2000` (~2KB); preview is truncated at the last newline to avoid cutting in the middle of a line.

A key **idempotency** design (`toolResultStorage.ts:145-158`): writes use `{ flag: 'wx' }` (exclusive create); if the file already exists, ignore the write error and use the existing file to generate the preview. This ensures microcompact replay of historical messages doesn't write twice, and doesn't error on EEXIST.

FileReadTool has a special handling: `maxResultSizeChars: Infinity` — read tool results are never persisted to disk. The reason is explained in the comment: "persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits" (persisting would create a loop: Read reads a file, the result is too large and persisted as a file, the model then uses Read to read that file...).

### Token Budget Management

`toolResultStorage.ts` also implements a more macro-level **per-message aggregate tool result budget**. This is driven by the `ContentReplacementState` mechanism (`toolResultStorage.ts:395-440`):

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // tool_use_ids that have passed budget check (results frozen)
  replacements: Map<string, string>  // replaced ID → replacement content string
}
```

Core constraint: **once a result is judged (replaced or not), it never changes** (guaranteed by the `seenIds` set). This is for prompt cache stability — the handling of the same tool_use_id must remain consistent throughout the session; otherwise the cache would be invalidated by content changes.

The budget limit is dynamically controlled by GrowthBook feature flag `tengu_hawthorn_window`. When total tool results in a message exceed the budget, the system replaces the largest tool results with disk-persisted versions until the total drops within budget.

---

## 3.8 Design Decision Analysis

### Self-Describing vs. External Registry Tradeoff

Claude Code chose the **self-describing** pattern (each tool carries its own schema, description, prompt, rendering logic) rather than centralizing this information in a registry.

Advantages:
- **Tool is fully self-contained**: adding a new tool only requires a directory, no need to modify central registry logic
- **Descriptions can be dynamically generated**: `description()` and `prompt()` are async functions, can dynamically adjust content based on environment, permissions, install state
- **Rendering logic co-located with tool**: React rendering components are directly in the tool file; changing tool behavior and changing UI is the same PR

Disadvantages:
- **Tool interface bloat**: `Tool` type has 40+ methods/fields; new tool authors need to understand many interface details
- **Duplicate code**: each tool has `renderToolUseMessage`, `renderToolResultMessage`, and other rendering methods with highly similar patterns
- **`buildTool()` can't fully eliminate this**: provides defaults, but many methods still need each tool to implement

In practice, Claude Code mitigates code duplication through **shared UI components** (e.g., `tools/shared/`) and **pattern extraction** (e.g., `lazySchema()`), but the fundamental interface complexity remains.

### Why Some Tools Are Lazy-Loaded

ToolSearch's lazy-loading decision follows one principle: **tools that the first round of conversation may not need should be deferred**.

`alwaysLoad` tools (never deferred) should satisfy: the model needs to know they exist on the first turn. Typical examples are AgentTool, BashTool, FileReadTool — these are fundamental tools for any programming task.

`shouldDefer` tools (lazy-loaded) are typically: tools needed only for specific scenarios (NotebookEditTool only needed for Jupyter tasks), large numbers of MCP tools (users install dozens of MCP servers but only use a few in each conversation).

MCP tools by default trigger the ToolSearch mechanism based on tool count, but can be forced to not defer by setting `_meta['anthropic/alwaysLoad']` in tool metadata.

### Layered Design of Tool Permissions

Tool permissions use a **three-layer defense** design:

1. **Zod type validation** (first step of `checkPermissionsAndCallTool`): the tool's inputSchema strictly validates parameter types; LLM-generated wrong-type parameters are rejected with error feedback
2. **Tool custom validation** (`validateInput()`): tools implement their own business logic validation, e.g., FileEditTool checks that old_string and new_string are different, checks if file size exceeds 1GiB
3. **General permission system** (`canUseTool()` + `checkPermissions()`): makes final judgment based on user-configured allow/deny rules, whether the tool is read-only, whether it's a destructive operation, etc.; may show interactive confirmation

These three layers execute sequentially; failure at any layer short-circuits to not enter the next.

---

## 3.9 Transferable Patterns

### General Design of the Self-Describing Tool Pattern

The most transferable pattern distilled from Claude Code's tool system is: **tool as self-contained plugin**.

Core principles:
1. **Schema is documentation is validator**: use Zod schema to define input, auto-generate JSON Schema for the LLM, while validating LLM output at runtime
2. **Factory function + safe defaults**: `buildTool()` provides fail-safe default behavior (default not concurrency-safe, default read-only is false); tool developers only need to declare their exceptions
3. **searchHint concise summary**: 3-10 word capability description, specifically optimized for keyword search, separated from full description
4. **Capability declaration over runtime judgment**: `isReadOnly()`, `isConcurrencySafe()`, `isDestructive()` let schedulers make scheduling decisions without executing the tool

### What Doramagic's Tool System (Brick System) Can Learn

Doramagic's Brick system (278+ bricks) has deep similarities to Claude Code's tool system, but also fundamental differences:

**Similarities**:
- Both are "plugin-style" architectures: each Brick/Tool is a self-contained functional unit
- Both need description mechanisms: letting LLM know when to use which tool/brick
- Both have classification systems: organized by functional domain

**Specific borrowable patterns**:

1. **`searchHint` analogous to Brick's tags**: Claude Code provides each tool with a 3-10 word concise capability description specifically for search matching. Doramagic's bricks currently use tags and categories for organization; adding a `hint` field specifically optimized for model brick discovery efficiency is worth considering.

2. **Lazy loading → on-demand brick activation**: Claude Code's deferred tools mechanism shows that cramming all bricks into the system prompt is not a good idea. Doramagic can reference the `shouldDefer` design, making infrequently-used bricks (domain-specific bricks) lazy-loaded, only activated when the model explicitly needs them.

3. **`maxResultSizeChars` → brick output budget**: each tool self-declares the maximum token budget for results, compressing if exceeded. Doramagic's brick output (extracted knowledge JSON) can also be very large; referencing this mechanism implements a "summary first, details on demand" output strategy.

4. **`isConcurrencySafe` → brick parallelism declaration**: in Doramagic's knowledge extraction pipeline, multiple bricks may simultaneously operate on the same codebase. Explicitly declaring bricks' concurrency safety lets the scheduler automatically decide which bricks can run in parallel and which need to be serial.

5. **Three-layer permission defense → brick runtime safety**: for Doramagic running as an OpenClaw Skill, the legitimacy validation of Brick execution can reference this three-layer design: schema validation → business validation → platform permission check.

**Fundamental difference**: Claude Code's tools primarily target **deterministic operations** (reading files, executing commands), with precisely definable outputs. Doramagic's bricks target **knowledge extraction**, with semantic outputs. This means Doramagic cannot strictly validate brick outputs with Zod schema like Claude Code does — this is exactly the meaning of Doramagic's "code tells facts, AI tells stories" architectural principle: deterministic skeleton (facts extraction) can be constrained with schema; non-deterministic interpretation (stories generation) does not need to.

---

## 3.10 Source Index

| File | LOC | Key Content |
|------|-----|------------|
| `src/Tool.ts` | 792 | `Tool` type definition, `buildTool()`, `ToolUseContext`, `ToolResult`, `TOOL_DEFAULTS` |
| `src/tools.ts` | 389 | `getAllBaseTools()`, `getTools()`, `assembleToolPool()`, `getMergedTools()`, `filterToolsByDenyRules()` |
| `src/services/tools/toolExecution.ts` | 1,745 | `runToolUse()`, `checkPermissionsAndCallTool()`, `streamedCheckPermissionsAndCallTool()`, `buildSchemaNotSentHint()` |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 471 | `searchToolsWithKeywords()`, `parseToolName()`, keyword scoring algorithm, `select:` prefix direct selection |
| `src/utils/toolResultStorage.ts` | 1,040 | `persistToolResult()`, `buildLargeToolResultMessage()`, `ContentReplacementState`, `enforceToolResultBudget()` |
| `src/tools/BashTool/BashTool.tsx` | ~1,800+ | `isSearchOrReadBashCommand()`, sandbox, background tasks, progress display |
| `src/tools/FileEditTool/FileEditTool.ts` | ~500+ | string replacement, large file protection (1GiB limit), secret detection |
| `src/tools/FileReadTool/FileReadTool.ts` | ~600+ | multi-format support (PDF/images/Notebooks), token counting, `maxResultSizeChars: Infinity` |
| `src/tools/GrepTool/GrepTool.ts` | ~400+ | ripgrep integration, head_limit/offset pagination, `DEFAULT_HEAD_LIMIT = 250` |
| `src/tools/WebFetchTool/utils.ts` | ~450+ | domain blacklist check, LRU cache (50MB/15min), HTML→Markdown conversion |
| `src/tools/MCPTool/classifyForCollapse.ts` | ~350 | MCP tool search/read classification (20+ pre-configured rules for Slack/GitHub/Linear/Jira, etc.) |

**Key constants** (scattered across multiple files):
- `PREVIEW_SIZE_BYTES = 2000` (toolResultStorage.ts) — large result preview size
- `DEFAULT_HEAD_LIMIT = 250` (GrepTool.ts) — grep default result limit
- `MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024` (WebFetchTool/utils.ts) — network fetch 10MB limit
- `FETCH_TIMEOUT_MS = 60_000` (WebFetchTool/utils.ts) — HTTP request 60 second timeout
- `CACHE_TTL_MS = 15 * 60 * 1000` (WebFetchTool/utils.ts) — URL cache 15 minutes
- `PROGRESS_THRESHOLD_MS = 2000` (BashTool.tsx) — display progress after 2 seconds
- `MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024` (FileEditTool.ts) — file editing 1GiB limit
