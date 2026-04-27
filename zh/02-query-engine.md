# 第 2 章：Query Engine — LLM 交互核心

## 2.1 概述与定位

**一句话定位：** Query Engine 是 Claude Code 的"心跳"——它拥有 LLM 对话的完整生命周期，将用户输入转化为经过权限控制、工具执行、上下文压缩、成本追踪的多轮 agentic 交互。

**它解决的核心问题：** 如何在一个 async generator pipeline 中，可靠地编排"LLM 流式响应 -> 工具调用 -> 结果注入 -> 再次请求"这个循环，同时处理 context window 溢出、token 预算、API 故障、用户中断、权限审批等至少 12 种异常分支。

**涉及文件与代码量统计：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `QueryEngine.ts` | 1,295 | 会话级状态管理，SDK/headless 入口 |
| `query.ts` | 1,729 | 查询主循环，工具执行编排，多层恢复 |
| `query/config.ts` | 46 | 不可变配置快照 |
| `query/tokenBudget.ts` | 93 | Token 预算 auto-continue 决策 |
| `query/stopHooks.ts` | 473 | 停止时钩子（安全门禁、后处理） |
| `query/deps.ts` | 40 | 依赖注入接口（测试友好） |
| `services/api/claude.ts` | 3,419 | API 调用、流式解析、重试、缓存策略 |
| `cost-tracker.ts` | 323 | 成本追踪与 usage 累积 |
| **总计** | **~7,418** | — |

这 7,400+ 行代码构成了 Claude Code 约 1.4% 的代码量，但它们是整个产品最关键的路径——每一次用户与 Claude 的交互都必须经过这里。

---

## 2.2 理论基础

### 2.2.1 Async Generator Pipeline（协程管道）

Query Engine 的核心架构基于 **ES2018 Async Generator** 作为流式处理原语。`query()` 函数签名揭示了这一设计：

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

这不是偶然的语法选择，而是对 **Coroutine 理论** 的精确应用。Async generator 同时具备：
- **惰性求值**：消费者驱动，不会预先计算所有 API 响应
- **双向通信**：yield 出 stream event，return 出 terminal reason
- **资源安全**：finally 块保证 stream 释放（`claude.ts` 中 `releaseStreamResources()`）

传统的 callback 或 Promise chain 无法同时满足"流式输出给 UI"和"等待工具执行结果"这两个需求。Async generator 天然支持这种"producer 暂停等 consumer 消费"的语义。

### 2.2.2 State Machine（隐式状态机）

`query.ts` 的主循环不是递归调用（尽管早期代码注释仍称 `query_recursive_call`），而是一个 **while(true) + continue 驱动的显式状态机**：

```typescript
// query.ts:218 (State 类型定义)
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

每次 `continue` 时，整个 State 被重建（不可变更新），`transition` 字段记录转移原因。这是经典的 **Mealy Machine**——输出取决于当前状态和输入事件的组合。

### 2.2.3 Backpressure 与资源管理

LLM streaming 中 backpressure 至关重要。Claude Code 的方案：

1. **for-await-of 天然限流**：`claude.ts` 的 stream 消费使用 `for await (const part of stream)`，consumer 的处理速度直接决定 producer 的推送速率
2. **Stream idle watchdog**（`claude.ts:2397`）：如果 90 秒没有收到 chunk，主动 abort stream 并 fallback 到 non-streaming
3. **Generator lifecycle guarantee**：`finally` 块确保 `releaseStreamResources()` 在所有退出路径（包括 `.return()` 和异常）执行

### 2.2.4 为什么这些理论在 LLM 场景特别重要

传统 HTTP API 调用是"请求-响应"模型，错误处理简单。LLM agentic loop 面临独特挑战：

- **单次调用可持续 10 分钟**（non-streaming limit）
- **响应中途可能触发新的 I/O**（工具调用）
- **context window 是有状态的稀缺资源**，需要在"压缩信息损失"和"溢出崩溃"之间权衡
- **成本实时累积**，需要随时可中断

这些约束使得 Event Loop + 状态机 + Backpressure 成为不可或缺的理论支撑。

---

## 2.3 架构与数据结构

### 2.3.1 QueryEngine 类核心接口

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

**设计要点：** QueryEngine 是 per-conversation 的。每个 `submitMessage()` 调用是同一对话中的新 turn，状态跨 turn 保持。

### 2.3.2 关键类型定义

**QueryEngineConfig**（`QueryEngine.ts:95-153`）——构造时传入的不可变配置：

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

**QueryConfig**（`query/config.ts:18-31`）——per-query 不可变环境快照：

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

注意源码注释的设计意图（`config.ts:9-12`）："Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." 这是 bun:bundle 编译优化与运行时配置的明确分界。

### 2.3.3 模块间依赖关系图

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- 会话状态管理
                    |  (1,295 lines)  |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- 主循环 & 状态机
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

**deps.ts 的依赖注入设计**（`deps.ts:18-37`）值得特别注意：

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" —— 这是 TypeScript 依赖注入的最佳实践：无需手写接口，签名自动跟踪实现。

---

## 2.4 核心算法与流程

### 2.4.1 query() 主循环完整执行流程

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    query() 入口
                        |
                +=======+========+  <--- while(true) 主循环开始
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        blocking-limit 检查      |
                |                |
                v                |
        callModel() [流式]       |
                |                |
        +-------+--------+      |
        | 流式事件消费     |      |
        | tool_use 收集    |      |
        | streaming exec  |      |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           YES           |
       |              \          |
       v               v        |
  +---------+   runTools() 或   |
  | 恢复逻辑 |   getRemainingR() |
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

### 2.4.2 工具调用循环（Tool-call Loop）

工具调用的核心创新是 **streaming tool execution**——在 LLM 流式响应*还在进行时*就开始执行工具：

```typescript
// query.ts:443 (StreamingToolExecutor 初始化)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

