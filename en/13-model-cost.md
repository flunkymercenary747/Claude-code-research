# Chapter 13: Model Selection and Cost Control

> Data source: Claude Code TypeScript source snapshot (2026-03-31, ~512K LOC)
> Core files: `services/api/claude.ts` (3,419 lines), `services/api/withRetry.ts`, `cost-tracker.ts` (323 lines), `utils/effort.ts`, `utils/modelCost.ts`, `utils/model/model.ts`, `migrations/` directory (11 files)

---

## 13.1 Overview and Purpose

Claude Code's design philosophy for model selection and cost control can be summarized in three sentences:

1. **User intent first**: The priority chain runs from `/model` command → `--model` flag → environment variable → config file, cascading downward; each layer can be overridden by the layer above but will not be accidentally replaced by the layer below.
2. **Full cost transparency**: At session end, token usage and USD cost broken down by model are forcibly printed; this cannot be disabled (only when `hasConsoleBillingAccess()` is true).
3. **No silent downgrade**: When Overload Fallback (Opus → Sonnet) occurs, a warning message must be shown to the user; silent switching is never permitted.

This chapter validates claims about this subsystem from a source code perspective and deepens the analysis.

---

## 13.2 Theoretical Foundation

### Routing Strategies in Multi-Model Systems

In multi-model systems, routing strategies typically balance three dimensions: **capability**, **cost**, and **latency**. Claude Code's choice is to route the main conversation (main loop) to the strongest available model, route background auxiliary tasks to the fastest and cheapest model, and provide transparent downgrade when the primary model is unavailable.

### Cost-Benefit Analysis in AI Systems

From `modelCost.ts`, Claude Code has a built-in precise price table:

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

Haiku 4.5 has the lowest pricing ($1/$5 per Mtok), Opus 4.6 Fast Mode has the highest ($30/$150 per Mtok) — a 30x difference. This price differential is the core economic logic behind the system assigning background tasks to Haiku.

### Graceful Degradation Pattern

In traditional software, graceful degradation means falling back without crashing when functionality is unavailable. In LLM systems, the fallback is "switching to a cheaper/more available model." Claude Code implements a counter-protected trigger mechanism: switching occurs after 3 consecutive 529 errors, not immediately (to avoid unnecessary quality downgrade from occasional overload).

---

## 13.3 Model Selection Architecture

### Model Priority Hierarchy

The `getUserSpecifiedModelSetting()` function in `utils/model/model.ts` precisely defines the priority order:

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

`getMainLoopModel()` adds a 5th priority on top of this — the built-in default:

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

Complete 5-level priority chain:
| Priority | Source | Description |
|----------|--------|-------------|
| 1 (highest) | `/model` command | Takes effect immediately within session, stored in memory override |
| 2 | `--model` startup flag | Written to memory override at startup |
| 3 | `ANTHROPIC_MODEL` environment variable | Process level |
| 4 | `settings.json` config file | Persistent user preferences |
| 5 (lowest) | Built-in default | Determined by subscription type |

### Default Models Tiered by Subscription Type

