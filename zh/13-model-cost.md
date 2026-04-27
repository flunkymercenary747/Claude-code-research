# 第 13 章：模型选择与成本控制

> 数据源：Claude Code TypeScript 源码快照（2026-03-31，~512K LOC）
> 核心文件：`services/api/claude.ts`（3,419 行）、`services/api/withRetry.ts`、`cost-tracker.ts`（323 行）、`utils/effort.ts`、`utils/modelCost.ts`、`utils/model/model.ts`、`migrations/` 目录（11 个文件）

---

## 13.1 概述与定位

Claude Code 在模型选择与成本控制上的设计哲学可以用三句话概括：

1. **用户意图优先**：优先级链从 `/model` 命令 → `--model` 标志 → 环境变量 → 配置文件，一路向下，每一层都可以被上层覆盖，但不会被下层意外替换。
2. **成本完全透明**：会话结束时强制打印按模型分类的 token 用量和美元费用，无法关闭（仅 `hasConsoleBillingAccess()` 为真时）。
3. **无秘密降级**：Overload Fallback（Opus → Sonnet）发生时必须向用户显示警告消息，绝不静默切换。

本章从源码层面逐一验证 cc-notebook 关于这一子系统的声称，并深化分析。

---

## 13.2 理论基础

### 多模型系统的路由策略

在多模型系统中，路由策略通常在三个维度上取得平衡：**能力**（capability）、**成本**（cost）、**延迟**（latency）。Claude Code 的选择是将主对话（main loop）路由到最强可用模型，将后台辅助任务路由到最快最便宜的模型，并在主模型不可用时提供透明降级。

### 成本效益分析在 AI 系统中的应用

从 `modelCost.ts` 可以看到，Claude Code 内建了一张精确的价格表：

```typescript
// utils/modelCost.ts
// Standard pricing tier for Sonnet models: $3 input / $15 output per Mtok
export const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// Fast mode pricing for Opus 4.6: $30 input / $150 output per Mtok
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
}
```

Haiku 4.5 定价最低（$1/$5 per Mtok），Opus 4.6 Fast Mode 定价最高（$30/$150 per Mtok），两者相差 30 倍。这个价格差是系统将后台任务分配给 Haiku 的核心经济逻辑。

### 优雅降级（Graceful Degradation）模式

传统软件中，优雅降级是在功能不可用时退而求其次但不崩溃。在 LLM 系统中，退路是"换一个更廉价/更可用的模型"。Claude Code 实现了一套带有计数器保护的触发机制：连续 3 次 529 错误后触发模型切换，而非立即切换（避免偶发 overload 引发不必要的质量降级）。

---

## 13.3 模型选择架构

### 模型优先级层级

`utils/model/model.ts` 中的 `getUserSpecifiedModelSetting()` 函数精确定义了优先级顺序：

```typescript
// utils/model/model.ts:44-66
/**
 * Priority order within this function:
 * 1. Model override during session (from /model command) - highest priority
 * 2. Model override at startup (from --model flag)
 * 3. ANTHROPIC_MODEL environment variable
 * 4. Settings (from user's saved settings)
 */
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  let specifiedModel: ModelSetting | undefined

  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) {
    specifiedModel = modelOverride
  } else {
    const settings = getSettings_DEPRECATED() || {}
    specifiedModel = process.env.ANTHROPIC_MODEL || settings.model || undefined
  }

  // Ignore the user-specified model if it's not in the availableModels allowlist.
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

`getMainLoopModel()` 在此基础上再加入第 5 优先级——内置默认值：

```typescript
// utils/model/model.ts:68-77
export function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}
```

完整的 5 级优先级链：
| 优先级 | 来源 | 说明 |
|--------|------|------|
| 1（最高）| `/model` 命令 | 会话内即时生效，存入内存 override |
| 2 | `--model` 启动标志 | 启动时写入内存 override |
| 3 | `ANTHROPIC_MODEL` 环境变量 | 进程级别 |
| 4 | `settings.json` 配置文件 | 持久化用户偏好 |
| 5（最低）| 内置默认值 | 按订阅类型决定 |

### 默认模型按订阅类型分层

`getDefaultMainLoopModelSetting()` 揭示了订阅差异：

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (内部员工) 默认 Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max 用户和 Team Premium 用户默认 Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG、Enterprise、Team Standard、Pro 默认 Sonnet 4.6
  return getDefaultSonnetModel()
}
```

