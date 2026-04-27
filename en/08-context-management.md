# Chapter 8: Context Management

## 8.1 Overview and Role

Context management is one of the most critical subsystems in the Claude Code architecture. A typical coding session can last for hours, involving hundreds of tool calls and generating hundreds of thousands of tokens of conversation history. Without management, the context window would be exhausted after 20–30 exchanges, causing the session to break.

The core problem the Claude Code context management system solves is: **How do you maintain session continuity and information integrity within a limited context window (typically 200K tokens) while minimizing user-perceived information loss?**

This system consists of 11 files under the `services/compact/` directory, totaling approximately 3,900 lines of TypeScript code, supplemented by two key utility modules: `utils/collapseReadSearch.ts` (1,109 lines) and `utils/toolResultStorage.ts` (1,040 lines). The entire subsystem's design embodies three core principles:

1. **Graceful Degradation**: From zero-cost microcompaction to lossy full compaction, escalating intervention in stages
2. **Cache-First**: Every compaction decision prioritizes preserving the prompt cache
3. **Safety Invariants**: tool_use/tool_result pairs must not be severed; recursive protection; circuit breaker mechanism

## 8.2 Theoretical Foundations

### 8.2.1 Information Theory Perspective: Lossy vs. Lossless Compression

Context management is fundamentally an **information compression problem**. Claude Code's multi-layer system corresponds to different compression strategies:

- **Lossless**: The microcompact `cache_edits` path — through the API's cache editing mechanism, old tool result server-side cache copies are deleted while local message content remains unchanged. The model sees a `[Old tool result content cleared]` placeholder, but the original data is saved on disk (`toolResultStorage.ts`). Information is not lost, only moved from hot storage to cold storage.
- **Lossy**: Full compaction generates a summary via a Fork Agent, compressing tens of thousands of tokens of conversation into thousands of tokens. This is an irreversible dimensionality reduction — code details, error stacks, and intermediate reasoning may all be lost.

From the perspective of Rate-Distortion Theory, Claude Code's design implicitly defines a **distortion measure function**: the 9 sections in the summary prompt (see Section 8.6) define which information dimensions tolerate the least distortion — "user messages" (fully preserved) have higher priority than "key technical concepts" (allowed to be summarized).

### 8.2.2 Cache Theory: Temporal and Spatial Locality

The whitelist mechanism of microcompaction embodies the classic cache theory assumption of **Temporal Locality**:

> Recently used tool results are more likely to be referenced subsequently.

The whitelist (`COMPACTABLE_TOOLS`) in `microCompact.ts` is a manifestation of eviction policy — only specific tools' (Read, Shell, Grep, Glob, WebFetch, WebSearch, Edit, Write) results can be cleared, because their outputs are regenerable (the tool can be re-executed to retrieve them). User-typed text, images, and other non-regenerable content are never cleared.

```typescript
// microCompact.ts:30-41 — Compactable tool whitelist
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

The `keepRecent` parameter (default: keep the 5 most recent) directly implements an LRU (Least Recently Used) eviction policy.

### 8.2.3 Circuit Breaker Pattern

The circuit breaker mechanism in `autoCompact.ts` is a precise adaptation of the classic distributed systems Circuit Breaker Pattern (from Michael Nygard's *Release It!*) to LLM applications. Its three-state model (Closed → Open → Half-Open) in Claude Code:

```typescript
// autoCompact.ts:70-73 — Circuit breaker threshold
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

This comment reveals the real disaster data from before the circuit breaker existed: **1,279 sessions were stuck in loops of 50+ consecutive failures**, with the worst single session reaching 3,272 failed attempts, wasting approximately 250K API calls per day globally. The circuit breaker caps the maximum retries at 3.

| State | Behavior | Corresponding Code |
|-------|----------|-------------------|
| Closed (normal) | `consecutiveFailures < 3`, attempts compaction normally | `autoCompactIfNeeded` default path |
| Open (tripped) | `consecutiveFailures >= 3`, skips compaction | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open (probing) | Resets `consecutiveFailures` to 0 on successful compaction | `consecutiveFailures: 0` on success |

## 8.3 Architecture Overview

### 8.3.1 Overall Architecture of the Multi-Layer Compression System