`getDefaultMainLoopModelSetting()` reveals subscription differences:

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (internal employees) default to Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max users and Team Premium users default to Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG, Enterprise, Team Standard, Pro default to Sonnet 4.6
  return getDefaultSonnetModel()
}
```

This design means: even without any configuration, Max/Team Premium users open with Opus 4.6, while Pro/Sonnet users open with Sonnet 4.6. **The default itself is a product differentiation strategy.**

### Model Alias System

`parseUserSpecifiedModel()` supports short alias resolution, so users don't need to memorize full Model IDs:

```typescript
// utils/model/model.ts — parseUserSpecifiedModel excerpt
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan mode uses Sonnet, non-plan uses Opus
```

The `[1m]` suffix can be appended to any alias (e.g., `opus[1m]`); the system automatically resolves it to the 1M context window variant.

### Model Capability Detection

`utils/model/modelCapabilities.ts` implements a caching mechanism, enabled only for internal employees (`USER_TYPE === 'ant'`):

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

External users do not request the model capabilities list; capability information is hard-coded in functions like `modelSupportsEffort()` and `modelSupports1M()`, avoiding additional API call overhead.

---

## 13.4 Haiku's Background Use Cases

The source code validates Haiku use cases through a full search for `queryHaiku` function call sites and `getSmallFastModel()` call sites, as follows:

### Background Use Case Summary (Source Verified)

| # | Use Case | File | Trigger Condition |
|---|----------|------|------------------|
| 1 | Web Fetch content extraction | `tools/WebFetchTool/utils.ts:503` | After fetching a webpage, Haiku filters Markdown to user-specified content |
| 2 | Shell command prefix extraction | `utils/shell/prefix.ts:220` | Before Bash tool execution, Haiku judges whether command needs permission prompt |
| 3 | Session title generation | `utils/sessionTitle.ts:87` | Automatically generates a short title after session starts (JSON schema output) |
| 4 | MCP DateTime parsing | `utils/mcp/dateTimeParser.ts:68` | Parses natural language time descriptions to ISO 8601 format |
| 5 | Tool call summary generation | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | Generates a one-line summary label after a batch of tool calls completes |
| 6 | Session renaming | `commands/rename/generateSessionName.ts:20` | `/rename` command generates kebab-case names |

**Additional discoveries** (not mentioned elsewhere, found via `getSmallFastModel()` search):

| # | Use Case | File | Trigger Condition |
|---|----------|------|------------------|
| 7 | API Key validation | `services/api/claude.ts:544` | Validates API Key validity (source comment: "WARNING: if you change this to use a non-Haiku model, this request will fail in 1P") |
| 8 | Away mode summary | `services/awaySummary.ts:49` | Generates context summary when user is away (AFK mode) |
| 9 | Web search assist | `tools/WebSearchTool/WebSearchTool.ts:280` | Some web search scenarios use Haiku to process results |
| 10 | Quota status check | `services/claudeAiLimits.ts:200` | Uses minimal Haiku request to probe current quota status |
| 11 | Token count estimation | `services/tokenEstimation.ts:277` | Estimates context window usage |
| 12 | Prompt/Exec Hook execution | `utils/hooks/execPromptHook.ts:79`, `execAgentHook.ts:118` | Hook callbacks default to Haiku (unless overridden by hook config) |
| 13 | Skill improvement analysis | `utils/hooks/skillImprovement.ts:169` | Automatically analyzes improvement suggestions after Skill execution |

**Conclusion**: Haiku/SmallFastModel is the background "base service layer" of the entire system, not an occasional optimization. There are at least 13 call sites for `queryHaiku` or `getSmallFastModel()` in the source code, spanning all phases of the session lifecycle (startup validation, in-execution assistance, session cleanup).

Key design detail: `queryHaiku` uses non-streaming calls (`queryModelWithoutStreaming`) without Tool permission context (`getEmptyToolPermissionContext()`):

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

## 13.5 Overload Fallback Mechanism

The "529 Overload Fallback, Opus → Sonnet rollback" claim is completely verified in source code, with richer details.

### 529 Error Identification

The `is529Error()` function in `services/api/withRetry.ts`:

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // check 529 status code, or cases where SDK fails to properly pass status code during streaming
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

Note the dual detection: status code `529` and the string `overloaded_error` in the error message. This is because the SDK sometimes fails to properly pass the 529 status code during streaming.

### Trigger Condition: 3 Consecutive 529s

```typescript
// services/api/withRetry.ts — withRetry function excerpt
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
      // throw special error to trigger upper-layer model switch
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

Key constraints:
- By default, only triggers for **non-ClaudeAI subscriber** users on **Opus series models** (`isNonCustomOpusModel()`)
- Environment variable `FALLBACK_FOR_ALL_PRIMARY_MODELS` can extend to all primary models
- Streaming request 529s count toward the counter (`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`), coordinating counting with non-streaming retries

### FallbackTriggeredError Signal Propagation

`FallbackTriggeredError` is a dedicated error class carrying `originalModel` and `fallbackModel` fields, propagating up the call stack to `query.ts`:

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

