# 第 8 章：上下文管理

## 8.1 概述与定位

上下文管理是 Claude Code 架构中最关键的子系统之一。一个典型的编程会话可能持续数小时，涉及数百次工具调用，产生数十万 token 的对话历史。如果不加管理，上下文窗口会在 20-30 轮交互后耗尽，导致会话中断。

Claude Code 的上下文管理系统解决的核心问题是：**如何在有限的上下文窗口（通常 200K token）内，保持会话的连续性和信息的完整性，同时最小化用户感知到的信息损失？**

该系统由 `services/compact/` 目录下的 11 个文件组成，总计约 3,900 行 TypeScript 代码，辅以 `utils/collapseReadSearch.ts`（1,109 行）和 `utils/toolResultStorage.ts`（1,040 行）两个关键工具模块。整个子系统的设计体现了三个核心原则：

1. **渐进式降级**（Graceful Degradation）：从零成本的微压缩到有损的全量压缩，逐级增加干预力度
2. **缓存优先**（Cache-First）：每个压缩决策都优先考虑 prompt cache 的保全
3. **安全性保证**（Safety Invariants）：tool_use/tool_result 配对不可切断，递归保护，断路器机制

## 8.2 理论基础

### 8.2.1 信息论视角：有损压缩 vs 无损压缩

上下文管理本质上是一个**信息压缩问题**。Claude Code 的多层体系对应了不同的压缩策略：

- **无损压缩**（Lossless）：微压缩的 `cache_edits` 路径——通过 API 的 cache editing 机制删除旧工具结果的服务端缓存副本，本地消息内容不变。模型看到的是 `[Old tool result content cleared]` 占位符，但原始数据保存在磁盘上（`toolResultStorage.ts`）。信息没有丢失，只是从热存储移到了冷存储。
- **有损压缩**（Lossy）：全量压缩通过 Fork Agent 生成摘要，将数万 token 的对话压缩为数千 token。这是一个不可逆的降维过程——代码细节、错误堆栈、中间推理都可能丢失。

从 Rate-Distortion Theory 的角度，Claude Code 的设计隐含了一个**失真度量函数**：摘要 prompt 中的 9 个章节（见 8.6 节）定义了哪些信息维度最不能容忍失真——"user messages"（完整保留）优先级高于 "key technical concepts"（允许概括）。

### 8.2.2 缓存理论：时间局部性与空间局部性

微压缩的白名单机制体现了经典缓存理论中的**时间局部性**（Temporal Locality）假设：

> 最近使用的工具结果更可能被后续引用。

`microCompact.ts` 中的白名单（`COMPACTABLE_TOOLS`）就是 eviction policy 的体现——只有特定工具（Read、Shell、Grep、Glob、WebFetch、WebSearch、Edit、Write）的结果可以被清除，因为它们的输出具有可再生性（可以重新执行工具获取）。而用户手工输入的文本、图片等不可再生内容则永远不会被清除。

```typescript
// microCompact.ts:30-41 — 可压缩工具白名单
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

`keepRecent` 参数（默认保留最近 5 个）则直接实现了 LRU（Least Recently Used）淘汰策略。

### 8.2.3 断路器模式（Circuit Breaker Pattern）

`autoCompact.ts` 中的断路器机制是分布式系统中经典的 Circuit Breaker Pattern 在 LLM 应用中的精确适配。该模式源自 Michael Nygard 的《Release It!》，其三态模型（Closed → Open → Half-Open）在 Claude Code 中的实现：

```typescript
// autoCompact.ts:70-73 — 断路器阈值
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

这个注释揭示了断路器存在之前的真实灾难数据：**1,279 个会话陷入了 50 次以上的连续失败循环**，最严重的单会话达到 3,272 次失败尝试，全球每天浪费约 250K 次 API 调用。断路器的引入将最大重试次数限制为 3 次。

| 状态 | 行为 | 对应代码 |
|------|------|---------|
| Closed（正常） | `consecutiveFailures < 3`，正常尝试压缩 | `autoCompactIfNeeded` 默认路径 |
| Open（熔断） | `consecutiveFailures >= 3`，跳过压缩 | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open（探测） | 成功压缩后 `consecutiveFailures` 重置为 0 | `consecutiveFailures: 0` on success |

## 8.3 架构总览

### 8.3.1 多层压缩体系的整体架构

Claude Code 的上下文管理采用**5 层防线**设计。从低干预到高干预排列：

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户请求                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage（预防层）                           │
│   大工具结果 → 磁盘持久化 + 2KB 预览                              │
│   触发: 结果 > 阈值（默认 50K chars）                             │
│   代价: 零上下文代价（存磁盘，preview 在上下文）                    │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact（微压缩）                                  │
│   路径 A: 时间触发 — 清除旧工具结果内容                            │
│   路径 B: 缓存编辑 — cache_edits API 删除服务端缓存               │
│   触发: 每次 API 调用前                                          │
│   代价: 极低（工具结果被占位符替换，可通过磁盘恢复）                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact（自动压缩）                                │
│   Session Memory → Fork Agent → 全量摘要                         │
│   触发: token 超过 effectiveContextWindow - 13K                  │
│   代价: 高（有损摘要，丢失细节，消耗一次 API 调用）                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact（手动压缩）                              │
│   用户主动触发，支持 Partial Compact                               │
│   触发: 用户命令                                                  │
│   代价: 同上                                                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact（响应式压缩）                           │
│   API 返回 prompt_too_long → 截断重试                             │
│   触发: 413 错误                                                 │
│   代价: 最高（紧急截断 + 摘要，信息损失最大）                        │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 各层触发条件、代价、信息损失对比