Claude Code's context management uses a **5-layer defense** design. Arranged from low to high intervention:

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Request                              │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage (Prevention Layer)                  │
│   Large tool results → disk persistence + 2KB preview           │
│   Trigger: result > threshold (default 50K chars)               │
│   Cost: zero context cost (stored on disk, preview in context)  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact                                            │
│   Path A: Time-based — clear old tool result content            │
│   Path B: Cache editing — cache_edits API removes server cache  │
│   Trigger: before every API call                                 │
│   Cost: minimal (tool results replaced by placeholder,          │
│          recoverable from disk)                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact                                            │
│   Session Memory → Fork Agent → Full summary                    │
│   Trigger: tokens exceed effectiveContextWindow - 13K           │
│   Cost: high (lossy summary, detail loss, one API call)         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact                                         │
│   User-triggered, supports Partial Compact                       │
│   Trigger: user command                                          │
│   Cost: same as above                                            │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact                                        │
│   API returns prompt_too_long → truncate and retry              │
│   Trigger: 413 error                                             │
│   Cost: highest (emergency truncation + summary,                │
│          maximum information loss)                               │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 Comparison of Trigger Conditions, Cost, and Information Loss Per Layer

| Layer | Trigger Condition | Timing | Latency | Information Loss | API Cost |
|-------|------------------|--------|---------|-----------------|---------|
| L0: Tool Result Storage | Single tool result > threshold | After tool execution | Disk I/O | Zero (original saved to disk) | Zero |
| L1a: Time-based MC | > 60 min since last assistant | Before API call | Zero (local op) | Low (old results cleared) | Zero |
| L1b: Cached MC | Compactable tool count exceeds threshold | Before API call | Zero (cache_edits) | Low (same) | Zero |
| L2: Auto-Compact | tokens > threshold | Between turns | 5–15s (API call) | High (lossy summary) | 1 API call |
| L3: Manual Compact | User /compact | User-triggered | Same | Medium–High (user can guide) | 1 API call |
| L4: Reactive Compact | prompt_too_long 413 | After API failure | 10–30s (retry) | Highest (truncation + summary) | 1–4 API calls |

### 8.3.3 Data Flow

```
Message array (Message[])
    │
    ▼
microcompactMessages()  ──→ [time-triggered?] ──Y──→ content cleared → return
    │ N                      │
    │                  [cache editing?] ──Y──→ pendingCacheEdits → return
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     no compaction, return directly
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [has session memory?]
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
New message array: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 Layer 1: Microcompact

Microcompact is the first line of defense in context management. It runs **before every API call** (via the `microcompactMessages` entry point), aiming to free context space at minimal cost.

### 8.4.1 Compactable Tool Whitelist

Microcompact only operates on the output of specific tools. The design principle behind the whitelist is: **only clear regenerable content**.

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // File read — can be re-read
  ...SHELL_TOOL_NAMES,     // Shell commands — can be re-executed
  GREP_TOOL_NAME,          // Search — can be re-searched
  GLOB_TOOL_NAME,          // File matching — can be re-matched
  WEB_SEARCH_TOOL_NAME,    // Web search — can be re-searched
  WEB_FETCH_TOOL_NAME,     // Web fetch — can be re-fetched
  FILE_EDIT_TOOL_NAME,     // File edit — result already saved to disk
  FILE_WRITE_TOOL_NAME,    // File write — same
])
```

Note that `apiMicrocompact.ts` defines a finer-grained distinction:

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // clears tool_result content
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // clears tool_use input
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

The distinction is subtle: for Read/Grep/Shell, the **output** (tool_result) is cleared; for Edit/Write, the **input** (tool_use input) is cleared, because the input of edit operations (diff content) is large but the result is already persisted to disk.

### 8.4.2 Two Sub-Paths in Detail

Microcompact has two mutually exclusive execution paths, dispatched by the `microcompactMessages()` function:

```typescript
// microCompact.ts:287-317 — Dispatch logic
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // Path A: time-based — highest priority, short-circuits subsequent paths
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // Path B: cache editing — main thread only, specific models only
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

**Path A: Time-based Microcompact**

Triggered when the user returns to a session after being away longer than the configured time threshold (default 60 minutes). The rationale is stated clearly in `timeBasedMCConfig.ts`:

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

The server-side prompt cache TTL is 1 hour. A user absence of over 1 hour means **the cache has definitely expired**, and the entire prompt prefix must be rewritten. Clearing old tool results at this point is "free" — it causes no additional cache invalidation cost.

Key logic for time-based triggering:

```typescript
// microCompact.ts:381-389 — Time-based trigger evaluation
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

After time-based triggering, the clearing strategy also uses LRU (`keepRecent` defaults to 5), but with a boundary guard:

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

This `Math.max(1, ...)` guards against the JavaScript trap where `slice(-0)` returns the entire array when `keepRecent=0` — a classic case of "defensive programming to avoid semantic ambiguity."