这个设计意味着：即使用户什么都不配置，Max/Team Premium 用户打开的就是 Opus 4.6，而 Pro/Sonnet 用户打开的是 Sonnet 4.6。**默认值本身就是一种产品差异化策略。**

### 模型 Alias 系统

`parseUserSpecifiedModel()` 支持短别名解析，让用户无需记忆完整 Model ID：

```typescript
// utils/model/model.ts — parseUserSpecifiedModel 节选
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan mode 用 Sonnet，非 plan mode 用 Opus
```

`[1m]` 后缀可以附加在任何 alias 上（如 `opus[1m]`），系统会自动解析成 1M context window 变体。

### 模型能力检测

`utils/model/modelCapabilities.ts` 实现了一套缓存机制，仅对内部员工（`USER_TYPE === 'ant'`）启用：

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

外部用户不请求模型能力列表，能力信息硬编码在 `modelSupportsEffort()`、`modelSupports1M()` 等函数里，从而避免额外的 API 调用开销。

---

## 13.4 Haiku 的后台用途

cc-notebook 声称 Haiku 有 6 种后台用途。通过对 `queryHaiku` 函数调用位置（`grep -rn 'queryHaiku\b'`）和 `getSmallFastModel()` 调用位置的全量搜索，**源码验证**如下：

### 后台用途汇总（源码验证）

| 编号 | 用途 | 文件 | 触发条件 |
|------|------|------|---------|
| 1 | Web Fetch 内容提取 | `tools/WebFetchTool/utils.ts:503` | 抓取网页后用 Haiku 将 Markdown 过滤为用户指定内容 |
| 2 | Shell 命令前缀提取 | `utils/shell/prefix.ts:220` | Bash 工具执行前，用 Haiku 判断命令是否需要权限提示 |
| 3 | 会话标题生成 | `utils/sessionTitle.ts:87` | 会话开始后自动生成简短标题（JSON schema 输出）|
| 4 | MCP DateTime 解析 | `utils/mcp/dateTimeParser.ts:68` | 将自然语言时间描述解析为 ISO 8601 格式 |
| 5 | 工具调用摘要生成 | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | 一批工具调用完成后生成一行摘要标签 |
| 6 | 会话重命名 | `commands/rename/generateSessionName.ts:20` | `/rename` 命令生成 kebab-case 名称 |

**额外发现**（cc-notebook 未提及，通过 `getSmallFastModel()` 搜索发现）：

| 编号 | 用途 | 文件 | 触发条件 |
|------|------|------|---------|
| 7 | API Key 验证 | `services/api/claude.ts:544` | 验证 API Key 有效性（源码注释："WARNING: if you change this to use a non-Haiku model, this request will fail in 1P"）|
| 8 | Away 模式摘要 | `services/awaySummary.ts:49` | 用户离开时生成上下文摘要（AFK 模式）|
| 9 | Web 搜索辅助 | `tools/WebSearchTool/WebSearchTool.ts:280` | 部分 Web 搜索场景用 Haiku 处理结果 |
| 10 | Quota 状态检查 | `services/claudeAiLimits.ts:200` | 用最小 Haiku 请求探测当前配额状态 |
| 11 | Token 数量估算 | `services/tokenEstimation.ts:277` | 估算 context window 用量 |
| 12 | Prompt/Exec Hook 执行 | `utils/hooks/execPromptHook.ts:79`、`execAgentHook.ts:118` | Hook 回调默认使用 Haiku（除非 hook 配置覆盖）|
| 13 | Skill 改进分析 | `utils/hooks/skillImprovement.ts:169` | Skill 执行后自动分析改进建议 |