| 层级 | 触发条件 | 时机 | 延迟 | 信息损失 | API 成本 |
|------|---------|------|------|---------|---------|
| L0: Tool Result Storage | 单个工具结果 > 阈值 | 工具执行后 | 磁盘 I/O | 零（原文存磁盘） | 零 |
| L1a: Time-based MC | 距上次 assistant > 60min | API 调用前 | 零（本地操作） | 低（旧结果清除） | 零 |
| L1b: Cached MC | 可压缩工具数超过阈值 | API 调用前 | 零（cache_edits） | 低（同上） | 零 |
| L2: Auto-Compact | token > threshold | 轮次间 | 5-15s（API 调用） | 高（有损摘要） | 1 次 API 调用 |
| L3: Manual Compact | 用户 /compact | 用户触发 | 同上 | 中-高（用户可指导） | 1 次 API 调用 |
| L4: Reactive Compact | prompt_too_long 413 | API 失败后 | 10-30s（重试） | 最高（截断+摘要） | 1-4 次 API 调用 |

### 8.3.3 数据流向

```
消息数组 (Message[])
    │
    ▼
microcompactMessages()  ──→ [时间触发?] ──Y──→ 内容清除 → 返回
    │ N                      │
    │                  [缓存编辑?] ──Y──→ pendingCacheEdits → 返回
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     不做压缩，直接返回
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [有 session memory?]
    │ N                              │ Y
    ▼                                ▼
compactConversation()           calculateMessagesToKeepIndex()
    │                                │
    ▼                                ▼
streamCompactSummary()          buildPostCompactMessages()
    │ (Fork Agent)
    ▼
formatCompactSummary()
    │
    ▼
buildPostCompactMessages()
    │
    ▼
新消息数组: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 第 1 层：微压缩（Microcompact）

微压缩是上下文管理的第一道防线。它在**每次 API 调用前**执行（`microcompactMessages` 入口），目标是以最小代价释放上下文空间。

### 8.4.1 可压缩工具白名单

微压缩只对特定工具的输出进行操作。白名单背后的设计原则是：**只清除可再生的内容**。

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // 文件读取 — 可重新读取
  ...SHELL_TOOL_NAMES,     // Shell 命令 — 可重新执行
  GREP_TOOL_NAME,          // 搜索 — 可重新搜索
  GLOB_TOOL_NAME,          // 文件匹配 — 可重新匹配
  WEB_SEARCH_TOOL_NAME,    // 网络搜索 — 可重新搜索
  WEB_FETCH_TOOL_NAME,     // 网络抓取 — 可重新抓取
  FILE_EDIT_TOOL_NAME,     // 文件编辑 — 结果已保存到磁盘
  FILE_WRITE_TOOL_NAME,    // 文件写入 — 同上
])
```

注意 `apiMicrocompact.ts` 中还定义了一组更细粒度的区分：

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // 清除 tool_result 内容
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // 清除 tool_use 输入
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

这里的区分很精妙：对于 Read/Grep/Shell，清除的是**输出**（tool_result）；对于 Edit/Write，清除的是**输入**（tool_use input），因为编辑操作的输入（差异内容）体积大但结果已持久化到磁盘。

### 8.4.2 两个子路径详解

微压缩有两个互斥的执行路径，由 `microcompactMessages()` 函数统一调度：

```typescript
// microCompact.ts:287-317 — 调度逻辑
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // 路径 A: 时间触发 — 优先级最高，短路后续路径
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // 路径 B: 缓存编辑 — 仅主线程，仅支持特定模型
  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    const model = toolUseContext?.options.mainLoopModel ?? getMainLoopModel()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  return { messages }
}
```

**路径 A: Time-based Microcompact（时间触发）**

当用户离开会话超过配置的时间阈值（默认 60 分钟）后返回时触发。设计理由在 `timeBasedMCConfig.ts` 中有明确阐述：

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

服务端 prompt cache 的 TTL 是 1 小时。用户离开超过 1 小时意味着**缓存必然失效**，整个 prompt prefix 都需要重新写入。此时清除旧工具结果是"免费"的——不会造成额外的缓存失效代价。

时间触发的关键逻辑：

```typescript
// microCompact.ts:381-389 — 时间触发评估
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) {
    return null
  }
  const gapMinutes =
    (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  return { gapMinutes, config }
}
```

时间触发后的清除策略也用 LRU（`keepRecent` 默认 5），但有一个边界保护：

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

这个 `Math.max(1, ...)` 是防止 `keepRecent=0` 时 `slice(-0)` 返回全部数组的 JavaScript 陷阱——一个典型的"防御性编程避免语义歧义"案例。

时间触发后还需要重置缓存编辑状态：

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**路径 B: Cached Microcompact（缓存编辑）**

