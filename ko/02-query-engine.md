# 2장: Query Engine — LLM 인터랙션 핵심

## 2.1 개요 및 위치

**한 줄 위치:** Query Engine은 Claude Code의 "심장 박동"이다—LLM 대화의 완전한 생명주기를 소유하며, 사용자 입력을 권한 제어, 도구 실행, 컨텍스트 압축, 비용 추적을 거친 다중 라운드 agentic 인터랙션으로 변환한다.

**해결하는 핵심 문제:** async generator pipeline에서 "LLM 스트리밍 응답 → 도구 호출 → 결과 주입 → 재요청" 루프를 신뢰성 있게 편성하는 방법, 그리고 context window 오버플로, 토큰 예산, API 오류, 사용자 중단, 권한 승인 등 최소 12종의 예외 분기를 처리하는 방법.

**관련 파일과 코드량 통계:**

| 파일 | 행수 | 직책 |
|------|------|------|
| `QueryEngine.ts` | 1,295 | 세션 수준 상태 관리, SDK/headless 진입점 |
| `query.ts` | 1,729 | 쿼리 주 루프, 도구 실행 편성, 다중 계층 복구 |
| `query/config.ts` | 46 | 불변 설정 스냅샷 |
| `query/tokenBudget.ts` | 93 | Token 예산 auto-continue 결정 |
| `query/stopHooks.ts` | 473 | 정지 시 훅 (보안 게이트, 후처리) |
| `query/deps.ts` | 40 | 의존성 주입 인터페이스 (테스트 친화적) |
| `services/api/claude.ts` | 3,419 | API 호출, 스트리밍 파싱, 재시도, 캐시 전략 |
| `cost-tracker.ts` | 323 | 비용 추적과 usage 누적 |
| **합계** | **~7,418** | — |

이 7,400+ 행의 코드는 Claude Code 전체 코드의 약 1.4%를 구성하지만, 전체 제품에서 가장 핵심적인 경로다—사용자와 Claude의 모든 인터랙션이 반드시 여기를 통과한다.

---

## 2.2 이론적 기반

### 2.2.1 Async Generator Pipeline (코루틴 파이프라인)

Query Engine의 핵심 아키텍처는 **ES2018 Async Generator**를 스트리밍 처리 원시 타입으로 사용한다. `query()` 함수 서명이 이 설계를 드러낸다:

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

이것은 우연한 문법적 선택이 아니라 **Coroutine 이론**의 정확한 적용이다. Async generator는 동시에:
- **지연 평가**: 소비자 구동, 모든 API 응답을 미리 계산하지 않음
- **양방향 통신**: stream event를 yield하고, terminal reason을 return
- **리소스 안전**: finally 블록이 모든 종료 경로(`.return()`과 예외 포함)에서 stream 해제 보장(`claude.ts`의 `releaseStreamResources()`)

전통적인 callback이나 Promise chain은 "UI에 스트리밍 출력"과 "도구 실행 결과 대기"라는 두 가지 요구를 동시에 만족할 수 없다. Async generator는 자연스럽게 이 "producer가 consumer 소비를 기다리며 일시 정지"하는 시맨틱을 지원한다.

### 2.2.2 State Machine (암묵적 상태 머신)

`query.ts`의 주 루프는 재귀 호출이 아니라(초기 코드 주석에 여전히 `query_recursive_call`이라고 부르지만), **while(true) + continue 구동의 명시적 상태 머신**이다:

```typescript
// query.ts:218 (State 타입 정의)
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

매 `continue` 시 전체 State가 재구성(불변 업데이트)되며, `transition` 필드가 전이 원인을 기록한다. 이것은 고전적 **Mealy Machine**—출력이 현재 상태와 입력 이벤트의 조합에 따라 결정된다.

### 2.2.3 Backpressure와 리소스 관리

LLM streaming에서 backpressure는 매우 중요하다. Claude Code의 방안:

1. **for-await-of 자연적 속도 제한**: `claude.ts`의 stream 소비는 `for await (const part of stream)`을 사용하여, consumer의 처리 속도가 직접 producer의 푸시 속도를 결정
2. **Stream idle watchdog** (`claude.ts:2397`): 90초 동안 chunk를 받지 못하면 주동적으로 stream을 abort하고 non-streaming으로 fallback
3. **Generator lifecycle guarantee**: `finally` 블록이 `releaseStreamResources()`가 모든 종료 경로에서 실행되도록 보장

### 2.2.4 왜 이 이론들이 LLM 시나리오에서 특히 중요한가

전통적인 HTTP API 호출은 "요청-응답" 모델로 에러 처리가 간단하다. LLM agentic loop는 고유한 문제를 갖는다:

- **단일 호출이 10분 지속될 수 있다** (non-streaming 한계)
- **응답 중간에 새로운 I/O를 트리거할 수 있다** (도구 호출)
- **context window는 상태를 가진 희소 자원**으로, "압축 정보 손실"과 "오버플로 충돌" 사이에서 균형을 잡아야 함
- **비용이 실시간으로 누적**되어 언제든 중단 가능해야 함

이 제약들이 Event Loop + 상태 머신 + Backpressure를 없어서는 안 될 이론적 지지대로 만든다.

---

## 2.3 아키텍처와 데이터 구조

### 2.3.1 QueryEngine 클래스 핵심 인터페이스

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

**설계 요점:** QueryEngine은 per-conversation이다. 각 `submitMessage()` 호출은 같은 대화의 새로운 turn이며, 상태는 turn을 넘어 유지된다.

### 2.3.2 핵심 타입 정의

**QueryEngineConfig** (`QueryEngine.ts:95-153`) — 생성 시 전달되는 불변 설정:

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

**QueryConfig** (`query/config.ts:18-31`) — per-query 불변 환경 스냅샷:

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

소스 주석의 설계 의도에 주목하라 (`config.ts:9-12`): "Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." 이것이 bun:bundle 컴파일 최적화와 런타임 설정의 명확한 경계다.

### 2.3.3 모듈 간 의존 관계 다이어그램

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- 세션 상태 관리
                    |  (1,295 lines)  |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- 주 루프 & 상태 머신
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

**deps.ts의 의존성 주입 설계** (`deps.ts:18-37`)는 특별히 주목할 만하다:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" — 이것은 TypeScript 의존성 주입의 모범 사례: 인터페이스를 직접 작성할 필요 없이 서명이 자동으로 구현을 추적한다.

---

## 2.4 핵심 알고리즘과 흐름

### 2.4.1 query() 주 루프 전체 실행 흐름

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    query() 진입점
                        |
                +=======+========+  <--- while(true) 주 루프 시작
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        blocking-limit 확인      |
                |                |
                v                |
        callModel() [스트리밍]   |
                |                |
        +-------+--------+      |
        | 스트리밍 이벤트 소비  |      |
        | tool_use 수집    |      |
        | streaming exec  |      |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           YES           |
       |              \          |
       v               v        |
  +---------+   runTools() 또는  |
  | 복구 로직 |   getRemainingR() |
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

### 2.4.2 도구 호출 루프 (Tool-call Loop)

도구 호출의 핵심 혁신은 **streaming tool execution**—LLM 스트리밍 응답이 *아직 진행 중일 때* 도구를 실행하기 시작한다:

```typescript
// query.ts:443 (StreamingToolExecutor 초기화)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

스트리밍 소비 루프에서 (`query.ts:536`):

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

도구는 `content_block_stop` 시점에 실행이 제출되며, 전체 assistant 응답이 끝날 때까지 기다리지 않는다. 즉 LLM이 3개의 tool_use 블록을 출력할 때, 앞의 두 개는 세 번째가 아직 스트리밍 중일 때 이미 실행이 완료될 수 있다.

### 2.4.3 스트리밍 처리의 구체적 구현

`claude.ts`의 `queryModel()`은 SSE stream 파싱을 수동으로 구현했으며, **의도적으로 Anthropic SDK의 BetaMessageStream을 우회했다**:

```typescript
// claude.ts 주석 (약 line 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

스트리밍 상태 누적 모델:

```
message_start → partialMessage = part.message, usage = initial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → yield AssistantMessage (블록당 하나씩!)
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (종료)
```

핵심 설계: **각 content 블록이 독립적으로 AssistantMessage를 yield한다**. 즉, LLM 응답 하나에 text + tool_use가 포함될 때, UI는 text가 완료되면 즉시 표시할 수 있으며 tool_use의 JSON이 완료될 때까지 기다릴 필요가 없다.

### 2.4.4 재시도와 에러 복구의 5층 메커니즘

Claude Code의 에러 복구 아키텍처는 종심방어(defense in depth)로, 총 5층이다:

**제1층: withRetry (API 수준)** — `claude.ts`의 `withRetry()`가 429(rate limit), 529(overload), 5xx 등의 재시도 가능한 에러를 처리하며, 지수 백오프와 model fallback을 포함한다.

**제2층: Streaming → Non-streaming Fallback** — 스트리밍 연결이 중단될 때 (`claude.ts:2592`):

```typescript
// Fall back to non-streaming mode with retries
const result = yield* executeNonStreamingRequest(...)
```

stream idle watchdog(90초 데이터 없을 시 타임아웃)과 404 stream creation fallback도 포함한다.

**제3층: max_output_tokens 복구** — `query.ts`의 3단계 점진적 복구:

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// 첫 번째 단계: 64k tokens로 에스컬레이션 (ESCALATED_MAX_TOKENS)
// 두 번째~네 번째 단계: meta message를 주입하여 "Resume directly — no apology, no recap" 요청
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**제4층: Prompt-too-long 복구** — 3단계 연동:
1. Context collapse drain (접힘 대기 중인 것 배출)
2. Reactive compact (긴급 전체 압축)
3. 에러를 표면화하고 종료 (무한 루프 방지)

소스에는 무한 루프를 방지하는 명시적 가드가 있다 (`query.ts` 주석):

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**제5층: Model Fallback** — `query.ts:673-719`에서, `FallbackTriggeredError`가 잡히면 fallback model로 전환하고 전체 요청을 재시도한다.

### 2.4.5 Token 계산과 예산 관리

Token 예산 시스템은 두 가지 독립적인 메커니즘으로 나뉜다:

**메커니즘 A: API task_budget** — 서버 측에서 감지하는 token 예산으로, compaction 경계를 넘어 추적한다:

```typescript
// query.ts:270-280 (taskBudgetRemaining 크로스 압축 추적)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**메커니즘 B: Client-side token budget auto-continue** (`tokenBudget.ts`) — turn 출력이 예산의 90%에 도달하지 않으면 자동으로 계속:

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
  // 하위 agent는 auto-continue를 하지 않음
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

