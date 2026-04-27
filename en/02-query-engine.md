# Chapter 2: Query Engine — LLM Interaction Core

## 2.1 Overview and Positioning

**One-line summary:** The Query Engine is Claude Code's "heartbeat" — it owns the complete lifecycle of LLM conversation, transforming user input into multi-round agentic interactions with permission control, tool execution, context compaction, and cost tracking.

**The core problem it solves:** How to reliably orchestrate the loop of "LLM streaming response → tool call → result injection → next request" inside an async generator pipeline, while handling at least 12 exceptional branches including context window overflow, token budget, API failures, user interruption, and permission approval.

**File and code volume statistics:**

| File | LOC | Responsibility |
|------|-----|---------------|
| `QueryEngine.ts` | 1,295 | Session-level state management, SDK/headless entry |
| `query.ts` | 1,729 | Query main loop, tool execution orchestration, multi-layer recovery |
| `query/config.ts` | 46 | Immutable config snapshot |
| `query/tokenBudget.ts` | 93 | Token budget auto-continue decision |
| `query/stopHooks.ts` | 473 | Stop-time hooks (safety gate, post-processing) |
| `query/deps.ts` | 40 | Dependency injection interface (test-friendly) |
| `services/api/claude.ts` | 3,419 | API calls, streaming parsing, retry, cache strategy |
| `cost-tracker.ts` | 323 | Cost tracking and usage accumulation |
| **Total** | **~7,418** | — |

These 7,400+ lines constitute about 1.4% of Claude Code's codebase, but they are the most critical path of the entire product — every interaction between the user and Claude must pass through here.

---

## 2.2 Theoretical Foundations

### 2.2.1 Async Generator Pipeline (Coroutine Pipeline)

The core architecture of the Query Engine is based on **ES2018 Async Generator** as a streaming processing primitive. The `query()` function signature reveals this design:

```typescript
// query.ts:162
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

This is not an accidental syntax choice, but a precise application of **Coroutine theory**. Async generators simultaneously have:
- **Lazy evaluation**: consumer-driven, no pre-computation of all API responses
- **Two-way communication**: yield stream events, return terminal reason
- **Resource safety**: `finally` block guarantees `releaseStreamResources()` runs at all exit paths (including `.return()` and exceptions)

Traditional callbacks or Promise chains cannot simultaneously satisfy both "stream output to UI" and "wait for tool execution results." Async generators natively support the semantics of "producer pauses, waiting for consumer to consume."

### 2.2.2 State Machine (Implicit State Machine)

The main loop of `query.ts` is not a recursive call (although early code comments still refer to `query_recursive_call`), but a **while(true) + continue-driven explicit state machine**:

```typescript
// query.ts:218 (State type definition)
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

On each `continue`, the entire State is rebuilt (immutable update), with the `transition` field recording the reason for the state transition. This is a classic **Mealy Machine** — output depends on the combination of current state and input events.

### 2.2.3 Backpressure and Resource Management

Backpressure is critical in LLM streaming. Claude Code's approach:

1. **Natural throttling with for-await-of**: `claude.ts`'s stream consumption uses `for await (const part of stream)`, where the consumer's processing speed directly determines the producer's push rate
2. **Stream idle watchdog** (`claude.ts:2397`): if no chunk is received in 90 seconds, actively abort the stream and fallback to non-streaming
3. **Generator lifecycle guarantee**: `finally` block ensures `releaseStreamResources()` runs at all exit paths (including `.return()` and exceptions)

### 2.2.4 Why These Theories Are Especially Important in LLM Scenarios

Traditional HTTP API calls are a "request-response" model with simple error handling. LLM agentic loops face unique challenges:

- **A single call can last 10 minutes** (non-streaming limit)
- **Mid-response, new I/O may be triggered** (tool calls)
- **Context window is a stateful scarce resource**, requiring tradeoffs between "compression information loss" and "overflow crash"
- **Costs accumulate in real time**, requiring interruption at any point

These constraints make Event Loop + state machine + Backpressure indispensable theoretical foundations.

---

## 2.3 Architecture and Data Structures

### 2.3.1 QueryEngine Class Core Interface

```typescript
// QueryEngine.ts:155-166
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
```

**Design point:** QueryEngine is per-conversation. Each `submitMessage()` call is a new turn in the same conversation; state persists across turns.