这是 Anthropic 内部的高级优化路径（`feature('CACHED_MICROCOMPACT')`），利用 API 的 `cache_edits` 机制在**不修改本地消息内容**的情况下删除服务端缓存中的工具结果。

```typescript
// microCompact.ts:327-370 — 缓存编辑路径核心
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // 注册工具结果
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && 
            compactableToolIds.has(block.tool_use_id) &&
            !state.registeredTools.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  const toolsToDelete = mod.getToolResultsToDelete(state)
  if (toolsToDelete.length > 0) {
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits
    }
    // ...
    return {
      messages,  // 消息不变!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

关键设计决策：**消息数组不变**——`return { messages }` 返回原始引用。缓存编辑发生在 API 层（通过 `cache_edits` 参数），本地状态保持完整。这意味着如果 API 调用失败或被重试，没有任何本地副作用。

### 8.4.3 缓存编辑的状态管理

缓存编辑路径维护三组关键状态：

```typescript
// microCompact.ts:43-49 — 模块级状态
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

这三个状态的生命周期管理是微妙的：

- `pendingCacheEdits` 是一次性的——`consumePendingCacheEdits()` 读取后清空（`microCompact.ts:80-84`），调用者必须在 API 请求中发送后将其 pin 住。
- `pinnedCacheEdits` 是累积的——每次成功的 cache edit 都被 pin 到特定的 user message 位置，后续请求必须在相同位置重新发送以保证缓存命中。
- `cachedMCState` 在压缩后（`resetMicrocompactState()`）或时间触发后被重置。

```typescript
// microCompact.ts:78-105 — 状态消费与 pin
export function consumePendingCacheEdits() {
  const edits = pendingCacheEdits
  pendingCacheEdits = null
  return edits
}

export function getPinnedCacheEdits() {
  if (!cachedMCState) return []
  return cachedMCState.pinnedEdits
}

export function pinCacheEdits(
  userMessageIndex: number,
  block: import('./cachedMicrocompact.js').CacheEditsBlock,
): void {
  if (cachedMCState) {
    cachedMCState.pinnedEdits.push({ userMessageIndex, block })
  }
}
```

### 8.4.4 Token 估算辅助函数

微压缩模块提供了全系统共享的 token 估算函数：

```typescript
// microCompact.ts:155-194 — estimateMessageTokens
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue
    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 固定 2000
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'tool_use') {
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {}),
        )
      }
      // ...
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // 4/3 保守填充
}
```

`roughTokenCountEstimation` 的核心公式极其简洁：`Math.round(content.length / 4)`（`tokenEstimation.ts:203-207`）。最终 `estimateMessageTokens` 对此结果再乘以 4/3 保守系数，等价于 `text.length / 3`。这个双重保守策略确保低估的概率极低。

## 8.5 第 2 层：自动压缩（Auto-Compact）

### 8.5.1 阈值计算公式

自动压缩的触发阈值由以下公式计算：

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

具体数值推导（以 Claude Opus 200K 为例）：

```
contextWindow = 200,000
maxOutputTokens = 16,384 (或模型特定值)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (基于 p99.99 = 17,387)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — 关键常量
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

`AUTOCOMPACT_BUFFER_TOKENS = 13,000` 的选择是一个工程权衡：太小则压缩太频繁（每次压缩消耗 5-15 秒和一次 API 调用），太大则浪费可用上下文。13K 大约是 3-5 轮普通对话的空间。

### 8.5.2 shouldAutoCompact 决策树

```typescript
// autoCompact.ts:127-178 — 完整决策链
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. 递归保护：session_memory 和 compact 查询源不触发
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. 上下文折叠保护：marble_origami（ctx-agent）不触发
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. 配置检查：用户是否启用
  if (!isAutoCompactEnabled()) return false

  // 4. 响应式模式：如果启用，抑制主动压缩
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. 上下文折叠模式：折叠 IS 上下文管理，压缩不应干预
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Token 计数 + 阈值比较
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

这个决策树暴露了 Claude Code 正在并行实验的多个上下文管理策略：
- **Reactive Compact**（`tengu_cobalt_raccoon`）：不主动压缩，等 API 报 prompt_too_long 再响应
- **Context Collapse**（`CONTEXT_COLLAPSE`）：以 90% 提交 / 95% 阻塞的流式方式管理上下文
- **Auto Compact**（当前默认）：主动在阈值处压缩

三者互斥，通过 feature flag 控制。

### 8.5.3 断路器机制