### Model Switching and User Notification in query.ts

`query.ts:894-946` catches this error and executes the actual model switch:

```typescript
// query.ts — FallbackTriggeredError handling excerpt
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // display to user at warning level — visible regardless of verbose mode
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // synchronously update main loop model in toolUseContext
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // retry entire request with new model
}
```

**User notification mechanism**: The switch message uses `'warning'` level, meaning the notification will be visible in the UI regardless of whether the user has verbose mode enabled. **The "no silent downgrade" claim is completely verified.**

### 529 Strategy for Background Tasks: Immediate Abandonment

Non-foreground tasks (such as summary, title, suggestions) **do not retry** on 529 — they are dropped directly:

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

// non-foreground task 529 throws directly, no retry
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

This is an architectural cost control decision: background task retries would produce 3-10x gateway amplification effects during capacity constraints, while users have no awareness of these task failures.

---

## 13.6 Effort Level Mechanism

The Effort Level system is completely verified in source code, with far richer detail than described.

### Four Effort Levels

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

Level semantics (from `getEffortLevelDescription()`):
- **low**: Quick, straightforward implementation with minimal overhead
- **medium**: Balanced approach with standard implementation and testing
- **high**: Comprehensive implementation with extensive testing and documentation
- **max**: Maximum capability with deepest reasoning (**Opus 4.6 only**)

### Model Support Matrix

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // only Opus 4.6 and Sonnet 4.6 support the effort parameter
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku, older Opus/Sonnet do not support it
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P defaults true, 3P defaults false
  return getAPIProvider() === 'firstParty'
}

// max effort available only for Opus 4.6
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### Priority Chain: env → appState → model default

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' or 'auto' → do not send effort parameter
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API rejects max effort for non-Opus 4.6 → auto-downgrade to high
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### Differentiated Default Effort

Opus 4.6's default effort varies by subscription type:

```typescript
// utils/effort.ts — getDefaultEffortForModel excerpt
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // Pro users default to medium (conserve quota)
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team can also be pushed to medium by GrowthBook config
  }
}
```

Notably, `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT`'s `dialogDescription` explicitly states: "We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits." — This shows that defaulting to medium is a deliberate quota management strategy, not a performance-first choice.

### Persistence Restriction for max

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max is session-level for non-ant users; not persisted
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

`max` effort set by external users is not written to `settings.json` and is only valid for the current session.

---

## 13.7 Cost Tracking System

### Core Responsibilities of cost-tracker.ts

`cost-tracker.ts` (323 lines) serves three responsibilities:
1. **Real-time accumulation**: Calls `addToTotalSessionCost()` after each API response
2. **Persistence**: Writes to the project config file at session end (`saveCurrentSessionCosts()`)
3. **Restoration**: Reads last session cost data from the config file on restart (`restoreCostStateForSession()`)

### Token Statistics Broken Down by Model

`addToTotalModelUsage()` accumulates 5-dimensional data by model name:

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

Formatted display at session end (`formatModelUsage()`): aggregated by short name (multiple API endpoints return the same model in different formats), displayed as:

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Cost Marking for Fast Mode

`addToTotalSessionCost()` has special handling for Fast Mode:

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

The `speed: 'fast'` marker affects billing — Opus 4.6 in Fast Mode uses `COST_TIER_30_150` ($30/$150), rather than the standard `COST_TIER_5_25` ($5/$25).

### Advisor Nested Cost Tracking

`addToTotalSessionCost()` recursively handles Advisor tool usage:

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

The Advisor is a secondary model call hidden within the main model's response; its cost is separately tracked and merged into the total cost.

### Cost Display Trigger Mechanism

`costHook.ts` (22 lines) is a React hook that listens for process exit events:

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

`hasConsoleBillingAccess()` controls whether costs are displayed, ensuring cost information is not shown in environments without billing access (such as CCR/Remote mode). Meanwhile, writing to `saveCurrentSessionCosts()` is unconditional — regardless of display, it is always persisted.