在流式消费循环中（`query.ts:536`）：

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

工具在 `content_block_stop` 时就被提交执行，而不是等待整个 assistant 响应结束。这意味着如果 LLM 输出了 3 个 tool_use block，前两个可能在第三个还在流式传输时已经执行完毕。

### 2.4.3 流式处理的具体实现

`claude.ts` 的 `queryModel()` 手工实现了 SSE stream 解析，**故意绕开了 Anthropic SDK 的 BetaMessageStream**：

```typescript
// claude.ts 注释 (约 line 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

流式状态累积模型：

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
message_stop → (终止)
```

关键设计：**每个 content block 独立 yield 一个 AssistantMessage**。这意味着一个 LLM 回复中包含 text + tool_use 时，UI 可以在 text 完成后立即显示，无需等待 tool_use 的 JSON 完成。

### 2.4.4 重试与错误恢复的 5 层机制

Claude Code 的错误恢复架构是纵深防御（defense in depth），共 5 层：

**第 1 层：withRetry（API 级）** —— `claude.ts` 中的 `withRetry()` 处理 429 (rate limit)、529 (overload)、5xx 等可重试错误，包含指数退避和 model fallback。

**第 2 层：Streaming → Non-streaming Fallback** —— 当流式连接中断时（`claude.ts:2592`）：

```typescript
// Fall back to non-streaming mode with retries
const result = yield* executeNonStreamingRequest(...)
```

同时包含 stream idle watchdog（90s 无数据超时）和 404 stream creation fallback。

**第 3 层：max_output_tokens 恢复** —— `query.ts` 中的 3 次递进恢复：

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// 第一步：escalate 到 64k tokens（ESCALATED_MAX_TOKENS）
// 第二步-四步：注入 meta message 要求 "Resume directly — no apology, no recap"
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**第 4 层：Prompt-too-long 恢复** —— 三级联动：
1. Context collapse drain（排空已暂存的折叠）
2. Reactive compact（应急全量压缩）
3. Surface error 并退出（避免死循环）

源码中有防止无限循环的显式守卫（`query.ts` 注释）：

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**第 5 层：Model Fallback** —— `query.ts:673-719` 中，当 `FallbackTriggeredError` 被捕获时，切换到 fallback model 并重试整个请求。

### 2.4.5 Token 计数与预算管理

Token 预算系统分为两个独立机制：

**机制 A：API task_budget** —— 服务器端感知的 token 预算，跨 compaction 边界跟踪：

```typescript
// query.ts:270-280 (taskBudgetRemaining 跨压缩追踪)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**机制 B：Client-side token budget auto-continue**（`tokenBudget.ts`）—— 当 turn output 未达到预算 90% 时自动续写：

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
  // 子 agent 不做 auto-continue
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

注意 **diminishing returns 检测**：连续 3 次 continuation 且每次增量 < 500 tokens 时自动停止，防止模型在低效输出上浪费预算。

### 2.4.6 Thinking Mode 处理

源码中的"wizard comment"总结了 thinking mode 的 3 条铁律（`query.ts:105-118`）：

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

`claude.ts` 中的 thinking 参数构建逻辑（约 line 2242）：

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

支持 adaptive thinking 的模型优先使用 adaptive（无需预设 budget），否则退回到 enabled + budget_tokens。

### 2.4.7 Prompt Cache 边界管理

Claude Code 的缓存策略令人印象深刻。核心设计在 `addCacheBreakpoints()` 中（`claude.ts:3045`）：

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**只放一个 cache_control marker**，位置在最后一条消息（或 `skipCacheWrite` 时放在倒数第二条），这是与 inference 团队的 KV cache 页管理器（Mycro）协同优化的结果。

1h TTL 缓存还有精细的"session stability latching"机制（`claude.ts:380-420`）——eligibility 一旦确定就固定整个 session，防止中途 GrowthBook 配置变化导致 cache_control TTL 翻转而打破缓存。

---

## 2.5 设计决策分析

### 2.5.1 关键 Tradeoff

**Tradeoff 1：流式执行 vs 完整性保证**

StreamingToolExecutor 在 LLM 还在输出时就开始执行工具，带来了显著的延迟优化，但也引入了复杂性——如果 streaming 中途 fallback，已执行的工具需要被 discard：

```typescript
// query.ts:534-538 (streaming fallback 时清理)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