After time-based triggering, the cache edit state must also be reset:

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**Path B: Cached Microcompact**

This is an advanced internal-only optimization path (`feature('CACHED_MICROCOMPACT')`), using the API's `cache_edits` mechanism to delete tool results from server-side cache **without modifying local message content**.

```typescript
// microCompact.ts:327-370 — Core of the cache editing path
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // Register tool results
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
      messages,  // messages unchanged!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

Key design decision: **the message array is unchanged** — `return { messages }` returns the original reference. Cache editing happens at the API layer (via the `cache_edits` parameter); local state remains intact. This means that if an API call fails or is retried, there are no local side effects.

### 8.4.3 State Management for Cache Editing

The cache editing path maintains three key module-level state variables:

```typescript
// microCompact.ts:43-49 — Module-level state
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

The lifecycle management of these three states is subtle:

- `pendingCacheEdits` is one-shot — `consumePendingCacheEdits()` reads and clears it (`microCompact.ts:80-84`); the caller must pin it after including it in the API request.
- `pinnedCacheEdits` is cumulative — each successful cache edit is pinned to a specific user message position; subsequent requests must resend it at the same position to ensure cache hits.
- `cachedMCState` is reset after compaction (`resetMicrocompactState()`) or after time-based triggering.

```typescript
// microCompact.ts:78-105 — State consumption and pinning
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

### 8.4.4 Token Estimation Helper Functions

The microcompact module provides a system-wide shared token estimation function:

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
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // fixed at 2000
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
  return Math.ceil(totalTokens * (4 / 3))  // 4/3 conservative padding
}
```

The core formula of `roughTokenCountEstimation` is extremely concise: `Math.round(content.length / 4)` (`tokenEstimation.ts:203-207`). `estimateMessageTokens` then multiplies this result by a 4/3 conservative coefficient, equivalent to `text.length / 3`. This double-conservative strategy ensures the probability of underestimation is extremely low.

## 8.5 Layer 2: Auto-Compact

### 8.5.1 Threshold Calculation Formula

The auto-compact trigger threshold is calculated by the following formula:

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

Concrete derivation (using Claude Opus 200K as an example):

```
contextWindow = 200,000
maxOutputTokens = 16,384 (or model-specific value)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (based on p99.99 = 17,387)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — Key constants
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

The choice of `AUTOCOMPACT_BUFFER_TOKENS = 13,000` is an engineering tradeoff: too small and compaction triggers too often (each compaction consumes 5–15 seconds and one API call); too large and usable context is wasted. 13K is roughly the space for 3–5 ordinary conversation turns.

### 8.5.2 shouldAutoCompact Decision Tree

```typescript
// autoCompact.ts:127-178 — Complete decision chain
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. Recursive protection: session_memory and compact query sources don't trigger
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. Context collapse protection: marble_origami (ctx-agent) doesn't trigger
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. Config check: is auto-compact enabled by user
  if (!isAutoCompactEnabled()) return false

  // 4. Reactive mode: if enabled, suppress proactive compaction
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. Context collapse mode: collapse IS context management, compaction should not interfere
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Token count + threshold comparison
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

This decision tree reveals multiple context management strategies Claude Code is experimenting with in parallel:
- **Reactive Compact** (`tengu_cobalt_raccoon`): no proactive compaction; wait for API to report prompt_too_long before responding
- **Context Collapse** (`CONTEXT_COLLAPSE`): manage context in a streaming fashion blocking at 90% submit / 95% block
- **Auto Compact** (current default): proactively compact at threshold

The three are mutually exclusive, controlled via feature flags.

### 8.5.3 Circuit Breaker Mechanism

