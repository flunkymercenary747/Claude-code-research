# Chapter 4: Agent Orchestration and Multi-Agent Architecture

## 4.1 Overview and Positioning

Claude Code's multi-agent system is the most complex subsystem in the entire product architecture, spanning approximately 8,700 lines of core code across 12 key modules. This system solves a fundamental engineering problem: **how to safely and efficiently orchestrate concurrent execution of multiple LLM Agents in a single-threaded REPL application**.

The system provides three progressive collaboration modes:

| Mode | Trigger | Concurrency | Communication | Isolation Level |
|------|---------|-------------|---------------|-----------------|
| **Subagent (default)** | AgentTool call | sync/async | function return values | In-process AsyncGenerator |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | fully async | `<task-notification>` XML | Independent AbortController |
| **Team Mode** | `spawnTeammate()` + TeamFile | persistent parallel | file mailbox + polling | Tmux Pane / InProcess / Remote |

These three modes are not independent implementations, but share the same `runAgent()` core engine (`runAgent.ts`), achieving different behavioral characteristics through parameter combinations — one of the most elegant design decisions in the entire system.

**Source code scale statistics:**

| File | LOC | Responsibility |
|------|-----|---------------|
| `AgentTool.tsx` | 1,397 | Unified entry, routing decisions, lifecycle management |
| `runAgent.ts` | 973 | Agent execution engine, query() loop |
| `loadAgentsDir.ts` | 755 | Agent definition parsing (Markdown/JSON/Plugin) |
| `agentToolUtils.ts` | 686 | Tool filtering, permissions, result serialization |
| `UI.tsx` | 871 | Agent progress and result rendering |
| `coordinatorMode.ts` | 369 | Coordinator system prompt and context |
| `SendMessageTool.ts` | 917 | 5-route message routing |
| `spawnMultiAgent.ts` | 1,093 | Teammate spawning (Tmux/InProcess) |
| `inProcessRunner.ts` | 1,552 | InProcess backend complete implementation |
| `teammateMailbox.ts` | 1,183 | File mailbox protocol |
| `worktree.ts` | 1,519 | Git Worktree isolation |

## 4.2 Theoretical Foundations

### 4.2.1 Actor Model and Agent Orchestration

Claude Code's multi-agent architecture is a pragmatic variant of the Actor model in the LLM orchestration domain. The three core primitives of the classic Actor model (Hewitt, 1973) — **receive messages, create new Actors, send messages** — have clear correspondences in the code:

| Actor Primitive | Claude Code Implementation | Source Location |
|----------------|---------------------------|-----------------|
| Receive messages | `waitForNextPromptOrShutdown()` polling loop | `inProcessRunner.ts:689-868` |
| Create Actor | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| Send messages | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

But there are two key deviations from the pure Actor model:

1. **Asymmetric hierarchy**: Leader has global view (AppState), Workers only have their own ToolUseContext. This is not peer-to-peer Actors, but a clear Leader-Worker hierarchy supervision tree.
2. **Shared state channel**: InProcess backend Teammates share the root AppState store via `setAppStateForTasks` (`runAgent.ts:336-337`), rather than pure message passing. This is a pragmatic compromise on the Actor model — in-process, shared state is more efficient than serialized messages.

### 4.2.2 Message Passing vs. Shared Memory Concurrency Model

The system simultaneously uses two concurrency models, chosen based on isolation level:

**Message passing model** (Team Mode - Tmux Pane backend):
```
Leader → writeToMailbox("worker-1", {...}) → file system → readMailbox() → Worker
```
Communication is implemented via JSON files + file locks. `teammateMailbox.ts`'s `LOCK_OPTIONS` configures exponential backoff retries (10 retries, 5-100ms) to serialize concurrent writes:

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**Shared memory model** (InProcess backend):
```
Leader → setAppState(prev => {...}) → same AppState store ← getAppState() ← Worker
```
InProcess Teammates directly read/write the root store via `toolUseContext.setAppStateForTasks`. Race conditions are avoided through React-style `setAppState(prev => {...})` functional update semantics (though the underlying implementation is not React, it uses the same CAS pattern).

### 4.2.3 Coordinator Pattern in Distributed Systems