```typescript
// autoCompact.ts:219-272 — autoCompactIfNeeded 含断路器
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // 断路器检查
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // 熔断状态，直接跳过
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // 优先尝试 Session Memory 压缩
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 传统压缩
  try {
    const compactionResult = await compactConversation(...)
    runPostCompactCleanup(querySource)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const nextFailures = (tracking?.consecutiveFailures ?? 0) + 1
    if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
      logForDebugging(
        `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`,
        { level: 'warn' })
    }
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

### 8.5.4 autoCompactIfNeeded 执行流程

完整执行顺序：

1. **环境变量检查**：`DISABLE_COMPACT` → 全局禁用
2. **断路器检查**：`consecutiveFailures >= 3` → 跳过
3. **阈值检查**：`shouldAutoCompact()` → 多层门禁
4. **Session Memory 压缩**（优先路径）：利用已有的 session memory 替代 API 调用
5. **传统 Fork Agent 压缩**（回退路径）：完整的 API 驱动摘要生成
6. **失败处理**：递增断路器计数器，传递给下一轮

## 8.6 第 3 层：传统压缩（Full Compact）

### 8.6.1 Fork Agent 机制

传统压缩的核心是通过 Fork Agent 生成对话摘要。`streamCompactSummary()` 函数（`compact.ts:1136-1396`）实现了两级回退策略：

**第一级：Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — 缓存共享 fork
if (promptCacheSharingEnabled) {
  try {
    const result = await runForkedAgent({
      promptMessages: [summaryRequest],
      cacheSafeParams,
      canUseTool: createCompactCanUseTool(),
      querySource: 'compact',
      forkLabel: 'compact',
      maxTurns: 1,
      skipCacheWrite: true,
      overrides: { abortController: context.abortController },
    })
    // ...
  }
}
```

Fork Agent 复用了主对话的完整 prompt cache（system prompt + tools + context messages），只追加一条摘要请求。关键设计：

1. `maxTurns: 1` — 不允许多轮交互
2. `canUseTool: createCompactCanUseTool()` — 拒绝所有工具调用
3. `skipCacheWrite: true` — 不写入缓存（临时分叉）
4. **不设 maxOutputTokens** — 注释解释：设置会改变 thinking config，导致 cache key 不匹配

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**第二级：Streaming Fallback**

当 Fork Agent 失败时，回退到直接流式 API 调用，此时**可以**设置 `maxOutputTokensOverride`：

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

流式回退还支持配置驱动的重试（`tengu_compact_streaming_retry`），最多 `MAX_COMPACT_STREAMING_RETRIES = 2` 次。

### 8.6.2 预处理管线

压缩前的消息经过三步预处理：

```typescript
// compact.ts:1293-1300 — 预处理链
normalizeMessagesForAPI(
  stripImagesFromMessages(
    stripReinjectedAttachments([
      ...getMessagesAfterCompactBoundary(messages),
      summaryRequest,
    ]),
  ),
  context.options.tools,
)
```

1. `getMessagesAfterCompactBoundary` — 只取最后一次压缩之后的消息
2. `stripReinjectedAttachments` — 移除 `skill_discovery` / `skill_listing` 附件（它们会在压缩后重新注入）
3. `stripImagesFromMessages` — 将图片块替换为 `[image]` 文本标记（`compact.ts:144-199`）

`stripImagesFromMessages` 的存在理由很实际：

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

CCD（Claude Code Desktop）用户经常附加截图，如果不剥离图片，压缩 API 调用本身可能因 prompt 过长而失败。

### 8.6.3 摘要输出的 9 章节格式

`prompt.ts` 定义了摘要必须遵循的 9 个结构化章节：

```
1. Primary Request and Intent    — 用户意图
2. Key Technical Concepts        — 技术概念
3. Files and Code Sections       — 文件和代码片段
4. Errors and fixes              — 错误和修复
5. Problem Solving               — 问题解决
6. All user messages             — 所有用户消息（非 tool result）
7. Pending Tasks                 — 待处理任务
8. Current Work                  — 当前工作
9. Optional Next Step            — 下一步（可选）
```

章节 6 的设计尤为重要——"List ALL user messages that are not tool results"。这确保了即使对话被压缩，用户的原始表述仍然被完整保留。这是**用户反馈信息零丢失**的保证。

章节 9 有一段精心设计的约束：

```
// prompt.ts — 第 9 章的约束
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

这防止了压缩后的模型"自作主张"——只有用户明确要求的下一步才会被记录。

### 8.6.4 NO_TOOLS_PREAMBLE 防绕过设计

Fork Agent 继承了主对话的完整工具集（为了 cache key 匹配），但压缩 agent 不应该使用任何工具。这形成了一个矛盾：工具在 schema 中存在，但不应被调用。

解决方案是**三层工具拒绝**：

```typescript
// prompt.ts:16-24 — 第一层：prompt 开头的强力声明
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — 第二层：prompt 结尾的重复提醒
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — 第三层：代码级拒绝
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

注释揭示了这三层设计的实际原因：

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

在 Sonnet 4.6 上，仅靠 prompt 指令有 2.79% 的概率仍然尝试工具调用（4.5 上只有 0.01%）。`createCompactCanUseTool` 是最后一道代码级保障。

### 8.6.5 后处理（formatCompactSummary）

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // 剥离 <analysis> 草稿区 — 提升摘要质量的中间推理，不需要保留
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // 提取 <summary> 内容
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${summaryMatch[1].trim()}`)
  }
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')
  return formattedSummary.trim()
}
```

`<analysis>` 标签的设计是一个 Chain-of-Thought 技巧：让模型先在分析区域"打草稿"，然后在 `<summary>` 中输出最终结果。分析区域的存在提升了摘要质量，但在最终输出中被剥离——因为它包含冗余的中间推理，会浪费后续轮次的上下文空间。

### 8.6.6 压缩后消息序列与附件重注入