```typescript
// autoCompact.ts:219-272 — autoCompactIfNeeded with circuit breaker
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // Circuit breaker check
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // tripped state, skip immediately
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // Prefer Session Memory compaction
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // Traditional compaction
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

### 8.5.4 autoCompactIfNeeded Execution Flow

Complete execution order:

1. **Environment variable check**: `DISABLE_COMPACT` → global disable
2. **Circuit breaker check**: `consecutiveFailures >= 3` → skip
3. **Threshold check**: `shouldAutoCompact()` → multi-layer gate
4. **Session Memory compaction** (preferred path): uses existing session memory instead of an API call
5. **Traditional Fork Agent compaction** (fallback path): complete API-driven summary generation
6. **Failure handling**: increment circuit breaker counter, carry over to next round

## 8.6 Layer 3: Traditional Compaction (Full Compact)

### 8.6.1 Fork Agent Mechanism

The core of traditional compaction is generating a conversation summary via a Fork Agent. The `streamCompactSummary()` function (`compact.ts:1136-1396`) implements a two-level fallback strategy:

**Level 1: Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — Cache-sharing fork
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

The Fork Agent reuses the full prompt cache of the main conversation (system prompt + tools + context messages), appending only one summary request. Key design choices:

1. `maxTurns: 1` — no multi-turn interaction allowed
2. `canUseTool: createCompactCanUseTool()` — rejects all tool calls
3. `skipCacheWrite: true` — no cache write (temporary fork)
4. **maxOutputTokens not set** — the comment explains: setting it would change the thinking config, causing a cache key mismatch

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**Level 2: Streaming Fallback**

When the Fork Agent fails, falls back to a direct streaming API call, at which point `maxOutputTokensOverride` **can** be set:

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

The streaming fallback also supports configuration-driven retries (`tengu_compact_streaming_retry`), up to `MAX_COMPACT_STREAMING_RETRIES = 2` times.

### 8.6.2 Pre-Processing Pipeline

Before compaction, messages go through three preprocessing steps:

```typescript
// compact.ts:1293-1300 — Preprocessing chain
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

1. `getMessagesAfterCompactBoundary` — only takes messages after the last compaction
2. `stripReinjectedAttachments` — removes `skill_discovery` / `skill_listing` attachments (they will be re-injected after compaction)
3. `stripImagesFromMessages` — replaces image blocks with `[image]` text markers (`compact.ts:144-199`)

The rationale for `stripImagesFromMessages` is practical:

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

CCD (Claude Code Desktop) users frequently attach screenshots; without stripping images, the compaction API call itself could fail due to an overly long prompt.

### 8.6.3 The 9-Section Format of Summary Output

`prompt.ts` defines the 9 structured sections the summary must follow:

```
1. Primary Request and Intent    — user intent
2. Key Technical Concepts        — technical concepts
3. Files and Code Sections       — files and code snippets
4. Errors and fixes              — errors and fixes
5. Problem Solving               — problem solving
6. All user messages             — all user messages (not tool results)
7. Pending Tasks                 — pending tasks
8. Current Work                  — current work
9. Optional Next Step            — next step (optional)
```

Section 6 is especially important — "List ALL user messages that are not tool results." This ensures that even after a conversation is compacted, the user's original wording is fully preserved. This is the guarantee of **zero information loss from user feedback**.

Section 9 has a carefully designed constraint:

```
// prompt.ts — constraint for section 9
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

This prevents the post-compaction model from "acting on its own" — only next steps explicitly requested by the user will be recorded.

### 8.6.4 NO_TOOLS_PREAMBLE Anti-Bypass Design

The Fork Agent inherits the complete tool set from the main conversation (for cache key matching), but the compaction agent should not use any tools. This creates a contradiction: tools exist in the schema, but should not be called.

The solution is a **three-layer tool rejection**:

```typescript
// prompt.ts:16-24 — Layer 1: strong declaration at prompt start
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — Layer 2: repeated reminder at prompt end
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — Layer 3: code-level rejection
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

The comments reveal the real reason for these three layers:

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

On Sonnet 4.6, prompt instructions alone have a 2.79% probability of still attempting a tool call (only 0.01% on 4.5). `createCompactCanUseTool` is the final code-level safeguard.

### 8.6.5 Post-Processing (formatCompactSummary)

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // Strip <analysis> draft area — intermediate reasoning that improves summary quality, not needed to preserve
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // Extract <summary> content
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

The `<analysis>` tag design is a Chain-of-Thought technique: the model "drafts" in the analysis area first, then outputs the final result in `<summary>`. The analysis area improves summary quality, but is stripped from the final output — because it contains redundant intermediate reasoning that would waste context space in subsequent turns.

### 8.6.6 Post-Compact Message Sequence and Attachment Re-injection