Coordinator Mode's design maps the classic Coordinator pattern (also called Master-Worker) from distributed systems, but adds a unique constraint: **the Coordinator itself is an LLM Agent, and its "coordination logic" is not hard-coded but programmed through system prompts**.

The `getCoordinatorSystemPrompt()` function defined in `coordinatorMode.ts:126-369` returns approximately 5,000 characters of structured prompt, containing the complete Worker scheduling strategy:

```typescript
// coordinatorMode.ts:161-167 — key scheduling rules
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

This pattern of "programming coordination logic through prompts" means the Coordinator's behavior can be adjusted by modifying prompts — the four-phase research→synthesis→implementation→verification workflow is not enforced by code but achieved through the LLM's instruction-following capability. This stands in sharp contrast to traditional distributed Coordinators with hard-coded scheduling logic.

## 4.3 Architecture and Data Structures

### 4.3.1 Overall Architecture (Leader-Worker)

```
                    ┌─────────────────────────────────────────┐
                    │           Human User (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ user input
                    ┌──────────────▼──────────────────────────┐
                    │         Main REPL (query() loop)         │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — routing   │     │
                    │    └──┬─────────┬─────────┬───────┘     │
                    │       │         │         │              │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐     │
                    │  │ Sync   │ │ Async  │ │Teammate │     │
                    │  │Agent   │ │Agent   │ │Spawn    │     │
                    │  │(block) │ │(fire&  │ │         │     │
                    │  │        │ │forget) │ │         │     │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘     │
                    │      │          │         │              │
                    │      └────┬─────┘    ┌────▼──────────┐  │
                    │           │          │  spawnMulti-   │  │
                    │      ┌────▼────┐     │  Agent.ts      │  │
                    │      │runAgent │     └────┬───────────┘  │
                    │      │  .ts    │          │              │
                    │      │         │     ┌────▼──────────┐  │
                    │      │ query() │     │  3 Backends:   │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    Communication layer:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → parent collects      │
    │  Async Agent:   <task-notification> XML → user msg   │
    │  Teammate:      file mailbox (.claude/teams/*/inboxes/)│
    │  InProcess:     AppState shared + mailbox fallback   │
    │  Remote (ant):  teleportToRemote() → CCR session     │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 AgentDefinition Type System

Agent definitions use a three-layer union type design:

```typescript
// loadAgentsDir.ts — core type hierarchy

// Base type: fields shared by all Agents
type BaseAgentDefinition = {
  agentType: string              // routing key (e.g., "Explore", "worker")
  whenToUse: string              // basis for LLM Agent selection
  tools?: string[]               // whitelist (undefined = all)
  disallowedTools?: string[]     // blacklist
  model?: string                 // 'inherit' | specific model name
  effort?: EffortValue           // reasoning effort level
  permissionMode?: PermissionMode // permission inheritance strategy
  maxTurns?: number              // max conversation turns
  background?: boolean           // always run in background
  isolation?: 'worktree' | 'remote' // isolation mode
  memory?: AgentMemoryScope      // persistent memory
  omitClaudeMd?: boolean         // omit CLAUDE.md (saves ~5-15 Gtok/week)
  // ...
}

// Built-in Agent: dynamic prompt, no static systemPrompt
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent: loaded from Markdown/JSON
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent: from plugin system
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// Final union type
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

The elegance of this design lies in the `getSystemPrompt` method: Built-in Agents accept a `toolUseContext` parameter (can dynamically adjust prompts based on the current tool set), while Custom/Plugin Agents use closures to capture Markdown content at parse time. This means:

- **Built-in Agent prompts are dynamic**: may differ on each call
- **Custom Agent prompts are static**: defined by Markdown files, but if `memory` is enabled, memory content is appended at runtime (`loadAgentsDir.ts:335-340`)

Agent definition loading priority follows the override chain: `builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`, implemented via Map with later entries overriding earlier ones in `getActiveAgentsFromList()` (`loadAgentsDir.ts:169-186`).

### 4.3.3 Unified Abstraction Across Three Execution Backends

The three backends share a `runAgent()` AsyncGenerator interface, but differ fundamentally in process model and communication mechanism:

| Dimension | Tmux Pane | InProcess | Remote (ant-only) |
|-----------|-----------|-----------|-------------------|
| **Process model** | Independent Claude CLI process | Same-process AsyncLocalStorage isolation | CCR remote session |
| **Launch method** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **Communication** | File mailbox polling (500ms) | Shared AppState + mailbox fallback | HTTP API |
| **Permissions** | Independent permission context | Leader UI queue bridge | Remote independent |
| **Resource overhead** | High (full process) | Low (shared V8 heap) | Very high (remote instance) |
| **Lifetime** | Independent from Leader | Bound to Leader process | Independent |
| **Detection logic** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

Backend detection and degradation at `spawnMultiAgent.ts:339-375` implements an elegant degradation chain:

```
iTerm2 (it2 backend) → Tmux → InProcess fallback
```

If iTerm2 is detected but the it2 CLI isn't installed, the system shows an interactive setup prompt (`It2SetupPrompt`), letting users choose to install it2 or fall back to Tmux.

### 4.3.4 Communication Protocol Data Structures

**File mailbox message format** (`teammateMailbox.ts:42-49`):

```typescript
type TeammateMessage = {
  from: string       // sender name
  text: string       // message content (can be plain text or JSON structured message)
  timestamp: string  // ISO timestamp
  read: boolean      // read flag
  color?: string     // sender color identifier
  summary?: string   // UI preview summary (5-10 words)
}
```

Mailbox path follows a fixed format: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**Structured message types** (transmitted via JSON encoding in the `text` field):

| Message Type | Direction | Purpose |
|-------------|-----------|---------|
| `shutdown_request` | Leader → Worker | Request shutdown |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | Shutdown response |
| `idle_notification` | Worker → Leader | Idle notification |
| `permission_request` | Worker → Leader | Permission request |
| `permission_response` | Leader → Worker | Permission response |
| `plan_approval_request` | Worker → Leader | Plan Mode approval request |
| `plan_approval_response` | Leader → Worker | Approval response |
| `sandbox_permission_request` / `_response` | Both directions | Network sandbox permissions |
| `task_assignment` | Leader → Worker | Task assignment |
| `team_permission_update` | Leader → Workers | Permission broadcast |

## 4.4 Core Algorithms and Flow

### 4.4.1 AgentTool Routing Decision Tree (Complete)

`AgentTool.call()` is the system's unified entry point; its routing logic is implemented in `AgentTool.tsx:238-764`. Complete decision tree:

```
AgentTool.call(input) entry
│
├─ [1] Both team name + name params present?
│   ├─ YES: Is this a Teammate attempting nested spawning?
│   │   ├─ YES: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NO: → spawnTeammate() (return teammate_spawned)
│   └─ NO: continue
│
├─ [2] Resolve effectiveType (subagent_type)
│   ├─ Explicitly specified → use specified value
│   ├─ Not specified + Fork Gate ON → undefined (Fork path)
│   └─ Not specified + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (Fork path)
│   ├─ YES: recursive Fork check
│   │   ├─ Already in Fork subprocess → throw
│   │   └─ Passes → selectedAgent = FORK_AGENT
│   └─ NO: search in activeAgents
│       ├─ Found → selectedAgent = found
│       ├─ Permission denied → throw (with deny rule info)
│       └─ Does not exist → throw (list available agents)
│
├─ [4] Resolve effectiveIsolation
│   ├─ 'remote' (ant-only) → teleportToRemote() → return remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → subsequent steps use worktreePath
│
├─ [5] Build system prompt and prompt messages
│   ├─ Fork path: inherit parent prompt + buildForkedMessages()
│   └─ Normal: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] shouldRunAsync determination
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — BUT NOT isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → return { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → enter while(true) loop
│       ├─ Race: nextMessage vs backgroundSignal
│       │   ├─ background wins → switch to async execution (wasBackgrounded=true)
│       │   └─ message wins → yield message, track progress
│       └─ loop ends → finalizeAgentTool() → return AgentToolResult
```

### 4.4.2 runAgent() AsyncGenerator Execution Flow

`runAgent()` is the core engine of the entire multi-agent system (`runAgent.ts:247-860`); it is an `AsyncGenerator<Message, void>` — every yielded Message allows the caller to process it (record, display, or push to background queue).

**Key phases of the execution flow:**

1. **Tool resolution**: `resolveAgentTools()` resolves the Agent definition's `tools` whitelist into actual Tool objects, while applying the `disallowedTools` blacklist (`runAgent.ts:500-502`)

2. **System Prompt construction**: built from `override?.systemPrompt` or `getAgentSystemPrompt()`; Explore/Plan Agents skip `claudeMd` and `gitStatus`, saving fleet-wide ~5-15 Gtok/week (`runAgent.ts:389-409`)

3. **AbortController strategy** (`runAgent.ts:524-528`):
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // external control (async path)
     : isAsync
       ? new AbortController()      // async: independent controller
       : toolUseContext.abortController  // sync: share parent controller
   ```

4. **Permission override** (`runAgent.ts:414-497`): Agent's `permissionMode` overrides the parent's mode, but the three parent modes `bypassPermissions`, `acceptEdits`, `auto` always take priority — ensuring admin-configured security policies can't be downgraded by sub-Agents.

5. **Core loop** — directly calling `query()` and yielding (`runAgent.ts:748-806`):
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... handle stream_event, attachment, recordable messages
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **Cleanup finally block** (`runAgent.ts:816-858`): MCP cleanup, session hooks cleanup, prompt cache tracking, file state cache release, Perfetto deregistration, AppState todos cleanup, background bash task kill — 9 cleanup operations in total, ensuring no resource leaks.

### 4.4.3 Async Agent Lifecycle (fire-and-forget)

The complete lifecycle of async Agents is driven by `runAsyncAgentLifecycle()` (`agentToolUtils.ts:322-497`):

```
registerAsyncAgent() → register task to AppState
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — collect all messages
   │   ├─ agentMessages.push(message)
   │   ├─ if task.retain → append to AppState.tasks[taskId].messages
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — SDK progress event
   │
   ├─ finalizeAgentTool() — extract final result
   │
   ├─ completeAsyncAgent() — mark complete (FIRST, before any slow operation)
   │   │                      ↑ key design: gh-20236 fix
   │   │                        classifyHandoff and worktree cleanup may hang
   │   │                        cannot block state transition
   │
   ├─ classifyHandoffIfNeeded() — safety classifier check (optional)
   │
   ├─ getWorktreeResult() — worktree cleanup
   │
   └─ enqueueAgentNotification() — notify parent with <task-notification> XML
```

**The gh-20236 fix** is a noteworthy design decision: `completeAsyncAgent()` is called before `classifyHandoffIfNeeded()` and `getWorktreeResult()`. The comment explicitly explains the reason:

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 Tool Filtering and Permission Inheritance

Tool filtering is a three-layer filter chain (`agentToolUtils.ts:66-115`):

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS — tools forbidden to all Agents
Layer 2: CUSTOM_AGENT_DISALLOWED_TOOLS — additionally forbidden only for Custom Agents
Layer 3: ASYNC_AGENT_ALLOWED_TOOLS — whitelist for async Agents (inverted logic)
```

Special exceptions:
- MCP tools (`mcp__` prefix) are always allowed
- `ExitPlanMode` is always allowed in Plan Mode
- InProcess Teammates in Agent Swarms mode can use `AgentTool` (spawning sync sub-Agents) and Task tools (coordinating via shared task list)

Tool resolution also supports wildcards (`'*'` or `undefined` = all tools) and Agent-scoped restrictions (`AgentTool(worker, researcher)` syntax, `agentToolUtils.ts:165-172`).

### 4.4.5 Coordinator Mode Four-Phase Workflow

The core logic of Coordinator Mode is defined through prompts in `getCoordinatorSystemPrompt()` at `coordinatorMode.ts:126-369`. It decomposes all tasks into four phases:

**Phase 1: Research** (Workers execute in parallel)
- Multiple Workers simultaneously explore the codebase
- Key prompt instruction: *"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Phase 2: Synthesis** (Coordinator does this)
- This is the most critical phase — Coordinator must personally read research results and understand them
- Explicitly forbidden anti-pattern: *"Never write 'based on your findings'"*
- Required output: synthesized spec containing specific file paths, line numbers, and modification content

**Phase 3: Implementation** (Workers execute)
- Coordinator decides whether to continue (`SendMessageTool`) or spawn fresh (`AgentTool`)
- Decision basis is context overlap (complete decision table in prompt)

**Phase 4: Verification** (Independent Worker)
- Explicitly requires independent verification: *"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- Verification standard: *"proving the code works, not confirming it exists"*

### 4.4.6 Team Mode Persistent Collaboration

Team Mode uses TeamFile (`.claude/teams/{team_name}/team.json`) to implement persistent team state. Unlike Coordinator Mode's fire-and-forget Workers, Teammates are **long-lived processes**:

1. **Spawning**: `spawnTeammate()` creates a Tmux pane or InProcess task
2. **Running**: Teammate executes prompt → completes → sends `idle_notification` → waits for next prompt
3. **Communication**: all messages through file mailbox (any backend can use filesystem communication)
4. **Shutdown**: Leader sends `shutdown_request` → Teammate's LLM decides to approve or reject

InProcess Runner's main loop (`inProcessRunner.ts:883-1464`) implements complete persistence semantics:

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. run current prompt (calling runAgent())
  // 2. mark as idle
  // 3. send idle_notification to Leader
  // 4. waitForNextPromptOrShutdown() — poll mailbox
  //    ├─ shutdown_request → pass to LLM to decide
  //    ├─ new_message → set as next round's prompt
  //    └─ aborted → shouldExit = true
}
```

Worth noting the message priority strategy (`inProcessRunner.ts:760-804`):
1. Highest priority: `shutdown_request` (Leader's shutdown command won't get buried)
2. Second: messages from `team-lead` (Leader represents user intent)
3. Last: FIFO queue of peer messages

### 4.4.7 File Mailbox Communication Protocol

The file mailbox is the communication foundation for all backends. Its design chooses **simplicity** over performance:

**Write protocol** (`teammateMailbox.ts:133-191`):
```
1. ensureInboxDir() — ensure directory exists
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — atomic create (if not exists)
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — acquire file lock
4. readMailbox() — re-read inside lock (avoid dirty reads)
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — write back
7. release() — release lock
```

**Read protocol** (`teammateMailbox.ts:83-107`):
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. return TeammateMessage[]
```

Note that reads are **lock-free** — this is intentional. The read side only needs eventual consistency, while the write side guarantees atomicity through `lockfile`.

### 4.4.8 SendMessage 5-Route Routing

`SendMessageTool.call()` implements 5 independent message routing paths (`SendMessageTool.ts`):

```
Value of input.to
│
├─ [Route 1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — cross-machine Remote Control
│   (requires safety check: cross-machine messages need explicit user consent)
│
├─ [Route 2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — local Unix Domain Socket
│
├─ [Route 3] agentNameRegistry or toAgentId match
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task stopped/evicted → resumeAgentBackground()
│       (automatically restore stopped Agent from disk transcript)
│
├─ [Route 4] to === '*'
│   → handleBroadcast() — iterate TeamFile.members, write to each mailbox
│
└─ [Route 5] other
    ├─ plain text → handleMessage() — write to mailbox
    └─ structured message → dispatch to corresponding handler:
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

The **auto-recovery** mechanism in Route 3 is particularly elegant: when sending a message to a stopped Agent, the system automatically recovers it from disk transcript and runs it in the background. This means the Coordinator can seamlessly continue a previously completed Worker via `SendMessage`, without caring whether it's still running.

### 4.4.9 Permission Delegation Complete Flow

InProcess Teammate permission handling is one of the most complex parts of the entire system (`inProcessRunner.ts:127-449`). The core challenge is: **how can a background Agent request human authorization?**

The solution is two-level fallback:

**Primary path: Leader UI queue bridge**
```
Worker triggers a tool requiring permissions
  → createInProcessCanUseTool() is called
  → hasPermissionsToUseTool() returns { behavior: 'ask' }
  → check Bash classifier auto-approval (if available)
  → getLeaderToolUseConfirmQueue() — get Leader's UI confirm queue
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ Worker identity badge
     └→ Leader terminal shows permission dialog with Worker badge
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Fallback path: mailbox permission request**
```
Worker triggers a tool requiring permissions
  → Leader UI queue unavailable
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → poll own mailbox (500ms interval)
  → wait for Leader to write back permission_response
  → processMailboxPermissionResponse()
```

Permission update propagation is also important: when Leader approves a permission and selects "Always allow," `persistPermissionUpdates()` writes to disk, and simultaneously `getLeaderSetToolPermissionContext()` writes the update back to Leader's shared context — but with `preserveMode: true`, preventing the Worker's `acceptEdits` mode from leaking back to the Coordinator (`inProcessRunner.ts:275-277`).

### 4.4.10 Worker Complete Lifecycle

```
Birth
  │
  ├─ Sync Agent path:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → return AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ Async Agent path:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ InProcess Teammate path:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — persistent loop:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → new message/shutdown/abort → decide whether to continue
          }

During execution
  │
  ├─ query() loop → API call → tool_use → canUseTool check
  │   ├─ allow → execute tool
  │   ├─ deny → tool rejected
  │   └─ ask → permission dialog (sync) or mailbox permission (async/teammate)
  │
  ├─ Progress tracking:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (SDK event)
  │
  └─ Auto-background (Sync Agent only):
      backgroundPromise race → if user presses Ctrl+Z
      → wasBackgrounded = true → continue running in background

Communication
  │
  ├─ Sync Agent: yield message → parent collects directly
  ├─ Async Agent: <task-notification> injected into parent user messages
  └─ Teammate: writeToMailbox() → Leader polls reads

Termination
  │
  ├─ Normal completion: finalizeAgentTool() → extract final text → mark completed
  ├─ User Kill: AbortError → killAsyncAgent() → extract partialResult → notify
  ├─ Error: catch → failAsyncAgent() → notify error
  └─ Cleanup: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Worktree Isolation Creation and Cleanup

Git Worktree provides file system-level isolation for Agents (`worktree.ts`). Core flow:

**Creation** (`worktree.ts:234-374`):
```
1. validateWorktreeSlug(slug) — prevent path traversal attacks
2. Quick recovery check: readWorktreeHeadSha() — if worktree exists, skip fetch
3. If not exists:
   a. Try reading local origin/<default> ref (avoid 6-8s git fetch overhead)
   b. If not local → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. Optional: sparse-checkout (only checkout specified paths)
4. performPostCreationSetup():
   - Copy settings.local.json
   - Configure git hooks (handle husky's core.hooksPath issue)
   - Symlink node_modules and other large directories
   - Copy gitignored files specified by .worktreeinclude
```

**Cleanup decision** (`AgentTool.tsx:644-685`):
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // Hook-based always preserve
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // no changes, delete worktree
    }
  }
  return { worktreePath, worktreeBranch }; // has changes, preserve
};
```

Key security measures:
- `validateWorktreeSlug()` validates each `/`-separated segment matches `[a-zA-Z0-9._-]+`, preventing `../../../` path traversal
- `flattenSlug()` flattens nested slugs (`user/feature` → `user+feature`), avoiding git ref D/F conflicts and directory nesting issues
- `GIT_NO_PROMPT_ENV` disables all git credential prompts, preventing CLI hangs

## 4.5 Design Decision Analysis

### 4.5.1 Why File Mailbox Instead of IPC

File mailbox looks like a "primitive" choice — why not use Unix Domain Sockets, Named Pipes, or gRPC?

**Core reason: backend agnosticism**. The file system is the greatest common denominator of all three backends (Tmux, InProcess, Remote):
- Tmux Pane is an independent process with no shared memory
- InProcess is in the same process but uses AsyncLocalStorage isolation
- Remote is cross-network, but can share a network filesystem

Additional advantages of file mailbox:
1. **Observability**: directly `cat ~/.claude/teams/*/inboxes/*.json` for debugging
2. **Persistence**: messages aren't lost if the process crashes
3. **Simplicity**: no complex connection management, heartbeats, reconnection
4. **Concurrency safety**: file locks provided by `proper-lockfile` are sufficiently reliable

The trade-off is **latency**: a 500ms polling interval means worst-case message delivery has 500ms latency. But in LLM Agent scenarios, each tool call already takes seconds, making 500ms negligible.

### 4.5.2 InProcess vs. Pane Backend Tradeoffs

| Dimension | InProcess | Tmux Pane |
|-----------|-----------|-----------|
| **Memory** | Shared V8 heap (low) | Independent process heap (high) |
| **Startup latency** | ~0ms | ~2-3s (CLI startup) |
| **Isolation** | AsyncLocalStorage (weak) | OS process (strong) |
| **Permissions** | Leader UI bridge (real-time) | Mailbox polling (delayed) |
| **Debugging** | Shared logs (complex) | Independent terminal (intuitive) |
| **Lifetime** | Bound to Leader | Independent |

InProcess backend's biggest advantage is **permission bridging** — via `getLeaderToolUseConfirmQueue()`, Worker's permission dialogs appear directly in Leader's terminal with Worker badge identification. Users don't need to switch to the Worker's terminal to approve permissions.

But InProcess has a fundamental limitation: **Workers cannot spawn background Agents** (`AgentTool.tsx:277-278`), because their lifecycle is bound to the Leader process, and background Agents require an independent AbortController.

### 4.5.3 Design Philosophy: Humans Always Control Permissions

The permission design of the entire multi-agent system follows one non-negotiable principle: **humans are always the ultimate permission grantor**.

This principle's embodiment in code:
1. **Sub-Agents cannot escalate permissions**: `runAgent.ts:419` — `bypassPermissions`, `acceptEdits`, `auto` mode parent settings always take priority over sub-Agent's `permissionMode`
2. **Leader's permissions don't leak to Workers**: `runAgent.ts:467-477` — when `allowedTools` is specified, clear session-level allow rules, keeping only CLI argument-level rules
3. **Cross-machine messages require explicit consent**: `SendMessageTool.ts:checkPermissions` — sending to `bridge:` addresses requires `safetyCheck`, and `classifierApprovable: false` (safety classifier cannot auto-approve)
4. **Plan Mode approval**: Teammates can be set to `plan_mode_required`; they must submit a plan for Leader approval before execution

### 4.5.4 Recursive Design of query() Loop Reuse

The core of `runAgent()` is calling the `query()` function — the same function the main REPL loop uses. This means **sub-Agents and the main Agent use exactly the same API call and tool execution pipeline**.

```typescript
// runAgent.ts:748-757 — Agent's query() call
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```

The far-reaching implications of this design:
- **Tool consistency**: tools used by Agents are exactly the same as those used by users (only filtered)
- **Recursive capability**: an Agent's tool pool can include `AgentTool`, so Agents can spawn sub-Agents (InProcess Teammates are allowed to spawn sync sub-Agents)
- **Prompt Cache reuse**: the Fork path ensures sub-Agent's API request prefix is byte-identical to the parent Agent's via `useExactTools`, maximizing prompt cache hit rate

But recursion also brings risks — infinite recursive forks. The solution is double-checking (`AgentTool.tsx:331-333`):
1. `querySource === 'agent:builtin:fork'` — compile-time durable (context.options unaffected by autocompact)
2. `isInForkChild(messages)` — message scan fallback

### 4.5.5 Comparison with LangGraph / AutoGen / CrewAI

| Dimension | Claude Code | LangGraph | AutoGen | CrewAI |
|-----------|------------|-----------|---------|--------|
| **Orchestration model** | Leader-Worker (prompt-programmed) | DAG/StateGraph | Agent Chat | Sequential/Hierarchical |
| **Communication** | File mailbox + shared AppState | State channels | Python function calls | Shared memory |
| **Isolation** | 3 levels (InProcess/Pane/Remote) | None | None | None |
| **Permissions** | Human-in-the-loop, always | Optional | Optional | None |
| **Persistence** | Disk transcript + TeamFile | Optional checkpointing | None | None |
| **Tool sharing** | Unified Tool pool + filtering | Per-node independent binding | Per-Agent independent | Per-Agent independent |
| **Model heterogeneity** | `model` param per Agent | Supported | Supported | Supported |

Claude Code's two biggest differentiators:

1. **Coordinator logic is prompt-programmed** — other frameworks' orchestration logic is hard-coded DAGs or state machines. Claude Code's Coordinator is programmed through natural language prompts, meaning orchestration strategies can be adjusted by modifying prompts, no code changes needed.
2. **File system as communication foundation** — this seems primitive, but provides unified cross-process, cross-machine communication capability and complete observability. Other frameworks rely on in-process Python function calls, requiring additional RPC layers in multi-machine scenarios.

## 4.6 Transferable Patterns

### 4.6.1 General Patterns for Agent Orchestration

Five general Agent orchestration patterns can be extracted from Claude Code's implementation:

**Pattern 1: AsyncGenerator as Agent Interface**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGenerator provides pull-based message stream semantics — callers decide when to consume the next message, naturally supporting background switching (insert race at yield points) and progress tracking.

**Pattern 2: Foreground → Background Seamless Switching**

Claude Code's Sync Agents can be backgrounded mid-execution via `Promise.race([nextMessage, backgroundSignal])`. This pattern applies to any scenario requiring "long tasks that can be backgrounded mid-way." The key is having a stable taskId that transfers between foreground and background.

**Pattern 3: File System as "Least Common Denominator" for Agent Communication**

When multiple backends (in-process/cross-process/cross-machine) need unified communication, the file system is the simplest choice. JSON files + file locks provide sufficient consistency guarantees.

**Pattern 4: Prompt-Programmed Coordination**

Writing orchestration logic in system prompts rather than code makes coordination strategies "configuration" rather than "implementation." This is especially valuable in the rapid iteration phase of Agent orchestration — the cost of changing a prompt is far lower than changing code.

**Pattern 5: Safe State Transition Before Notification Decoration**

The gh-20236 fix pattern: in async flows, complete core state transitions first (`completeAsyncAgent`), then execute potentially hanging decorative operations (classifier check, worktree cleanup). Any potentially blocking operation should not gate critical state changes.

### 4.6.2 What Doramagic FlowController Can Learn

Claude Code's Agent architecture has several points worth comparing to Doramagic's FlowController (lease system + staging/delivery isolation + 12-state machine):

1. **State machine vs. Prompt-Programmed**: Doramagic uses a 12-state machine to hard-code flow control; Claude Code uses prompt programming for Coordinator. Both have applicable scenarios — use state machines for deterministic flows, prompt programming for flows requiring flexible judgment.

2. **Direct applicability of file mailbox**: Doramagic's staging/delivery directory isolation is structurally similar to Claude Code's `.claude/teams/*/inboxes/`. Doramagic's FlowController can directly adopt the file mailbox pattern to implement loose-coupling communication between skills.

3. **Permission model borrowing**: Claude Code's "sub-Agent cannot escalate permissions" principle can map to Doramagic's skill permissions — a called skill should not obtain higher system access than the caller.

4. **Worktree isolation idea**: for Doramagic's parallel skill execution (e.g., multiple soul extractors extracting from different projects in parallel), the Worktree file system isolation pattern can be borrowed, creating independent working directories for each parallel execution.

## 4.7 Source Index

| File | Path | Key Exports |
|------|------|------------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool` (buildTool definition), `inputSchema`, `outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | `runAgent()` AsyncGenerator, `filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition` union type, `getAgentDefinitionsWithOverrides()`, `parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`, `resolveAgentTools()`, `finalizeAgentTool()`, `runAsyncAgentLifecycle()`, `classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`, `getCoordinatorSystemPrompt()`, `getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool` (5-route routing), `handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`, `handleSpawnSplitPane()`, `resolveTeammateModel()`, `buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`, `createInProcessCanUseTool()`, `waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`, `writeToMailbox()`, `markMessageAsReadByIndex()`, all structured message types |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`, `createAgentWorktree()`, `removeAgentWorktree()`, `validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | `TaskState` union (7 task types), `isBackgroundTask()` |

**TaskState union type** (`tasks/types.ts`):
```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

---

*This chapter was completed based on analysis of the Claude Code TypeScript source snapshot (2026-03-31, ~512K LOC). All code references include specific file names and line number ranges.*