压缩完成后，新的消息序列由 `buildPostCompactMessages()` 构建：

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // 系统消息：标记压缩边界
    ...result.summaryMessages,    // 用户消息：摘要内容
    ...(result.messagesToKeep ?? []),  // 保留的原始消息
    ...result.attachments,        // 文件附件 + 技能 + 计划
    ...result.hookResults,        // SessionStart 钩子结果
  ]
}
```

附件重注入是一个复杂的过程（`compact.ts:532-585`），包括：

1. **文件附件**：最近访问的前 5 个文件，受 50K token 预算约束，每个文件最多 5K token
2. **计划文件**：如果有活跃计划
3. **计划模式指令**：如果在 plan mode
4. **技能内容**：已调用技能的内容，按最近使用排序，每个最多 5K token，总预算 25K token
5. **Deferred Tools Delta**：重新声明延迟加载工具的 schema
6. **Agent Listing Delta**：重新声明 agent 列表
7. **MCP Instructions Delta**：重新声明 MCP 服务器指令

### 8.6.7 PTL 重试机制（Prompt-Too-Long Recovery）

当压缩 API 调用自身因 prompt 过长而失败时，系统通过渐进式截断重试：

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 先清除之前重试留下的标记消息
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // 精确截断：根据 API 返回的 token 差值
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 模糊截断：丢弃 20% 的消息组
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // 至少保留一组
  if (dropCount < 1) return null

  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

重试上限为 `MAX_PTL_RETRIES = 3`。截断策略有两条路径：
- **精确路径**：API 错误中包含 token 差值 → 按差值逐组丢弃
- **模糊路径**（Vertex/Bedrock 等非标准错误格式）：每次丢弃 20%

注意第 283 行的边界处理：丢弃 group 0 后，消息序列可能以 assistant 消息开头，违反 API 约束（第一条消息必须是 user）。系统插入一条合成的 user 标记消息来修复。

### 8.6.8 部分压缩（Partial Compact）的两个方向

`partialCompactConversation()`（`compact.ts:772-1106`）支持两个方向：

```
Direction 'from': 
  [压缩后保留] | pivot | [被摘要的消息]
  → 保留 prompt cache（保留的在前，cache prefix 不变）

Direction 'up_to':
  [被摘要的消息] | pivot | [压缩后保留]
  → prompt cache 失效（摘要在前，前缀改变）
```

`up_to` 方向有一个额外的清理逻辑——必须从保留消息中移除旧的 compact boundary 和 summary：

```typescript
// compact.ts:791-799
const messagesToKeep =
  direction === 'up_to'
    ? allMessages.slice(pivotIndex)
        .filter(m =>
          m.type !== 'progress' &&
          !isCompactBoundaryMessage(m) &&
          !(m.type === 'user' && m.isCompactSummary))
    : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

注释解释了原因：`up_to` 模式下摘要在保留消息之前，旧的 boundary 会误导 `findLastCompactBoundaryIndex` 的反向扫描。

## 8.7 第 4 层：Session Memory 压缩

### 8.7.1 核心思想与优势

Session Memory 压缩（`sessionMemoryCompact.ts`）是传统压缩的优化替代。核心思想：利用后台持续提取的 session memory（对话的增量摘要）代替实时生成的 Fork Agent 摘要。

优势：
- **零额外 API 调用**：session memory 由后台 agent 持续维护，压缩时直接使用
- **更低延迟**：无需等待 5-15 秒的 API 响应
- **更细粒度的保留**：可以精确计算保留多少最近消息

### 8.7.2 calculateMessagesToKeepIndex 算法详解

这是 Session Memory 压缩的核心算法（`sessionMemoryCompact.ts:262-327`），决定压缩后保留多少消息：

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // 从 lastSummarizedIndex + 1 开始（session memory 已覆盖之前的内容）
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // 计算当前保留范围的 token 和文本消息数
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // 已达上限 → 不扩展
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 已满足两个最低要求 → 不扩展
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 向前扩展，但不越过最后一个 compact boundary
  const idx = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  const floor = idx === -1 ? 0 : idx + 1

  for (let i = startIndex - 1; i >= floor; i--) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
    startIndex = i
    if (totalTokens >= config.maxTokens) break
    if (totalTokens >= config.minTokens && 
        textBlockMessageCount >= config.minTextBlockMessages) break
  }

  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

配置参数（可通过 GrowthBook 远程配置覆盖）：

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // 至少保留 10K token
  minTextBlockMessages: 5,     // 至少保留 5 条有文本的消息
  maxTokens: 40_000,           // 最多保留 40K token
}
```

算法的双约束设计（`minTokens` AND `minTextBlockMessages`）确保了：
- 不会因为少数几条超大消息就停止扩展（满足 token 但消息太少）
- 不会保留过多小消息但实际 token 不足

**Floor 机制**：向前扩展时不能越过最后一个 compact boundary（`floor = lastBoundaryIndex + 1`）。注释解释了原因：

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

磁盘存储层的消息链在 compact boundary 处有不连续性，越过它会导致加载器的反向遍历跳过保留的消息。

### 8.7.3 adjustIndexToPreserveAPIInvariants 的 bug 修复

这个函数（`sessionMemoryCompact.ts:172-260`）是整个压缩系统中最精妙的一段代码，它解决了两个 API 不变量问题：

**Bug 场景 1：孤儿 tool_result**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

如果 startIndex = N+2:
  旧代码只检查 message N+2 的 tool_results → 找不到 → 返回 N+2
  normalizeMessagesForAPI 按 message.id 合并后:
    msg[1]: assistant with [tool_use: VALID_ID]  (ORPHAN tool_use 被排除!)
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → API 错误：orphan tool_result 引用了不存在的 tool_use
```