**diminishing returns 감지**에 주목하라: 연속 3회 continuation이며 매번 증가량이 500 tokens 미만일 때 자동으로 중지하여, 모델이 비효율적인 출력에 예산을 낭비하는 것을 방지한다.

### 2.4.6 Thinking Mode 처리

소스의 "wizard comment"가 thinking mode의 3가지 철칙을 요약한다 (`query.ts:105-118`):

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

`claude.ts`의 thinking 파라미터 구성 로직 (약 line 2242):

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

adaptive thinking을 지원하는 모델은 adaptive를 우선 사용(preset budget이 필요 없음), 그렇지 않으면 enabled + budget_tokens로 폴백한다.

### 2.4.7 Prompt Cache 경계 관리

Claude Code의 캐시 전략은 인상적이다. 핵심 설계가 `addCacheBreakpoints()`에 있다 (`claude.ts:3045`):

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**캐시 control 마커를 하나만 배치**하고, 마지막 메시지에 위치시킨다(또는 `skipCacheWrite` 시 뒤에서 두 번째에). 이것은 inference 팀의 KV cache 페이지 관리자(Mycro)와의 협동 최적화 결과다.

1시간 TTL 캐시에는 세밀한 "session stability latching" 메커니즘이 있다 (`claude.ts:380-420`)—eligibility가 한 번 결정되면 세션 전체가 고정되어, 중간에 GrowthBook 설정이 변경되어 cache_control TTL이 뒤집혀 캐시가 깨지는 것을 방지한다.

---

## 2.5 설계 결정 분석

### 2.5.1 핵심 Tradeoff

**Tradeoff 1: 스트리밍 실행 vs 완전성 보장**

StreamingToolExecutor가 LLM이 아직 출력 중일 때 도구를 실행하기 시작하면 지연이 현저히 줄어들지만, 복잡성이 추가된다—스트리밍 중간에 fallback이 발생하면 이미 실행된 도구를 버려야 한다:

```typescript
// query.ts:534-538 (streaming fallback 시 정리)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

이 문제는 이미 버그를 일으킨 적이 있다 (`claude.ts:2575` 주석이 inc-4258을 인용: double tool execution).

**Tradeoff 2: 캐시 안정성 vs 동적 기능**

여러 beta 헤더가 "sticky-on latch" 패턴을 사용한다 (`claude.ts:2102-2126`)—한 번 활성화되면 기능이 꺼져도 세션 전체에서 유지:

```typescript
// claude.ts:2104 주석
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

이것은 "캐시 적중률 우선, 기능 유연성 차선"의 명확한 tradeoff다.

**Tradeoff 3: 상태 머신 vs 재귀**

주 루프는 초기의 재귀 `query()` 호출에서 `while(true)` + State 재구성으로 진화했다. 소스 주석에 여전히 `query_recursive_call` checkpoint 이름이 있지만, 실제로는 이미 반복(iterative) 방식이다. 장점:
- 스택 오버플로 위험 없음 (긴 대화는 수백 turn이 될 수 있음)
- State 재구성이 명시적이어서 디버그가 편리
- `transition` 필드가 완전한 상태 전이 감사 추적을 제공

### 2.5.2 소스 주석이 드러내는 알려진 문제

1. **SDK text delta 중복** (`claude.ts:2350`):

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Non-streaming fallback과 streaming tool execution의 충돌** (`claude.ts:2575`):

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Task budget의 compaction을 넘은 token 계산 편차** (`query.ts:268`):

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 LangChain 등의 방안과의 설계 차이