---

## 13.8 API Call Layer

### Core Parameters of claude.ts Request Construction

`services/api/claude.ts` (3,419 lines) is the unified entry point for API calls. Key parameters converge from multiple systems:

```typescript
// services/api/claude.ts — request parameter assembly (illustrative)
{
  model: normalizeModelStringForAPI(options.model),  // strips [1m] suffix
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // Effort parameter (only for supported models)
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` strips `[1m]` and `[2m]` suffixes before sending to the API — these suffixes are only a client-internal convention for marking 1M context window models; the API layer does not recognize them.

### Streaming Response and Non-Streaming Fallback

The main conversation uses streaming (Server-Sent Events), but falls back to non-streaming when streaming fails:

```typescript
// services/api/claude.ts:2535-2559
// If the streaming failure itself is 529, count this towards consecutive 529s
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

Non-streaming fallback has a maximum token limit:

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Dynamic Beta Header Injection

Different features correspond to different Beta Headers, dynamically attached at request time:

```typescript
// constants/betas.ts (reference)
EFFORT_BETA_HEADER        // effort parameter support
CONTEXT_1M_BETA_HEADER    // 1M context window
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // budget control
```

---

## 13.9 Design Decision Analysis

### The Design Philosophy of No Silent Downgrade

From the `'warning'` level switch notification in `query.ts` and the dedicated error class design of `FallbackTriggeredError`, this is clearly a deliberate architectural choice:

**Why can't there be silent switching?** Because Claude Code is a code-writing tool and model quality directly affects output quality. Users have the right to know "I'm using Sonnet, not Opus," allowing them to decide whether to keep waiting or use a different strategy. Unlike consumer chat products, code tool users are more professional and more sensitive to model differences.

### Design Considerations for Cost Transparency

The design of `hasConsoleBillingAccess()` in `costHook.ts` is worth noting: even when not displayed, cost data is persisted. This suggests that cost tracking's primary purpose is **session recovery** (showing the cost of the last session on next startup) rather than real-time alerts. This is an "offline-aware" design: users see the complete expenditure after the session ends, rather than being interrupted after every API call.

### Product Logic Behind Model Default Differentiation

Making Opus the default for Max/Team Premium and Sonnet the default for Pro/PAYG has clear product logic: one of Max subscription's value propositions is "access to the strongest model," and the default itself embodies that value proposition.

At the same time, even for Max users, Opus 4.6's default effort is `medium` (controlled by GrowthBook) — indicating that Anthropic is **balancing quality and quota** through the effort system, rather than simply giving Max users the highest configuration.

---

## 13.10 The Necessity of Model Migrations

The 11 migration files in the `migrations/` directory reveal the traces of product evolution; each migration corresponds to a product decision:

| Migration File | Trigger Timing | Core Logic |
|---------------|---------------|-----------|
| `migrateFennecToOpus.ts` | Internal employees (ant) | fennec codename alias → opus alias (cleanup of internal codenames) |
| `migrateLegacyOpusToCurrent.ts` | 1P users with `opus-4-0`/`4-1` still in settings | Old Opus model ID → `opus` alias (Opus 4.0/4.1 sunset) |
| `migrateOpusToOpus1m.ts` | Max/Team Premium (1P), `opus` in settings | `opus` → `opus[1m]` (merging 1M experience) |
| `migrateSonnet1mToSonnet45.ts` | Users using `sonnet[1m]` | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` (pin to 4.5, as 4.6 1M targets different audience) |
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium (1P), pinned to Sonnet 4.5 | Sonnet 4.5 string → `sonnet` alias (upgrade to 4.6) |
| `resetProToOpusDefault.ts` | Pro 1P users, no custom model | Records migration timestamp, REPL shows upgrade notification once |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode enabled, old OptIn dialog users | Clears `skipAutoPermissionPrompt`, re-shows new permission dialog |
| `migrateAutoUpdatesToSettings.ts` | User explicitly disabled auto-updates | Migrates `autoUpdates: false` to settings.json environment variable |
| `migrateBypassPermissionsAcceptedToSettings.ts` | Global config has `bypassPermissionsModeAccepted` | Migrates to settings.json's `skipDangerousModePermissionPrompt` |
| `migrateSonnet45ToSonnet46.ts` | Same as above | Aforementioned same-name migration |
| `migrateEnableAllProjectMcpServersToSettings.ts` | MCP-related config | MCP server settings structure adjustment |