这个问题已经导致过 bug（见 `claude.ts:2575` 注释引用 inc-4258：double tool execution）。

**Tradeoff 2：缓存稳定性 vs 动态功能**

多个 beta header 使用 "sticky-on latch" 模式（`claude.ts:2102-2126`）——一旦激活就保持整个 session，即使功能被关闭：

```typescript
// claude.ts:2104 注释
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

这是"缓存命中率优先于功能灵活性"的明确取舍。

**Tradeoff 3：状态机 vs 递归**

主循环从早期的递归 `query()` 调用演进为 `while(true)` + State 重建。源码注释中仍有 `query_recursive_call` checkpoint 名称，但实际已是迭代。好处是：
- 无栈溢出风险（长对话可能有数百 turns）
- State 重建是显式的，便于调试
- `transition` 字段提供了完整的状态转移审计线索

### 2.5.2 源码注释暴露的已知问题

1. **SDK text delta 重复**（`claude.ts:2350`）：

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Non-streaming fallback 与 streaming tool execution 冲突**（`claude.ts:2575`）：

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Task budget 跨 compaction 的 token 计数偏差**（`query.ts:268`）：

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 与 LangChain 等方案的设计差异

| 维度 | Claude Code Query Engine | LangChain AgentExecutor |
|------|--------------------------|-------------------------|
| 流式原语 | ES Async Generator（原生） | Callback + Stream wrapper |
| 状态管理 | 显式 State struct + 不可变更新 | Mutable AgentState dict |
| 工具执行 | 可流式并行（StreamingToolExecutor） | 串行 await |
| 重试 | 5 层纵深防御 + model fallback | 简单 max_iterations |
| 依赖注入 | QueryDeps + typeof 签名同步 | 运行时 duck typing |
| 缓存 | 与 inference KV cache 深度协同 | 无（黑盒 API 调用） |

最根本的差异：Claude Code 是 **inference-aware** 的——它理解 prompt cache 的物理机制（Mycro 页管理器）并据此优化，而开源框架只能把 API 当黑盒。

---

## 2.6 可迁移模式

### 2.6.1 从 Query Engine 中提炼的通用工程模式

**模式 1：Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

将每次状态转移的**原因**记录在 state 中，使得调试和遥测成为 first-class concern。任何需要多步决策的系统都可以采用。

**模式 2：Typed Dependency Injection via `typeof`**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

无需手写 interface，签名自动与实现同步。适用于任何需要 mock 重量级 I/O 的系统。

**模式 3：Withholding Pattern（延迟错误暴露）**

对于可恢复的错误（prompt-too-long, max_output_tokens），先不 yield 给消费者，在恢复逻辑执行后再决定是否暴露：

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

这防止 SDK 消费者在"错误已恢复"的情况下收到虚假的错误信号。

**模式 4：Session-stable Latching**

对影响缓存键的配置项，一旦激活就锁定整个 session：

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // 写入 bootstrap state
}
```

适用于任何有"配置变化会导致 expensive 资源失效"的场景。

### 2.6.2 Doramagic 可借鉴之处

Doramagic 的 `flow_controller` 管线与 Query Engine 面临相似的编排问题，可借鉴：

1. **State + Transition 模式**：Doramagic 的 12 状态机可以用类似的 `{ state, transition: { reason } }` 结构，既简化调试又天然支持日志审计
2. **Dependency Injection via typeof**：Doramagic 的 LLM 调用层可以用 QueryDeps 模式，测试时注入 fake model 而不是 mock 整个 module
3. **Diminishing returns 检测**（`tokenBudget.ts`）：Doramagic 的 Soul Extractor 在 iterative refinement 时可以用同样的"连续 N 次增量低于阈值则停止"策略，避免 LLM 在低质量输出上浪费 tokens

---

## 2.7 源码索引

| 文件 | 行数 | 一句话职责 |
|------|------|----------|
| `QueryEngine.ts` | 1,295 | 会话级 owner：持有 mutableMessages、totalUsage，将 submitMessage() 翻译为 query() 调用，处理 SDK 消息路由 |
| `query.ts` | 1,729 | 主循环状态机：while(true) 编排 compaction → API call → tool execution → stop hooks → budget check |
| `query/config.ts` | 46 | 不可变 QueryConfig 快照：sessionId + 4 个 runtime gates（feature() gates 刻意排除以保留 tree-shaking） |
| `query/tokenBudget.ts` | 93 | Client-side token budget auto-continue：90% 完成度阈值 + diminishing returns 提前停止 |
| `query/stopHooks.ts` | 473 | Turn-end 钩子编排：Stop hooks → TaskCompleted hooks → TeammateIdle hooks，支持 blocking error 回注 |
| `query/deps.ts` | 40 | 4 个 I/O 依赖的注入接口：callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3,419 | API 全生命周期：参数构建 → stream 创建 → SSE 解析 → content block 累积 → cost 计算 → non-streaming fallback → cache breakpoint 管理 |
| `cost-tracker.ts` | 323 | Session 级成本累积：per-model usage 追踪、session 持久化/恢复、格式化输出 |