| 차원 | Claude Code Query Engine | LangChain AgentExecutor |
|------|--------------------------|-------------------------|
| 스트리밍 원시 타입 | ES Async Generator (네이티브) | Callback + Stream wrapper |
| 상태 관리 | 명시적 State struct + 불변 업데이트 | Mutable AgentState dict |
| 도구 실행 | 스트리밍 병렬 가능 (StreamingToolExecutor) | 직렬 await |
| 재시도 | 5층 종심방어 + model fallback | 단순 max_iterations |
| 의존성 주입 | QueryDeps + typeof 서명 동기화 | 런타임 duck typing |
| 캐시 | inference KV cache와 깊은 협력 | 없음 (블랙박스 API 호출) |

가장 근본적인 차이: Claude Code는 **inference-aware**하다—Prompt cache의 물리적 메커니즘(Mycro 페이지 관리자)을 이해하고 이를 기반으로 최적화하는 반면, 오픈소스 프레임워크는 API를 블랙박스로만 다룰 수 있다.

---

## 2.6 이식 가능한 패턴

### 2.6.1 Query Engine에서 추출한 범용 공학 패턴

**패턴 1: Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

각 상태 전이의 **원인**을 state에 기록하면, 디버그와 원격 측정이 first-class concern이 된다. 다단계 결정이 필요한 모든 시스템에 적용 가능하다.

**패턴 2: `typeof`를 통한 타입화된 의존성 주입**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

인터페이스를 직접 작성할 필요 없이, 서명이 자동으로 구현과 동기화된다. 무거운 I/O를 mock해야 하는 모든 시스템에 적용 가능하다.

**패턴 3: Withholding Pattern (에러 노출 지연)**

복구 가능한 에러(prompt-too-long, max_output_tokens)에 대해 먼저 소비자에게 yield하지 않고, 복구 로직 실행 후에 노출 여부를 결정한다:

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

이는 SDK 소비자가 "에러가 이미 복구됨" 상황에서 가짜 에러 신호를 받는 것을 방지한다.

**패턴 4: Session-stable Latching**

캐시 키에 영향을 미치는 설정 항목은 한 번 활성화되면 세션 전체에서 잠근다:

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // bootstrap state에 기록
}
```

"설정 변경이 비용이 큰 리소스를 무효화할 수 있는" 모든 시나리오에 적용 가능하다.

### 2.6.2 Doramagic이 참고할 수 있는 점

Doramagic의 `flow_controller` 파이프라인은 Query Engine과 유사한 편성 문제를 갖고 있으며, 다음을 참고할 수 있다:

1. **State + Transition 패턴**: Doramagic의 12 상태 머신은 `{ state, transition: { reason } }` 구조를 사용하면 디버그를 단순화하고 자연스럽게 로그 감사를 지원한다
2. **typeof를 통한 의존성 주입**: Doramagic의 LLM 호출 계층은 QueryDeps 패턴을 사용하면, 테스트 시 전체 모듈을 mock하는 대신 fake model을 주입할 수 있다
3. **Diminishing returns 감지** (`tokenBudget.ts`): Doramagic의 Soul Extractor는 iterative refinement 시 같은 "연속 N회 증가량이 임계값 이하이면 중지" 전략을 사용하면, LLM이 품질 낮은 출력에 tokens을 낭비하는 것을 방지할 수 있다

---

## 2.7 소스 인덱스

| 파일 | 행수 | 한 줄 직책 |
|------|------|----------|
| `QueryEngine.ts` | 1,295 | 세션 수준 owner: mutableMessages, totalUsage 보유, submitMessage()를 query() 호출로 변환, SDK 메시지 라우팅 처리 |
| `query.ts` | 1,729 | 주 루프 상태 머신: while(true)로 compaction → API call → tool execution → stop hooks → budget check 편성 |
| `query/config.ts` | 46 | 불변 QueryConfig 스냅샷: sessionId + 4 개의 runtime gates (feature() gates는 의도적으로 제외하여 tree-shaking 유지) |
| `query/tokenBudget.ts` | 93 | Client-side token budget auto-continue: 90% 완료도 임계값 + diminishing returns 조기 중지 |
| `query/stopHooks.ts` | 473 | Turn-end 훅 편성: Stop hooks → TaskCompleted hooks → TeammateIdle hooks, blocking error 재주입 지원 |
| `query/deps.ts` | 40 | 4 개의 I/O 의존성 주입 인터페이스: callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3,419 | API 전체 생명주기: 파라미터 구성 → stream 생성 → SSE 파싱 → content block 누적 → cost 계산 → non-streaming fallback → cache breakpoint 관리 |
| `cost-tracker.ts` | 323 | Session 수준 비용 누적: per-model usage 추적, session 지속화/복원, 형식화된 출력 |