After compaction, the new message sequence is built by `buildPostCompactMessages()`:

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // system message: marks compaction boundary
    ...result.summaryMessages,    // user messages: summary content
    ...(result.messagesToKeep ?? []),  // preserved original messages
    ...result.attachments,        // file attachments + skills + plans
    ...result.hookResults,        // SessionStart hook results
  ]
}
```

Attachment re-injection is a complex process (`compact.ts:532-585`), including:

1. **File attachments**: the 5 most recently accessed files, subject to a 50K token budget, up to 5K tokens per file
2. **Plan files**: if there is an active plan
3. **Plan mode instructions**: if in plan mode
4. **Skill content**: content of invoked skills, sorted by most recent use, up to 5K tokens each, 25K token total budget
5. **Deferred Tools Delta**: re-declares the schema of deferred-loaded tools
6. **Agent Listing Delta**: re-declares the agent list
7. **MCP Instructions Delta**: re-declares MCP server instructions

### 8.6.7 PTL Retry Mechanism (Prompt-Too-Long Recovery)

When the compaction API call itself fails due to an overly long prompt, the system retries through progressive truncation:

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // First clear marker messages left by previous retries
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // Precise truncation: based on token gap returned by API
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // Fuzzy truncation: drop 20% of message groups
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // keep at least one group
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

The retry limit is `MAX_PTL_RETRIES = 3`. The truncation strategy has two paths:
- **Precise path**: API error contains token gap → drop groups based on gap
- **Fuzzy path** (Vertex/Bedrock and other non-standard error formats): drop 20% each time

Note the boundary handling on line 283: after dropping group 0, the message sequence may start with an assistant message, violating the API constraint (first message must be a user message). The system inserts a synthetic user marker message to fix this.

### 8.6.8 Two Directions of Partial Compact

`partialCompactConversation()` (`compact.ts:772-1106`) supports two directions:

```
Direction 'from': 
  [preserved after compaction] | pivot | [messages to summarize]
  → preserves prompt cache (preserved portion comes first, cache prefix unchanged)

Direction 'up_to':
  [messages to summarize] | pivot | [preserved after compaction]
  → prompt cache invalidated (summary comes first, prefix changes)
```

The `up_to` direction has an additional cleanup step — old compact boundaries and summaries must be removed from the preserved messages:

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

The comment explains: in `up_to` mode, the summary precedes the preserved messages, and an old boundary would mislead `findLastCompactBoundaryIndex`'s reverse scan.

## 8.7 Layer 4: Session Memory Compaction

### 8.7.1 Core Idea and Advantages

Session Memory Compaction (`sessionMemoryCompact.ts`) is an optimized alternative to traditional compaction. The core idea: use the session memory continuously extracted in the background (an incremental summary of the conversation) instead of a real-time Fork Agent summary.

Advantages:
- **Zero additional API calls**: session memory is continuously maintained by a background agent; at compaction time it is used directly
- **Lower latency**: no need to wait 5–15 seconds for an API response
- **Finer-grained retention**: can precisely calculate how many recent messages to keep

### 8.7.2 calculateMessagesToKeepIndex Algorithm in Detail

This is the core algorithm of Session Memory Compaction (`sessionMemoryCompact.ts:262-327`), determining how many messages to keep after compaction:

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // Start from lastSummarizedIndex + 1 (session memory already covers earlier content)
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // Calculate token count and text message count for current retained range
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // Already at upper limit → do not expand
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Both minimum requirements already met → do not expand
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Expand forward, but do not cross the last compact boundary
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

Configuration parameters (can be overridden via GrowthBook remote config):

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // keep at least 10K tokens
  minTextBlockMessages: 5,     // keep at least 5 messages with text
  maxTokens: 40_000,           // keep at most 40K tokens
}
```

The algorithm's dual-constraint design (`minTokens` AND `minTextBlockMessages`) ensures:
- Expansion does not stop just because a few very large messages satisfy the token requirement (token requirement met but too few messages)
- Retention of many small messages with insufficient actual token count is avoided

**Floor mechanism**: when expanding forward, cannot cross the last compact boundary (`floor = lastBoundaryIndex + 1`). The comment explains:

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

The disk storage layer's message chain has a discontinuity at the compact boundary; crossing it would cause the loader's reverse traversal to skip retained messages.

### 8.7.3 Bug Fix in adjustIndexToPreserveAPIInvariants

This function (`sessionMemoryCompact.ts:172-260`) is the most subtle piece of code in the entire compaction system, solving two API invariant problems:

**Bug Scenario 1: Orphaned tool_result**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

If startIndex = N+2:
  Old code only checks message N+2's tool_results → not found → returns N+2
  After normalizeMessagesForAPI merges by message.id:
    msg[1]: assistant with [tool_use: VALID_ID]  (ORPHAN tool_use excluded!)
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → API error: orphan tool_result references a non-existent tool_use
```

**Bug Scenario 2: Missing thinking block**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

If startIndex = N+1:
  thinking block at N is excluded
  normalizeMessagesForAPI cannot merge (no message with same ID to merge with)
  → thinking block permanently lost
```