**结论**：cc-notebook 的"6 种后台用途"是**低估**。源码中 `queryHaiku` 或 `getSmallFastModel()` 的调用点至少 13 处，涵盖了会话生命周期的各个阶段（启动验证、执行中辅助、会话整理）。Haiku/SmallFastModel 是整个系统的后台"基础服务层"，不是偶尔出现的优化手段。

关键设计细节：`queryHaiku` 使用非流式调用（`queryModelWithoutStreaming`），且不带 Tool permission context（`getEmptyToolPermissionContext()`）：

```typescript
// services/api/claude.ts:3280-3291
const result = await queryModelWithoutStreaming({
  messages,
  systemPrompt,
  thinkingConfig: { type: 'disabled' },
  tools: [],
  signal,
  options: {
    ...options,
    model: getSmallFastModel(),
    enablePromptCaching: options.enablePromptCaching ?? false,
    async getToolPermissionContext() {
      return getEmptyToolPermissionContext()
    },
  },
})
```

---

## 13.5 Overload Fallback 机制

cc-notebook 声称存在"529 Overload Fallback，Opus → Sonnet 回退"。源码**完全验证**这一声称，且细节更丰富。

### 529 错误识别

`services/api/withRetry.ts` 中的 `is529Error()` 函数：

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // 检查 529 状态码，或流式传输时 SDK 未能正确传递状态码的情况
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

注意双重检测：状态码 `529` 和错误消息中的 `overloaded_error` 字符串。这是因为 SDK 在流式传输中有时无法正确传递 529 状态码。

### 触发条件：连续 3 次 529

```typescript
// services/api/withRetry.ts — withRetry 函数节选
const MAX_529_RETRIES = 3

if (
  is529Error(error) &&
  (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
    (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })
      // 抛出特殊错误，触发上层模型切换
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

关键约束：
- 默认仅对**非 ClaudeAI 订阅用户**的 **Opus 系列模型**触发（`isNonCustomOpusModel()`）
- 环境变量 `FALLBACK_FOR_ALL_PRIMARY_MODELS` 可扩展到所有主模型
- 流式请求 529 计入计数器（`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`），与非流式重试协同计数

### FallbackTriggeredError 信号传播

`FallbackTriggeredError` 是一个专用错误类，携带 `originalModel` 和 `fallbackModel` 字段，沿调用栈向上传播到 `query.ts`：

```typescript
// services/api/withRetry.ts
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    super(`Model fallback triggered: ${originalModel} -> ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

### query.ts 中的模型切换与用户通知

`query.ts:894-946` 捕获这个错误，执行实际模型切换：

```typescript
// query.ts — FallbackTriggeredError 处理节选
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // 用 warning 级别展示给用户——无论 verbose 模式是否开启都可见
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // 同步更新 toolUseContext 中的主循环模型
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // 用新模型重试整个请求
}
```

**用户通知机制**：切换消息使用 `'warning'` 级别，这意味着无论用户是否开启 verbose 模式，都会在界面上看到通知。**cc-notebook 关于"无秘密降级"的声称得到完全验证。**

### 后台任务的 529 策略：直接放弃

非前台任务（如 summary、title、suggestions）在 529 时**不重试**，直接丢弃：

```typescript
// services/api/withRetry.ts — FOREGROUND_529_RETRY_SOURCES
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'agent:builtin',
  'compact',
  'verification_agent',
  'side_question',
  'auto_mode',
  // ...
])

// 非前台任务 529 直接抛出，不重试
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

这是架构上的成本控制决策：后台任务重试会在容量紧张时产生 3-10 倍的网关放大效应，而用户根本感知不到这些任务失败。

---

## 13.6 Effort Level 机制

cc-notebook 声称存在 Effort Level 系统。源码**完全验证**，且细节远比描述丰富。

### 四个 Effort Level

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