**Architectural insight**: Each migration only operates on `userSettings` (user-level settings.json), never touching `projectSettings` (project-level) or `policySettings` (enterprise policy level). This is by design:

1. **Idempotency**: Reading and writing the same data source; re-running produces no side effects
2. **Least privilege**: Cannot (and should not) modify project-level pins on behalf of users
3. **Avoid global escalation**: If a user pinned an old Opus in a specific project, the migration won't elevate it to a global setting

The existence of this migration system itself demonstrates: **schema migration in AI systems is far more complex than traditional database migrations** — needing to account for subscription type changes, model sunsets, context window upgrades, and multiple other dimensions, without crudely overwriting user intent.

---

## 13.11 Transferable Patterns

Five design patterns distilled from this chapter's analysis that can be used in other systems:

### Pattern 1: Multi-Level Override Chain
```
session_override > startup_flag > env_var > config_file > builtin_default
```
Any layer can be overridden by the layer above, but lower layers cannot secretly influence upper layers. Combined with allowlist checking to prevent illegal model ID injection.

### Pattern 2: Foreground/Background 529 Strategy Separation
Foreground tasks (user waiting for results): retry N times, trigger fallback when exceeded.
Background tasks (user unaware): abandon on first 529, avoiding retry amplification effect during capacity crunch.

### Pattern 3: FallbackTriggeredError Signaling
Don't silently switch models inside the retry logic; instead throw a dedicated error and let the upper-level caller handle the switch logic. This way, switch logic is centralized in one place (query.ts) and is necessarily accompanied by user notification.

### Pattern 4: toPersistableEffort Persistence Filtering
Session-level settings (like `max` effort) are filtered before writing to settings.json. States that "should not persist across sessions" and "user preferences that should persist" are differentiated at the data model level.

### Pattern 5: Cost Tracking Bucketed by Model
Track not just total cost, but bucket by model name (after normalization). This enables displaying "how much Opus cost, how much Haiku cost" at session end, helping users understand which features are most expensive.

---

## 13.12 Source Index

| File | Lines | Core Content |
|------|-------|-------------|
| `services/api/claude.ts` | 3,419 | API call layer, queryHaiku, request construction, streaming handling |
| `services/api/withRetry.ts` | ~600 | Retry logic, 529 handling, FallbackTriggeredError |
| `cost-tracker.ts` | 323 | Cost tracking, persistence, formatted display |
| `costHook.ts` | 22 | React hook, listens for process exit to trigger cost display |
| `utils/effort.ts` | ~350 | Effort Level definitions, priority chain, model support detection |
| `utils/modelCost.ts` | ~200 | Price table, cost calculation functions |
| `utils/model/model.ts` | ~450 | Model priority chain, alias resolution, default model logic |
| `utils/model/modelCapabilities.ts` | ~100 | Model capability caching (internal users only) |
| `query.ts` | ~1000 | FallbackTriggeredError handling, user notification, model switching |
| `migrations/*.ts` | 11 files | Model version migration scripts |
| `tools/WebFetchTool/utils.ts:503` | — | Haiku use case 1: Web Fetch content extraction |
| `utils/shell/prefix.ts:220` | — | Haiku use case 2: Shell command prefix judgment |
| `utils/sessionTitle.ts:87` | — | Haiku use case 3: Session title generation |
| `utils/mcp/dateTimeParser.ts:68` | — | Haiku use case 4: DateTime parsing |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Haiku use case 5: Tool call summary |
| `commands/rename/generateSessionName.ts:20` | — | Haiku use case 6: Session renaming |
| `services/api/claude.ts:544` | — | Haiku use case 7: API Key validation |

---