**Bug 场景 2：丢失 thinking 块**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

如果 startIndex = N+1:
  thinking 块在 N 被排除
  normalizeMessagesForAPI 无法合并（没有同 ID 的消息可合并）
  → thinking 块永久丢失
```

修复代码执行两步调整：

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Step 1: 处理 tool_use/tool_result 配对
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... 收集保留范围内已有的 tool_use IDs
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // 向前搜索缺失的 tool_use
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 删除已找到的 ID
      }
    }
  }

  // Step 2: 处理共享 message.id 的 thinking 块
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id)
      messageIdsInKeptRange.add(messages[i]!.message.id)
  }
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id &&
        messageIdsInKeptRange.has(messages[i]!.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

这段代码的关键洞察是：Claude API 的流式响应会将同一个 API 回复拆分成多条 assistant 消息（共享 `message.id`，但 UUID 不同），其中 thinking 块和 tool_use 块是分开的。`normalizeMessagesForAPI` 会按 `message.id` 合并这些消息——如果压缩切断了同 ID 的消息组，合并后会出现不一致。

### 8.7.4 trySessionMemoryCompaction 完整流程

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. 门禁检查
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. 初始化远程配置（仅首次）
  await initSessionMemoryCompactConfig()

  // 3. 等待正在进行的 session memory 提取完成
  await waitForSessionMemoryExtraction()

  // 4. 获取 session memory 内容
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. 确定边界
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID 不存在 → 回退
  } else {
    // Resumed session: 没有边界 → 从末尾开始
    lastSummarizedIndex = messages.length - 1
  }

  // 6. 计算保留范围
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // 过滤旧 boundary

  // 7. 运行 session start hooks
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. 构建结果
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. 阈值检查（仅 autocompact）
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // 压缩后仍超阈值 → 回退到传统压缩
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 配置参数（GrowthBook 远程配置）

Session Memory 压缩的所有关键参数都可通过 GrowthBook 远程配置覆盖：

```typescript
// sessionMemoryCompact.ts:91-109 — 远程配置初始化
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // 防御性编码：只使用正数值，忽略 0 值
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

功能门禁由两个独立的 feature flag 控制：

```typescript
// sessionMemoryCompact.ts:333-349
export function shouldUseSessionMemoryCompaction(): boolean {
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) return true
  if (isEnvTruthy(process.env.DISABLE_CLAUDE_CODE_SM_COMPACT)) return false
  
  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory', false)
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact', false)
  return sessionMemoryFlag && smCompactFlag
}
```

## 8.8 上下文折叠与工具结果存储

### 8.8.1 collapseReadSearch 机制

`utils/collapseReadSearch.ts`（1,109 行）实现了 UI 层的消息折叠——将连续的搜索/读取操作折叠为单行摘要（如 "Read 5 files, searched 3 patterns"）。

核心分类逻辑（`getToolSearchOrReadInfo`，`collapseReadSearch.ts:142-237`）将工具调用分为：

| 类别 | 可折叠 | 折叠行为 |
|------|-------|---------|
| Read (file_path) | 是 | "Read N files" |
| Search (Grep/Glob) | 是 | "Searched N patterns" |
| Shell (Bash) | 全屏模式下是 | "Ran N bash commands" |
| REPL | 是（静默吸收） | 内部工具独立计数 |
| Memory Write | 是 | 特殊标记 |
| ToolSearch | 是（静默吸收） | 不增加计数 |
| Edit/Write (非 memory) | 否 | 打断折叠组 |

"静默吸收"（`isAbsorbedSilently`）是一个精妙的设计：REPL 和 ToolSearch 不增加计数器但也不打断当前折叠组。这意味着 `[Read, ToolSearch, Read]` 折叠为 "Read 2 files"，而不是被 ToolSearch 切成两组。

折叠是**仅 UI 层**的优化——它不改变发送给 API 的消息内容，只影响终端显示。

### 8.8.2 toolResultStorage 的磁盘存储策略

`utils/toolResultStorage.ts`（1,040 行）是上下文管理的"第零层"——在工具结果进入对话历史之前就处理超大结果。