### 2.3.2 Key Type Definitions

**QueryEngineConfig** (`QueryEngine.ts:95-153`) — immutable configuration passed at construction time:

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**QueryConfig** (`query/config.ts:18-31`) — per-query immutable environment snapshot:

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

Note the design intent in the source comment (`config.ts:9-12`): "Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." This is the clear boundary between bun:bundle compile optimization and runtime configuration.

### 2.3.3 Inter-Module Dependency Diagram

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- session state management
                    |  (1,295 lines)  |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- main loop & state machine
                    |  (1,729 lines)  |  <-- State, while(true) loop
                    +--------+--------+
                             |
              +--------------+------------------+
              |              |                  |
              v              v                  v
    +---------+---+  +-------+--------+  +------+-------+
    | query/      |  | query/         |  | query/       |
    | config.ts   |  | tokenBudget.ts |  | stopHooks.ts |
    | (46 lines)  |  | (93 lines)     |  | (473 lines)  |
    +-------------+  +----------------+  +--------------+
              |              |
              v              v
    +---------+--------------+--------+
    |         query/deps.ts           |
    |   QueryDeps (DI interface)      |
    +---------+-----------------------+
              |
              |  callModel / autocompact / microcompact
              v
    +---------+-----------+     +----------+--------+
    | services/api/       |     | cost-tracker.ts   |
    | claude.ts           |     | (323 lines)       |
    | (3,419 lines)       |     | addToTotalSession |
    | queryModelWith      |     | Cost, formatCost  |
    | Streaming           |     +-------------------+
    +---------+-----------+
              |
              v
    +---------+-----------+
    | withRetry / stream  |
    | SSE parsing         |
    | Non-streaming       |
    | fallback            |
    +---------------------+
```

**The dependency injection design in `deps.ts`** (`deps.ts:18-37`) deserves special attention:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" — this is a best practice for TypeScript dependency injection: no need to write interfaces manually; signatures automatically track implementations.

---

## 2.4 Core Algorithms and Flow

### 2.4.1 query() Main Loop Complete Execution Flow

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    query() entry
                        |
                +=======+========+  <--- while(true) main loop start
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        blocking-limit check     |
                |                |
                v                |
        callModel() [streaming]  |
                |                |
        +-------+--------+      |
        | streaming event  |     |
        | consumption      |     |
        | tool_use collect |     |
        | streaming exec   |     |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           YES           |
       |              \          |
       v               v        |
  +---------+   runTools() or   |
  | recovery|   getRemainingR() |
  +---------+         |         |
       |              v         |
       |     getAttachments()   |
       |     memory prefetch    |
       |     skill discovery    |
       |              |         |
       v              v         |
  handleStopHooks()   |         |
       |         maxTurns?      |
       |              |         |
       v         state = next   |
  checkTokenBudget()  |         |
       |              +---------+
       v
  return Terminal { reason }
```

### 2.4.2 Tool Call Loop

The core innovation in tool calls is **streaming tool execution** — starting to execute tools while the LLM streaming response is *still in progress*:

```typescript
// query.ts:443 (StreamingToolExecutor initialization)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

In the streaming consumption loop (`query.ts:536`):

```typescript
if (
  streamingToolExecutor &&
  !toolUseContext.abortController.signal.aborted
) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

Tools are submitted for execution at `content_block_stop`, not after the entire assistant response ends. This means if the LLM outputs 3 tool_use blocks, the first two may have already finished executing while the third is still streaming.

### 2.4.3 Concrete Implementation of Streaming Processing

`claude.ts`'s `queryModel()` manually implements SSE stream parsing, **intentionally bypassing the Anthropic SDK's BetaMessageStream**:

```typescript
// claude.ts comment (~line 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

Streaming state accumulation model:

```
message_start → partialMessage = part.message, usage = initial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → yield AssistantMessage (per block!)
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (terminate)
```

Key design: **each content block independently yields one AssistantMessage**. This means when an LLM reply contains text + tool_use, the UI can display the text immediately after it's complete, without waiting for the tool_use JSON to complete.

### 2.4.4 5-Layer Error Recovery Mechanism

Claude Code's error recovery architecture is defense in depth, with 5 layers:

**Layer 1: withRetry (API level)** — `withRetry()` in `claude.ts` handles 429 (rate limit), 529 (overload), 5xx, and other retryable errors, with exponential backoff and model fallback.

**Layer 2: Streaming → Non-streaming Fallback** — when the streaming connection drops (`claude.ts:2592`):

```typescript
// Fall back to non-streaming mode with retries
const result = yield* executeNonStreamingRequest(...)
```

Also includes stream idle watchdog (90s no-data timeout) and 404 stream creation fallback.

**Layer 3: max_output_tokens recovery** — 3-step progressive recovery in `query.ts`:

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// Step 1: escalate to 64k tokens (ESCALATED_MAX_TOKENS)
// Steps 2-4: inject meta message requiring "Resume directly — no apology, no recap"
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**Layer 4: Prompt-too-long recovery** — three-level cascade:
1. Context collapse drain (drain already-staged collapses)
2. Reactive compact (emergency full compaction)
3. Surface error and exit (avoid infinite loops)

Source code has explicit guards against infinite loops (note in `query.ts`):

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**Layer 5: Model Fallback** — in `query.ts:673-719`, when `FallbackTriggeredError` is caught, switch to the fallback model and retry the entire request.

### 2.4.5 Token Counting and Budget Management

The token budget system has two independent mechanisms:

**Mechanism A: API task_budget** — a server-side token budget that tracks across compaction boundaries:

```typescript
// query.ts:270-280 (taskBudgetRemaining cross-compaction tracking)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**Mechanism B: Client-side token budget auto-continue** (`tokenBudget.ts`) — automatically continues writing when turn output hasn't reached 90% of the budget:

```typescript
// tokenBudget.ts:46-62
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // sub-agents don't do auto-continue
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }
  // ...
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
  // ...
}
```

Note the **diminishing returns detection**: after 3 consecutive continuations where each increment is < 500 tokens, auto-stop to prevent the model from wasting budget on inefficient output.

### 2.4.6 Thinking Mode Handling

The "wizard comment" in the source summarizes 3 ironclad rules for thinking mode (`query.ts:105-118`):

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

Thinking parameter construction logic in `claude.ts` (~line 2242):

```typescript
if (
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING) &&
  modelSupportsAdaptiveThinking(options.model)
) {
  thinking = {
    type: 'adaptive',
  } satisfies BetaMessageStreamParams['thinking']
} else {
  let thinkingBudget = getMaxThinkingTokensForModel(options.model)
  // ...
  thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
  thinking = {
    budget_tokens: thinkingBudget,
    type: 'enabled',
  }
}
```

Models supporting adaptive thinking use adaptive first (no preset budget needed), otherwise fall back to enabled + budget_tokens.

### 2.4.7 Prompt Cache Boundary Management

Claude Code's caching strategy is impressive. The core design is in `addCacheBreakpoints()` (`claude.ts:3045`):

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**Only one cache_control marker** is placed, at the last message (or second-to-last when `skipCacheWrite`), a result of co-optimization with the inference team's KV cache page manager (Mycro).

The 1h TTL cache also has a refined "session stability latching" mechanism (`claude.ts:380-420`) — once eligibility is determined, it's fixed for the entire session, preventing mid-session GrowthBook config changes from flipping the cache_control TTL and breaking the cache.

---

## 2.5 Design Decision Analysis

### 2.5.1 Key Tradeoffs

**Tradeoff 1: Streaming execution vs. completeness guarantee**

StreamingToolExecutor starts executing tools while the LLM is still outputting, bringing significant latency improvements but also introducing complexity — if streaming falls back mid-way, already-executed tools need to be discarded:

```typescript
// query.ts:534-538 (cleanup during streaming fallback)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

This problem has caused bugs (see `claude.ts:2575` comment referencing inc-4258: double tool execution).

**Tradeoff 2: Cache stability vs. dynamic features**

Multiple beta headers use the "sticky-on latch" pattern (`claude.ts:2102-2126`) — once activated, they're maintained for the entire session even if the feature is disabled:

```typescript
// claude.ts:2104 comment
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

This is an explicit tradeoff of "cache hit rate over feature flexibility."

**Tradeoff 3: State machine vs. recursion**