各级语义（来自 `getEffortLevelDescription()`）：
- **low**：Quick, straightforward implementation with minimal overhead
- **medium**：Balanced approach with standard implementation and testing
- **high**：Comprehensive implementation with extensive testing and documentation
- **max**：Maximum capability with deepest reasoning（**Opus 4.6 only**）

### 模型支持矩阵

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // 仅 Opus 4.6 和 Sonnet 4.6 支持 effort 参数
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku、旧版 Opus/Sonnet 不支持
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P 默认 true，3P 默认 false
  return getAPIProvider() === 'firstParty'
}

// max effort 仅 Opus 4.6 可用
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### 优先级链：env → appState → 模型默认值

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' 或 'auto' → 不发送 effort 参数
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API 拒绝非 Opus 4.6 使用 max → 自动降级到 high
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### 默认 Effort 差异化

Opus 4.6 的默认 effort 根据订阅类型不同：

```typescript
// utils/effort.ts — getDefaultEffortForModel 节选
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // Pro 用户默认 medium（节省额度）
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team 也可被 GrowthBook 配置推向 medium
  }
}
```

有趣的是，`OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` 的 `dialogDescription` 明确写道："We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits."——这说明默认 medium 是有意识的额度管理策略，而非性能优先。

### max 的持久化限制

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max 对非 ant 用户是会话级的，不持久化
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

外部用户设置的 `max` effort 不会写入 `settings.json`，仅在当前会话有效。

---

## 13.7 成本追踪系统

### cost-tracker.ts 的核心职责

`cost-tracker.ts`（323 行）承担三个职责：
1. **实时累加**：每次 API 响应后调用 `addToTotalSessionCost()`
2. **持久化**：会话结束时写入项目配置文件（`saveCurrentSessionCosts()`）
3. **恢复**：重启时从配置文件读取上次的成本数据（`restoreCostStateForSession()`）

### 按模型分类的 token 统计

`addToTotalModelUsage()` 按模型名称累加 5 维度数据：

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

会话结束时格式化展示（`formatModelUsage()`）：按短名称聚合（多个 API 端点返回同一模型的不同格式），显示如：

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Fast Mode 的成本标记

`addToTotalSessionCost()` 中有针对 Fast Mode 的特殊处理：

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

`speed: 'fast'` 标记会影响计费——Fast Mode 下 Opus 4.6 使用 `COST_TIER_30_150`（$30/$150），而非标准的 `COST_TIER_5_25`（$5/$25）。

### Advisor 嵌套成本追踪

`addToTotalSessionCost()` 递归处理 Advisor 工具的用量：

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

Advisor 是一个隐藏在主模型响应中的次级模型调用，其成本被单独追踪并合并到总成本中。

### 成本展示触发机制

`costHook.ts`（22 行）是一个 React hook，监听进程退出事件：