**持久化阈值解析**：

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Read 工具特殊：Infinity → 不持久化（Read 自身有 maxTokens 限制）
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // GrowthBook 覆盖（tengu_satin_quoll）
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // 默认：min(工具声明值, 全局 50K 默认值)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**去重优化**：`tool_use_id` 是唯一的，使用 `flag: 'wx'`（exclusive write）避免重复写入：

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: 已在之前的轮次持久化，跳过
}
```

**空结果处理**：

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

这个修复解决了一个模型行为 bug：空的 tool_result 会导致某些模型将 `\n\nHuman:` 模式匹配为对话结束。

**Per-Message Aggregate Budget**（`enforceToolResultBudget`，`toolResultStorage.ts:769-908`）：

这是 `toolResultStorage.ts` 中最复杂的功能——按 API 级别的 user 消息（`normalizeMessagesForAPI` 合并后的）强制执行总工具结果大小预算。

设计要点：
- **状态冻结**（`ContentReplacementState`）：一旦某个 tool_result 被"看见"（sent to model），它的决定（替换/不替换）就被冻结，永远不会改变——这保证了 prompt cache 的稳定性
- **三分区**策略：`mustReapply`（之前替换过 → 重新应用缓存的替换内容）、`frozen`（之前看过但没替换 → 不动）、`fresh`（新的 → 可能被替换）
- **最大优先**：当需要替换时，选择最大的 fresh 结果优先替换

## 8.9 5 层错误恢复中的压缩角色

### 8.9.1 完整 5 层错误恢复机制

压缩系统在 Claude Code 的错误恢复机制中扮演多个角色：

| 层级 | 触发条件 | 压缩行为 | 出处 |
|------|---------|---------|------|
| L1 | API 返回 prompt_too_long (413) | Reactive Compact：截断 + 重新摘要 | `compactMessages.ts` |
| L2 | 压缩 API 自身返回 413 | PTL Retry：截断最旧消息组 × 3 次 | `compact.ts:truncateHeadForPTLRetry` |
| L3 | 压缩后仍超阈值 | Re-compaction：自动再次压缩 | `autoCompact.ts:recompactionInfo` |
| L4 | 连续 3 次压缩失败 | Circuit Breaker：停止尝试 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent 无文本输出 | Streaming Fallback：直接流式 API 调用 | `compact.ts:streamCompactSummary` |

### 8.9.2 响应式压缩 vs 主动压缩

两种策略的取舍：

**主动压缩**（Auto-Compact，当前默认）：
- 在 token 达到阈值时主动触发
- 优点：用户体验更平滑，不会遇到 413 错误
- 缺点：可能过早压缩，浪费可用上下文

**响应式压缩**（Reactive Compact，`tengu_cobalt_raccoon` 实验）：
- 等 API 报 prompt_too_long 才触发
- 优点：最大化上下文利用率
- 缺点：用户体验有明显中断，需要等待重试

代码中可以看到两者的互斥关系：

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // 响应式模式下不主动压缩
  }
}
```

## 8.10 消息分组与 Token 估算

### 8.10.1 groupMessagesByApiRound 算法

`grouping.ts`（63 行）将消息按 API 轮次分组——每个组对应一次完整的 API 往返：

```typescript
// grouping.ts:28-62
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (msg.type === 'assistant' && 
        msg.message.id !== lastAssistantId && 
        current.length > 0) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }
  if (current.length > 0) groups.push(current)
  return groups
}
```

分组边界的唯一判断依据是 `message.id` 的变化——同一个 API 响应的多个流式块共享相同的 `message.id`，所以它们自然归入同一组。

这个设计替代了之前基于"人类轮次"的分组（只在真实用户消息处分组），后者无法处理 SDK/CCR/eval 场景下的长时间单轮 agent 会话。

### 8.10.2 roughTokenCountEstimation 与保守填充

token 估算采用两级保守策略：

**第一级**：基础估算

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

默认 4 bytes/token，JSON 文件用 2 bytes/token（因为 JSON 有大量单字符 token 如 `{`, `}`, `:`, `,`）。

**第二级**：消息级填充

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

组合效果：对于普通文本，有效估算是 `text.length / 4 * 4/3 = text.length / 3`。

### 8.10.3 精确 vs 估算的混合策略

系统在不同场景使用不同精度：