The fix executes two adjustment steps:

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Step 1: handle tool_use/tool_result pairing
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... collect tool_use IDs already in the kept range
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // search backward for missing tool_use
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // delete found IDs
      }
    }
  }

  // Step 2: handle thinking blocks sharing message.id
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

The key insight in this code: the Claude API's streaming response splits a single API reply into multiple assistant messages (sharing `message.id`, but with different UUIDs), where thinking blocks and tool_use blocks are separate. `normalizeMessagesForAPI` merges these by `message.id` — if compaction severs a same-ID message group, the merged result will be inconsistent.

### 8.7.4 trySessionMemoryCompaction Complete Flow

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. Gate check
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. Initialize remote config (first time only)
  await initSessionMemoryCompactConfig()

  // 3. Wait for any ongoing session memory extraction to complete
  await waitForSessionMemoryExtraction()

  // 4. Get session memory content
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. Determine boundary
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID not found → fall back
  } else {
    // Resumed session: no boundary → start from end
    lastSummarizedIndex = messages.length - 1
  }

  // 6. Calculate retention range
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // filter old boundaries

  // 7. Run session start hooks
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. Build result
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. Threshold check (auto-compact only)
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // still exceeds threshold after compaction → fall back to traditional compaction
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 Configuration Parameters (GrowthBook Remote Config)

All key parameters for Session Memory Compaction can be overridden via GrowthBook remote config:

```typescript
// sessionMemoryCompact.ts:91-109 — Remote config initialization
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // Defensive coding: only use positive values, ignore 0 values
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

The feature gate is controlled by two independent feature flags:

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

## 8.8 Context Folding and Tool Result Storage

### 8.8.1 collapseReadSearch Mechanism

`utils/collapseReadSearch.ts` (1,109 lines) implements UI-layer message folding — collapsing consecutive search/read operations into single-line summaries (e.g., "Read 5 files, searched 3 patterns").

Core classification logic (`getToolSearchOrReadInfo`, `collapseReadSearch.ts:142-237`) categorizes tool calls as:

| Category | Collapsible | Fold Behavior |
|----------|-------------|---------------|
| Read (file_path) | Yes | "Read N files" |
| Search (Grep/Glob) | Yes | "Searched N patterns" |
| Shell (Bash) | Yes in fullscreen mode | "Ran N bash commands" |
| REPL | Yes (silently absorbed) | Internal tool counted separately |
| Memory Write | Yes | Special marker |
| ToolSearch | Yes (silently absorbed) | Does not increment counter |
| Edit/Write (non-memory) | No | Breaks fold group |

"Silently absorbed" (`isAbsorbedSilently`) is a subtle design: REPL and ToolSearch do not increment the counter but also do not break the current fold group. This means `[Read, ToolSearch, Read]` folds to "Read 2 files" instead of being split into two groups by ToolSearch.

Folding is a **UI-layer only** optimization — it does not change the messages sent to the API, only affects terminal display.

### 8.8.2 toolResultStorage Disk Storage Strategy

`utils/toolResultStorage.ts` (1,040 lines) is the "Layer 0" of context management — handling oversized results before they enter the conversation history.

**Persistence threshold analysis**:

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Read tool special case: Infinity → no persistence (Read has its own maxTokens limit)
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // GrowthBook override (tengu_satin_quoll)
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // Default: min(tool declared value, global 50K default)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**Deduplication optimization**: `tool_use_id` is unique; uses `flag: 'wx'` (exclusive write) to avoid duplicate writes:

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: already persisted in a previous turn, skip
}
```

**Empty result handling**:

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

This fix addresses a model behavior bug: empty tool_result causes some models to pattern-match `\n\nHuman:` as the end of conversation.

**Per-Message Aggregate Budget** (`enforceToolResultBudget`, `toolResultStorage.ts:769-908`):

This is the most complex feature in `toolResultStorage.ts` — enforcing a total tool result size budget per API-level user message (after `normalizeMessagesForAPI` merging).

Design highlights:
- **State freezing** (`ContentReplacementState`): once a tool_result has been "seen" (sent to model), its decision (replace/keep) is frozen and never changes — this guarantees prompt cache stability
- **Three-partition** strategy: `mustReapply` (previously replaced → re-apply cached replacement), `frozen` (previously seen but not replaced → do not touch), `fresh` (new → may be replaced)
- **Largest-first**: when replacement is needed, choose the largest fresh result to replace first

## 8.9 Role of Compaction in the 5-Layer Error Recovery

### 8.9.1 Complete 5-Layer Error Recovery Mechanism

The compaction system plays multiple roles in Claude Code's error recovery mechanism:

| Layer | Trigger | Compaction Behavior | Source |
|-------|---------|---------------------|--------|
| L1 | API returns prompt_too_long (413) | Reactive Compact: truncate + re-summarize | `compactMessages.ts` |
| L2 | Compaction API itself returns 413 | PTL Retry: truncate oldest message group × 3 | `compact.ts:truncateHeadForPTLRetry` |
| L3 | Still exceeds threshold after compaction | Re-compaction: automatically compact again | `autoCompact.ts:recompactionInfo` |
| L4 | 3 consecutive compaction failures | Circuit Breaker: stop trying | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent produces no text output | Streaming Fallback: direct streaming API call | `compact.ts:streamCompactSummary` |

### 8.9.2 Reactive Compaction vs. Proactive Compaction

The tradeoffs between the two strategies:

**Proactive Compaction** (Auto-Compact, current default):
- Triggers proactively when tokens reach the threshold
- Pros: smoother user experience, no 413 errors encountered
- Cons: may compact too early, wasting available context

**Reactive Compaction** (Reactive Compact, `tengu_cobalt_raccoon` experiment):
- Waits for API to report prompt_too_long before triggering
- Pros: maximizes context utilization
- Cons: noticeable disruption to user experience; requires waiting for retry

The mutual exclusion between the two is visible in the code:

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // reactive mode: do not proactively compact
  }
}
```

## 8.10 Message Grouping and Token Estimation

### 8.10.1 groupMessagesByApiRound Algorithm

`grouping.ts` (63 lines) groups messages by API round — each group corresponds to one complete API round-trip:

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

The sole criterion for a group boundary is a change in `message.id` — multiple streaming chunks from the same API response share the same `message.id`, so they naturally fall into the same group.

This design replaces the previous "human-turn"-based grouping (only grouping at real user messages), which could not handle long single-turn agent sessions in SDK/CCR/eval scenarios.

### 8.10.2 roughTokenCountEstimation and Conservative Padding

Token estimation uses a two-level conservative strategy:

**Level 1**: Basic estimation

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

Default 4 bytes/token; JSON files use 2 bytes/token (because JSON has many single-character tokens like `{`, `}`, `:`, `,`).

**Level 2**: Message-level padding

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

Combined effect: for ordinary text, the effective estimate is `text.length / 4 * 4/3 = text.length / 3`.

### 8.10.3 Mixed Strategy of Precise vs. Estimated

The system uses different precision levels in different scenarios:

| Scenario | Precision | Source | Latency |
|----------|-----------|--------|---------|
| shouldAutoCompact | Mixed: prioritizes precise value from API response | `tokenCountWithEstimation` | 0 (already cached) |
| estimateMessageTokens | Rough estimate (`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | Rough estimate | `estimateMessageTokens` | 0 |
| Token count after compaction | Precise | `tokenCountFromLastAPIResponse` | 0 (API already returned) |

The mixed strategy of `tokenCountWithEstimation`: prioritizes `usage.input_tokens` (precise value) returned by the most recent API call; falls back to estimation if unavailable (e.g., before the first request).

## 8.11 Design Decision Analysis

### 8.11.1 Progressive Degradation Philosophy

Claude Code's context management adopts **no-skipping** progressive degradation: each layer tries to solve the problem with minimal cost, escalating to the next layer only when the current layer fails. This avoids the common "overreaction" problem — for example, triggering full compaction merely because of a large Read tool result.

Comparison with industry practice:
- **ChatGPT**: truncates old messages (simple but blunt)
- **GitHub Copilot Chat**: fixed context window + most recent N messages (no compaction)
- **Claude Code**: 5-layer escalation (prevention → fine-tuning → summarization → emergency recovery)

### 8.11.2 Cache-First Design

Prompt cache is the lifeline of Claude Code — for a 200K token request, if 180K is cache read ($0.30/M tokens), the cost is 10× lower than a full cache miss ($3/M tokens). Almost every design decision serves this economic constraint:

1. **Fork Agent shares cache prefix**: compaction API call reuses the main conversation's cache
2. **maxOutputTokens not set in Fork**: avoids thinking config mismatch causing cache miss
3. **Cached MC does not modify local messages**: keeps prompt prefix unchanged
4. **ContentReplacementState freezes seen IDs**: ensures the replacement decision for the same tool_result remains unchanged throughout its lifecycle
5. **sentSkillNames not reset**: avoids re-injecting ~4K tokens of skill_listing
6. **pinnedCacheEdits re-sent at fixed positions**: ensures cache edit position consistency

### 8.11.3 Safety Guarantees