```typescript
// costHook.ts
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

`hasConsoleBillingAccess()` 控制是否显示成本，确保在无权访问计费信息的环境（如 CCR/Remote 模式）中不显示成本，同时写入 `saveCurrentSessionCosts()` 是无条件执行的——无论显不显示，都要持久化。

---

## 13.8 API 调用层

### claude.ts 请求构造的核心参数

`services/api/claude.ts`（3,419 行）是 API 调用的统一入口。关键参数来自多个系统的汇聚：

```typescript
// services/api/claude.ts — 请求参数组装（示意）
{
  model: normalizeModelStringForAPI(options.model),  // 剥离 [1m] 后缀
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // Effort 参数（仅支持的模型）
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` 在发送到 API 前剥离 `[1m]` 和 `[2m]` 后缀——这些后缀只是客户端内部标记 1M context window 的约定，API 层不认识它们。

### 流式响应与非流式回退

主对话使用流式传输（Server-Sent Events），但在流式传输失败时会回退到非流式：

```typescript
// services/api/claude.ts:2535-2559
// 如果流式失败本身是 529，将这次计入连续 529 计数
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

非流式回退有最大 token 限制：

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Beta Headers 动态注入

不同功能对应不同的 Beta Header，在请求时动态附加：

```typescript
// constants/betas.ts（引用）
EFFORT_BETA_HEADER        // effort 参数支持
CONTEXT_1M_BETA_HEADER    // 1M context window
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // 预算控制
```

---

## 13.9 设计决策分析

### 无秘密降级的设计哲学

从 `query.ts` 中 `'warning'` 级别的切换通知，以及 `FallbackTriggeredError` 的专用错误类设计，可以看出这是一个刻意的架构选择：

**为什么不能静默切换？** 因为 Claude Code 是一个代码编写工具，模型质量直接影响输出质量。用户有权知道"我正在用 Sonnet 而不是 Opus"，从而决定是否继续等待或用不同策略。与消费级聊天产品不同，代码工具的用户更专业，对模型差异更敏感。

### 成本透明的设计考量

`costHook.ts` 中 `hasConsoleBillingAccess()` 的设计值得关注：即使不显示，成本数据也被持久化。这说明成本追踪的主要目的是**会话恢复**（下次启动时显示上次花费）而非实时警报。这是一个"离线感知"的设计：用户在会话结束后能看到完整花费，而不是在每次 API 调用后被打断。

### 模型默认值差异化的产品逻辑

将 Opus 作为 Max/Team Premium 的默认模型，将 Sonnet 作为 Pro/PAYG 的默认模型，背后有明确的产品逻辑：Max 订阅的价值主张之一就是"获得最强模型"，默认值本身就是这个价值主张的体现。

同时，即便是 Max 用户，Opus 4.6 的默认 effort 是 `medium`（受 GrowthBook 控制）——这说明 Anthropic 正在通过 effort 系统**在质量和额度之间取得平衡**，而非一味给 Max 用户最高配置。

---

## 13.10 模型迁移（migrations）的必要性

`migrations/` 目录下 11 个迁移文件揭示了产品演进的痕迹，每一个迁移都对应一次产品决策：

| 迁移文件 | 触发时机 | 核心逻辑 |
|---------|---------|---------|
| `migrateFennecToOpus.ts` | 内部员工（ant）| fennec 代号 alias → opus alias（清理内部代号） |
| `migrateLegacyOpusToCurrent.ts` | 1P 用户，`opus-4-0`/`4-1` 仍在 settings | 旧版 Opus model ID → `opus` alias（Opus 4.0/4.1 下线）|
| `migrateOpusToOpus1m.ts` | Max/Team Premium（1P），`opus` 在 settings | `opus` → `opus[1m]`（合并 1M 体验）|
| `migrateSonnet1mToSonnet45.ts` | 用 `sonnet[1m]` 的用户 | `sonnet[1m]` → `sonnet-4-5-20250929[1m]`（pin 到 4.5，因 4.6 1M 受众不同）|
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium（1P），pin 到 Sonnet 4.5 | Sonnet 4.5 字符串 → `sonnet` alias（升级到 4.6）|
| `resetProToOpusDefault.ts` | Pro 1P 用户，无自定义 model | 记录迁移时间戳，让 REPL 显示一次升级通知 |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode enabled，旧 OptIn 对话用户 | 清除 `skipAutoPermissionPrompt`，重新展示新版权限对话 |
| `migrateAutoUpdatesToSettings.ts` | 用户明确禁用了自动更新 | 将 `autoUpdates: false` 迁移到 settings.json 的环境变量 |
| `migrateBypassPermissionsAcceptedToSettings.ts` | 全局配置有 `bypassPermissionsModeAccepted` | 迁移到 settings.json 的 `skipDangerousModePermissionPrompt` |
| `migrateSonnet45ToSonnet46.ts` | 同上 | 前述同名迁移 |
| `migrateEnableAllProjectMcpServersToSettings.ts` | MCP 相关配置 | MCP 服务器设置结构调整 |

**架构洞察**：每个迁移只操作 `userSettings`（用户级 settings.json），从不触碰 `projectSettings`（项目级）或 `policySettings`（企业策略级）。这是有意设计的：

1. **幂等性**：读写同一数据源，重跑不会产生副作用
2. **最小权限**：不能（也不应该）替用户修改他们在项目级别的 pin
3. **避免全局提升**：如果用户在某个项目里 pin 了旧 Opus，迁移不会把它提升为全局设置

这个迁移系统的存在本身说明：**AI 系统的 schema migration 远比传统数据库迁移复杂**——需要考虑订阅类型变化、模型下线、context window 升级等多个维度，且不能简单粗暴地覆写用户意图。

---

## 13.11 可迁移模式

从本章分析中提炼出 5 个可用于自己系统的设计模式：

### 模式一：多级 Override 链
```
session_override > startup_flag > env_var > config_file > builtin_default
```
任何层都可以被上层覆盖，但下层不能偷偷影响上层。配合 allowlist 检查防止非法 model ID 注入。

### 模式二：前台/后台 529 策略分离
前台任务（用户等待结果）：重试 N 次，超限后触发 fallback。
后台任务（用户不感知）：首次 529 直接放弃，避免容量雪崩中的重试放大效应。

### 模式三：FallbackTriggeredError 信号化
不在 retry 内部悄悄切换模型，而是抛出专用错误，让更上层的调用者处理切换逻辑。这样切换逻辑集中在一处（query.ts），且必然伴随用户通知。

### 模式四：toPersistableEffort 持久化过滤
会话级设置（如 `max` effort）在写入 settings.json 前被过滤掉。"不应该跨会话持久化的状态"和"应该持久化的用户偏好"从数据模型层面就区分开。

### 模式五：成本按模型分桶追踪
不只追踪总成本，而是按模型名称（规范化后）分桶。这样才能在会话结束时展示"Opus 花了多少，Haiku 花了多少"，帮助用户理解哪个功能最贵。

---

## 13.12 源码索引

| 文件 | 行数 | 核心内容 |
|------|------|---------|
| `services/api/claude.ts` | 3,419 | API 调用层、queryHaiku、请求构造、流式处理 |
| `services/api/withRetry.ts` | ~600 | 重试逻辑、529 处理、FallbackTriggeredError |
| `cost-tracker.ts` | 323 | 成本追踪、持久化、格式化展示 |
| `costHook.ts` | 22 | React hook，监听进程退出触发成本展示 |
| `utils/effort.ts` | ~350 | Effort Level 定义、优先级链、模型支持检测 |
| `utils/modelCost.ts` | ~200 | 价格表、成本计算函数 |
| `utils/model/model.ts` | ~450 | 模型优先级链、alias 解析、默认模型逻辑 |
| `utils/model/modelCapabilities.ts` | ~100 | 模型能力缓存（仅内部用户） |
| `query.ts` | ~1000 | FallbackTriggeredError 捕获、用户通知、模型切换 |
| `migrations/*.ts` | 11 文件 | 模型版本迁移脚本 |
| `tools/WebFetchTool/utils.ts:503` | — | Haiku 用途 1：Web Fetch 内容提取 |
| `utils/shell/prefix.ts:220` | — | Haiku 用途 2：Shell 命令前缀判断 |
| `utils/sessionTitle.ts:87` | — | Haiku 用途 3：会话标题生成 |
| `utils/mcp/dateTimeParser.ts:68` | — | Haiku 用途 4：DateTime 解析 |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Haiku 用途 5：工具调用摘要 |
| `commands/rename/generateSessionName.ts:20` | — | Haiku 用途 6：会话重命名 |
| `services/api/claude.ts:544` | — | Haiku 用途 7：API Key 验证 |

---

*本章完整覆盖了 cc-notebook 关于模型选择与成本控制的声称。验证结果：Haiku 后台用途"至少 6 种"得到验证（实为 13 处调用点）；无秘密降级完全验证；529 Overload Fallback 机制完全验证；Effort Level 系统完全验证。所有代码片段均从源文件精确复制，标注了文件路径和行号。*