| 场景 | 精度 | 来源 | 延迟 |
|------|------|------|------|
| shouldAutoCompact | 混合：优先 API 返回的精确值 | `tokenCountWithEstimation` | 0（已缓存） |
| estimateMessageTokens | 粗估 (`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | 粗估 | `estimateMessageTokens` | 0 |
| Compact 后 token 统计 | 精确 | `tokenCountFromLastAPIResponse` | 0（API 已返回） |

`tokenCountWithEstimation` 的混合策略是：优先使用最近一次 API 返回的 `usage.input_tokens`（精确值），如果不可用（如首次请求前）则回退到估算。

## 8.11 设计决策分析

### 8.11.1 渐进式降级哲学

Claude Code 的上下文管理采用**不跳级**的渐进式降级：每一层都尝试用最小代价解决问题，只有当前层失败才升级到下一层。这避免了常见的"过度反应"问题——比如仅因为一个大文件的 Read 结果就触发全量压缩。

对比行业实践：
- **ChatGPT**：截断旧消息（简单但粗暴）
- **GitHub Copilot Chat**：固定上下文窗口 + 最近 N 条消息（无压缩）
- **Claude Code**：5 层递进（预防 → 微调 → 摘要 → 紧急恢复）

### 8.11.2 缓存优先设计

Prompt cache 是 Claude Code 的生命线——一次 200K token 的请求，如果 180K 是 cache read（$0.30/M token），成本比全部 cache miss（$3/M token）低 10 倍。几乎所有设计决策都服务于这个经济学约束：

1. **Fork Agent 共享缓存前缀**：压缩 API 调用复用主对话的 cache
2. **不在 Fork 中设 maxOutputTokens**：避免 thinking config 不匹配导致 cache miss
3. **Cached MC 不修改本地消息**：保持 prompt prefix 不变
4. **ContentReplacementState 冻结已见 ID**：确保同一 tool_result 的替换决策在生命周期内不变
5. **sentSkillNames 不重置**：避免重新注入 ~4K token 的 skill_listing
6. **pinnedCacheEdits 在固定位置重发**：确保 cache edit 位置一致

### 8.11.3 安全性保证

系统维护三类不变量：

**配对不可切断**：`adjustIndexToPreserveAPIInvariants` 确保 tool_use 和 tool_result 永远不会被分到不同侧。这不仅是功能正确性的要求（API 会报错），也是语义正确性的要求（模型需要看到它之前调用的工具的结果）。

**递归保护**：`shouldAutoCompact` 中的 `querySource` 检查确保 session_memory agent、compact agent、context collapse agent 不会触发自动压缩——这些 agent 本身就是上下文管理的一部分，递归压缩会导致死锁。

**断路器机制**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 基于真实数据（1,279 sessions 的失败循环）设定，将无限重试改为有限重试 + 熔断。

### 8.11.4 与 API-native Context Management 的对比

`apiMicrocompact.ts` 揭示了 Claude Code 正在探索将部分上下文管理卸载到 API 层的方向：

```typescript
// apiMicrocompact.ts:37-47
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

这些 `context_management.edits` 策略直接在 API 请求中声明，由服务端执行。优势是延迟更低（不需要客户端处理），且与服务端的 token 计数精确对齐。目前仅对内部用户（`USER_TYPE === 'ant'`）开放工具清除策略，外部用户只用 thinking 清除。

## 8.12 可迁移模式

### 8.12.1 多层压缩体系的通用设计模式

Claude Code 的上下文管理提炼出以下可迁移的通用模式：

**模式 1：分级淘汰（Tiered Eviction）**
- 对不同类型的内容应用不同的淘汰策略
- 可再生内容（工具输出）优先淘汰，不可再生内容（用户输入）最后淘汰
- 实现方式：白名单 + 优先级排序

**模式 2：估算-精确混合（Hybrid Estimation）**
- 快速决策用粗估（`text.length / 3`），精确核算用 API 返回值
- 粗估始终偏保守（宁可高估导致提前压缩，不可低估导致 API 报错）

**模式 3：冻结-重放（Freeze-Replay）**
- 一旦内容被模型"看到"，其处理决策被冻结
- 后续轮次对已冻结内容只做"重放"（re-apply cached replacement），不做新决策
- 保证了 prompt prefix 的比特级稳定性 → cache hit

**模式 4：边界感知截断（Boundary-Aware Truncation）**
- 永远不在语义单元中间截断（tool_use/tool_result 对、同 ID 消息组）
- 截断后主动修复（插入合成消息、调整索引）

**模式 5：断路器保护（Circuit Breaker Protection）**
- 对可能无限重试的操作设置失败计数器
- 基于真实运营数据（而非直觉）设定阈值

### 8.12.2 Doramagic 可借鉴之处

Doramagic 的 Soul Extractor 管线中，提取过程可能产生大量中间结果（代码片段、API 文档、社区讨论）。可借鉴的模式：

1. **分层提取缓存**：类似 microcompact 的白名单机制，对中间 API 响应和代码分析结果按可再生性分类，优先淘汰可重新获取的内容
2. **增量摘要**：类似 Session Memory Compact，维护提取知识的增量摘要而非全量历史
3. **冻结决策**：一旦知识块被确认为"有价值"或"无价值"，决策不可逆转——避免在不同提取轮次间反复重新评估

## 8.13 源码索引

| 文件 | 行数 | 核心职责 |
|------|------|---------|
| `services/compact/compact.ts` | ~1,705 | 传统压缩主逻辑：Fork Agent、PTL 重试、附件重注入、部分压缩 |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Session Memory 压缩：calculateMessagesToKeepIndex、adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | 微压缩：时间触发、缓存编辑、token 估算 |
| `services/compact/prompt.ts` | ~374 | 压缩提示词：9 章节模板、NO_TOOLS_PREAMBLE、formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | 自动压缩：阈值计算、shouldAutoCompact 决策链、断路器 |
| `services/compact/apiMicrocompact.ts` | ~153 | API-native 上下文管理：clear_tool_uses、clear_thinking |
| `services/compact/grouping.ts` | ~63 | 消息分组：groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | 压缩后清理：重置缓存、模块状态、分类器 |
| `services/compact/timeBasedMCConfig.ts` | ~43 | 时间触发配置：GrowthBook 远程配置 |
| `services/compact/compactWarningHook.ts` | ~16 | React hook：compact 警告抑制状态订阅 |
| `services/compact/compactWarningState.ts` | ~18 | 状态存储：compact 警告抑制标志 |
| `services/cost-tracker.ts` | ~323 | 成本追踪：token 计费、模型使用统计 |
| `utils/collapseReadSearch.ts` | ~1,109 | 上下文折叠：UI 层消息分组与折叠 |
| `utils/toolResultStorage.ts` | ~1,040 | 工具结果存储：磁盘持久化、per-message budget、ContentReplacementState |
| `services/tokenEstimation.ts` | ~350+ | Token 估算：roughTokenCountEstimation（text.length/4） |