The system maintains three categories of invariants:

**Pairs must not be severed**: `adjustIndexToPreserveAPIInvariants` ensures tool_use and tool_result are never split to different sides. This is a requirement for both functional correctness (the API will error) and semantic correctness (the model needs to see the result of the tool it previously called).

**Recursive protection**: the `querySource` check in `shouldAutoCompact` ensures session_memory agent, compact agent, and context collapse agent never trigger auto-compaction — these agents are themselves part of context management; recursive compaction would cause deadlock.

**Circuit breaker mechanism**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` is set based on real data (failure loops in 1,279 sessions), converting infinite retries to bounded retries with tripping.

### 8.11.4 Comparison with API-Native Context Management

`apiMicrocompact.ts` reveals that Claude Code is exploring offloading part of context management to the API layer:

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

These `context_management.edits` strategies are declared directly in the API request and executed server-side. The advantages are lower latency (no client-side processing required) and precise alignment with server-side token counts. Currently, the tool-clearing strategy is only available to internal users (`USER_TYPE === 'ant'`); external users only have access to the thinking-clearing strategy.

## 8.12 Transferable Patterns

### 8.12.1 General Design Patterns from the Multi-Layer Compression System

Claude Code's context management distills the following transferable general patterns:

**Pattern 1: Tiered Eviction**
- Apply different eviction strategies to different types of content
- Regenerable content (tool output) is evicted first; non-regenerable content (user input) is evicted last
- Implementation: whitelist + priority ordering

**Pattern 2: Hybrid Estimation**
- Fast decisions use rough estimates (`text.length / 3`); precise accounting uses API-returned values
- Rough estimates are always conservative (better to overestimate and compact early than underestimate and get an API error)

**Pattern 3: Freeze-Replay**
- Once content has been "seen" by the model, its processing decision is frozen
- Subsequent turns only "replay" (re-apply cached replacement) for frozen content, making no new decisions
- Guarantees bit-level stability of the prompt prefix → cache hit

**Pattern 4: Boundary-Aware Truncation**
- Never truncate in the middle of a semantic unit (tool_use/tool_result pairs, same-ID message groups)
- Actively repair after truncation (insert synthetic messages, adjust indices)

**Pattern 5: Circuit Breaker Protection**
- Set failure counters for operations that could retry infinitely
- Set thresholds based on real operational data (not intuition)

### 8.12.2 What Doramagic Can Learn

In Doramagic's Soul Extractor pipeline, the extraction process can generate large amounts of intermediate results (code snippets, API documentation, community discussions). Patterns to borrow:

1. **Layered extraction cache**: similar to microcompact's whitelist mechanism, classify intermediate API responses and code analysis results by regenerability, prioritizing eviction of re-fetchable content
2. **Incremental summaries**: similar to Session Memory Compact, maintain incremental summaries of extracted knowledge rather than the full history
3. **Frozen decisions**: once a knowledge block has been confirmed as "valuable" or "valueless," the decision is irreversible — avoids re-evaluating repeatedly across different extraction rounds

## 8.13 Source Index

| File | Lines | Core Responsibility |
|------|-------|-------------------|
| `services/compact/compact.ts` | ~1,705 | Traditional compaction main logic: Fork Agent, PTL retry, attachment re-injection, partial compact |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Session Memory Compaction: calculateMessagesToKeepIndex, adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | Microcompact: time-based trigger, cache editing, token estimation |
| `services/compact/prompt.ts` | ~374 | Compaction prompts: 9-section template, NO_TOOLS_PREAMBLE, formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | Auto-compact: threshold calculation, shouldAutoCompact decision chain, circuit breaker |
| `services/compact/apiMicrocompact.ts` | ~153 | API-native context management: clear_tool_uses, clear_thinking |
| `services/compact/grouping.ts` | ~63 | Message grouping: groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | Post-compaction cleanup: reset cache, module state, classifiers |
| `services/compact/timeBasedMCConfig.ts` | ~43 | Time-based trigger config: GrowthBook remote config |
| `services/compact/compactWarningHook.ts` | ~16 | React hook: compact warning suppression state subscription |
| `services/compact/compactWarningState.ts` | ~18 | State storage: compact warning suppression flag |
| `services/cost-tracker.ts` | ~323 | Cost tracking: token billing, model usage statistics |
| `utils/collapseReadSearch.ts` | ~1,109 | Context folding: UI-layer message grouping and folding |
| `utils/toolResultStorage.ts` | ~1,040 | Tool result storage: disk persistence, per-message budget, ContentReplacementState |