The main loop evolved from early recursive `query()` calls to `while(true)` + State reconstruction. Source code comments still have `query_recursive_call` checkpoint names, but the implementation is now iterative. Benefits:
- No stack overflow risk (long conversations may have hundreds of turns)
- State reconstruction is explicit, making debugging easier
- The `transition` field provides a complete audit trail of state transitions

### 2.5.2 Known Issues Revealed by Source Comments

1. **SDK text delta duplication** (`claude.ts:2350`):

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Non-streaming fallback conflicts with streaming tool execution** (`claude.ts:2575`):

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Task budget token counting deviation across compaction** (`query.ts:268`):

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 Design Differences from LangChain and Similar Solutions

| Dimension | Claude Code Query Engine | LangChain AgentExecutor |
|-----------|--------------------------|-------------------------|
| Streaming primitive | ES Async Generator (native) | Callback + Stream wrapper |
| State management | Explicit State struct + immutable updates | Mutable AgentState dict |
| Tool execution | Streamable parallel (StreamingToolExecutor) | Serial await |
| Retry | 5-layer defense in depth + model fallback | Simple max_iterations |
| Dependency injection | QueryDeps + typeof signature sync | Runtime duck typing |
| Caching | Deep co-optimization with inference KV cache | None (black-box API calls) |

The most fundamental difference: Claude Code is **inference-aware** — it understands the physical mechanism of Prompt Cache (Mycro page manager) and optimizes accordingly, while open-source frameworks can only treat the API as a black box.

---

## 2.6 Transferable Patterns

### 2.6.1 General Engineering Patterns Distilled from the Query Engine

**Pattern 1: Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

Recording the **reason** for each state transition in state makes debugging and telemetry first-class concerns. Any system requiring multi-step decisions can adopt this.

**Pattern 2: Typed Dependency Injection via `typeof`**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

No need to write interfaces manually; signatures automatically sync with implementations. Suitable for any system that needs to mock heavyweight I/O.

**Pattern 3: Withholding Pattern (delayed error exposure)**

For recoverable errors (prompt-too-long, max_output_tokens), don't yield to consumers first; after recovery logic executes, decide whether to expose:

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

This prevents SDK consumers from receiving false error signals when "error has already recovered."

**Pattern 4: Session-stable Latching**

For config items that affect cache keys, once activated, lock for the entire session:

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // write to bootstrap state
}
```

Applicable to any scenario where "config changes would invalidate expensive resources."

### 2.6.2 What Doramagic Can Learn

Doramagic's `flow_controller` pipeline faces similar orchestration problems to the Query Engine, and can learn from:

1. **State + Transition pattern**: Doramagic's 12-state machine can use a similar `{ state, transition: { reason } }` structure, simplifying debugging while natively supporting log auditing
2. **Dependency Injection via typeof**: Doramagic's LLM call layer can use the QueryDeps pattern, injecting a fake model in tests rather than mocking entire modules
3. **Diminishing returns detection** (`tokenBudget.ts`): Doramagic's Soul Extractor can use the same "stop if N consecutive increments are below threshold" strategy during iterative refinement, avoiding LLM wasting tokens on low-quality output

---

## 2.7 Source Index

| File | LOC | One-line responsibility |
|------|-----|------------------------|
| `QueryEngine.ts` | 1,295 | Session-level owner: holds mutableMessages, totalUsage, translates submitMessage() into query() calls, handles SDK message routing |
| `query.ts` | 1,729 | Main loop state machine: while(true) orchestrates compaction → API call → tool execution → stop hooks → budget check |
| `query/config.ts` | 46 | Immutable QueryConfig snapshot: sessionId + 4 runtime gates (feature() gates intentionally excluded to preserve tree-shaking) |
| `query/tokenBudget.ts` | 93 | Client-side token budget auto-continue: 90% completion threshold + diminishing returns early stop |
| `query/stopHooks.ts` | 473 | Turn-end hook orchestration: Stop hooks → TaskCompleted hooks → TeammateIdle hooks, supports blocking error injection |
| `query/deps.ts` | 40 | Injection interface for 4 I/O dependencies: callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3,419 | Full API lifecycle: parameter construction → stream creation → SSE parsing → content block accumulation → cost calculation → non-streaming fallback → cache breakpoint management |
| `cost-tracker.ts` | 323 | Session-level cost accumulation: per-model usage tracking, session persistence/recovery, formatted output |
