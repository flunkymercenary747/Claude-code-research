# 제 8 장：컨텍스트 관리

## 8.1 개요 및 위치

컨텍스트 관리는 Claude Code 아키텍처에서 가장 핵심적인 하위 시스템 중 하나다. 전형적인 코딩 세션은 수 시간 동안 지속되며, 수백 번의 도구 호출과 수십만 토큰의 대화 기록을 생성한다. 관리하지 않으면 컨텍스트 창은 20~30번의 상호작용 후 고갈되어 세션이 중단된다.

Claude Code의 컨텍스트 관리 시스템이 해결하는 핵심 문제는 다음과 같다: **제한된 컨텍스트 창(일반적으로 200K 토큰) 내에서 세션의 연속성과 정보의 완전성을 유지하면서, 사용자가 체감하는 정보 손실을 최소화하는 방법은 무엇인가?**

이 시스템은 `services/compact/` 디렉터리 아래의 11개 파일로 구성되며, 총 약 3,900줄의 TypeScript 코드로 이루어져 있다. 여기에 두 개의 핵심 유틸리티 모듈인 `utils/collapseReadSearch.ts`(1,109줄)와 `utils/toolResultStorage.ts`(1,040줄)가 보조한다. 이 하위 시스템 전체의 설계는 세 가지 핵심 원칙을 구현한다:

1. **점진적 성능 저하**(Graceful Degradation): 비용이 없는 마이크로 압축에서 손실 있는 전체 압축까지, 단계적으로 개입 강도를 높인다
2. **캐시 우선**(Cache-First): 모든 압축 결정은 prompt cache 보존을 최우선으로 고려한다
3. **안전성 보장**(Safety Invariants): tool_use/tool_result 페어는 분리 불가, 재귀 보호, 서킷 브레이커 메커니즘

## 8.2 이론적 기반

### 8.2.1 정보이론 관점: 손실 압축 vs 무손실 압축

컨텍스트 관리는 본질적으로 **정보 압축 문제**다. Claude Code의 다층 구조는 각기 다른 압축 전략에 대응한다:

- **무손실 압축**(Lossless): 마이크로 압축의 `cache_edits` 경로——API의 cache editing 메커니즘을 통해 오래된 도구 결과의 서버 측 캐시 복사본을 삭제하지만, 로컬 메시지 내용은 변경하지 않는다. 모델은 `[Old tool result content cleared]` 플레이스홀더를 보지만, 원본 데이터는 디스크에 저장된다(`toolResultStorage.ts`). 정보는 손실되지 않고, 핫 스토리지에서 콜드 스토리지로 이동할 뿐이다.
- **손실 압축**(Lossy): 전체 압축은 Fork Agent를 통해 요약을 생성하며, 수만 토큰의 대화를 수천 토큰으로 압축한다. 이는 되돌릴 수 없는 차원 축소 과정으로——코드 세부 사항, 에러 스택, 중간 추론 등이 손실될 수 있다.

Rate-Distortion Theory 관점에서, Claude Code의 설계는 암묵적인 **왜곡 측정 함수**를 내포한다: 요약 prompt의 9개 섹션(8.6절 참조)이 어느 정보 차원이 왜곡을 가장 적게 허용하는지 정의한다——"user messages"(완전 보존)는 "key technical concepts"(요약 허용)보다 우선순위가 높다.

### 8.2.2 캐시 이론: 시간적 지역성과 공간적 지역성

마이크로 압축의 화이트리스트 메커니즘은 고전 캐시 이론의 **시간적 지역성**(Temporal Locality) 가정을 구현한다:

> 최근 사용된 도구 결과는 이후 참조될 가능성이 더 높다.

`microCompact.ts`의 화이트리스트(`COMPACTABLE_TOOLS`)는 eviction policy의 구현이다——특정 도구(Read, Shell, Grep, Glob, WebFetch, WebSearch, Edit, Write)의 결과만 지울 수 있는데, 이들의 출력은 재생성 가능하기 때문이다(도구를 다시 실행하여 얻을 수 있다). 사용자가 수동으로 입력한 텍스트, 이미지 등 재생성 불가능한 내용은 절대 지워지지 않는다.

```typescript
// microCompact.ts:30-41 — 압축 가능한 도구 화이트리스트
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

`keepRecent` 파라미터(기본값: 최근 5개 보존)는 LRU(Least Recently Used) 축출 정책을 직접 구현한다.

### 8.2.3 서킷 브레이커 패턴(Circuit Breaker Pattern)

`autoCompact.ts`의 서킷 브레이커 메커니즘은 분산 시스템의 고전적인 Circuit Breaker Pattern을 LLM 애플리케이션에 정밀하게 적용한 것이다. 이 패턴은 Michael Nygard의 『Release It!』에서 유래했으며, 3상 모델(Closed → Open → Half-Open)이 Claude Code에서 다음과 같이 구현된다:

```typescript
// autoCompact.ts:70-73 — 서킷 브레이커 임계값
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

이 주석은 서킷 브레이커 도입 전의 실제 재앙적 데이터를 드러낸다: **1,279개의 세션이 50회 이상의 연속 실패 루프에 빠졌으며**, 가장 심각한 단일 세션은 3,272번의 실패를 기록했고, 전 세계적으로 매일 약 250K번의 API 호출이 낭비되었다. 서킷 브레이커 도입으로 최대 재시도 횟수가 3회로 제한되었다.

| 상태 | 동작 | 대응 코드 |
|------|------|---------|
| Closed(정상) | `consecutiveFailures < 3`, 정상적으로 압축 시도 | `autoCompactIfNeeded` 기본 경로 |
| Open(트립) | `consecutiveFailures >= 3`, 압축 건너뜀 | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open(탐지) | 압축 성공 후 `consecutiveFailures` 0으로 재설정 | 성공 시 `consecutiveFailures: 0` |

## 8.3 아키텍처 개요

### 8.3.1 다층 압축 체계의 전체 아키텍처

Claude Code의 컨텍스트 관리는 **5층 방어선** 설계를 채택한다. 낮은 개입에서 높은 개입 순서로:

```
┌─────────────────────────────────────────────────────────────────┐
│                        사용자 요청                                │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage(예방층)                             │
│   대형 도구 결과 → 디스크 영속화 + 2KB 미리보기                    │
│   트리거: 결과 > 임계값(기본 50K chars)                           │
│   비용: 컨텍스트 비용 없음(디스크 저장, preview만 컨텍스트에)        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact(마이크로 압축)                              │
│   경로 A: 시간 트리거 — 오래된 도구 결과 내용 지우기               │
│   경로 B: 캐시 편집 — cache_edits API로 서버 측 캐시 삭제         │
│   트리거: 매 API 호출 전                                         │
│   비용: 극히 낮음(도구 결과가 플레이스홀더로 대체, 디스크에서 복구 가능) │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact(자동 압축)                                 │
│   Session Memory → Fork Agent → 전체 요약                        │
│   트리거: 토큰이 effectiveContextWindow - 13K 초과               │
│   비용: 높음(손실 요약, 세부 정보 손실, API 호출 1회 소모)          │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact(수동 압축)                              │
│   사용자가 직접 트리거, Partial Compact 지원                       │
│   트리거: 사용자 명령                                             │
│   비용: 위와 동일                                                 │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact(반응형 압축)                           │
│   API가 prompt_too_long 반환 → 잘라내기 재시도                    │
│   트리거: 413 오류                                               │
│   비용: 최고(긴급 잘라내기 + 요약, 정보 손실 최대)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 각 층의 트리거 조건, 비용, 정보 손실 비교

| 층 | 트리거 조건 | 시점 | 지연 | 정보 손실 | API 비용 |
|------|---------|------|------|---------|---------|
| L0: Tool Result Storage | 단일 도구 결과 > 임계값 | 도구 실행 후 | 디스크 I/O | 없음(원본 디스크 저장) | 없음 |
| L1a: Time-based MC | 마지막 assistant 이후 60분 경과 | API 호출 전 | 없음(로컬 작업) | 낮음(오래된 결과 삭제) | 없음 |
| L1b: Cached MC | 압축 가능 도구 수 임계값 초과 | API 호출 전 | 없음(cache_edits) | 낮음(위와 동일) | 없음 |
| L2: Auto-Compact | 토큰 > 임계값 | 턴 사이 | 5-15초(API 호출) | 높음(손실 요약) | API 호출 1회 |
| L3: Manual Compact | 사용자 /compact | 사용자 트리거 | 위와 동일 | 중-높음(사용자 안내 가능) | API 호출 1회 |
| L4: Reactive Compact | prompt_too_long 413 | API 실패 후 | 10-30초(재시도) | 최고(잘라내기+요약) | API 호출 1-4회 |

### 8.3.3 데이터 흐름

```
메시지 배열 (Message[])
    │
    ▼
microcompactMessages()  ──→ [시간 트리거?] ──Y──→ 내용 삭제 → 반환
    │ N                      │
    │                  [캐시 편집?] ──Y──→ pendingCacheEdits → 반환
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     압축 없이 그대로 반환
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [session memory 있음?]
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
새 메시지 배열: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 제1층: 마이크로 압축(Microcompact)

마이크로 압축은 컨텍스트 관리의 첫 번째 방어선이다. **매 API 호출 전**에 실행되며(`microcompactMessages` 진입점), 최소한의 비용으로 컨텍스트 공간을 확보하는 것이 목표다.

### 8.4.1 압축 가능 도구 화이트리스트

마이크로 압축은 특정 도구의 출력에 대해서만 동작한다. 화이트리스트의 배후 설계 원칙은 **재생성 가능한 내용만 지운다**는 것이다.

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // 파일 읽기 — 다시 읽기 가능
  ...SHELL_TOOL_NAMES,     // Shell 명령 — 다시 실행 가능
  GREP_TOOL_NAME,          // 검색 — 다시 검색 가능
  GLOB_TOOL_NAME,          // 파일 매칭 — 다시 매칭 가능
  WEB_SEARCH_TOOL_NAME,    // 웹 검색 — 다시 검색 가능
  WEB_FETCH_TOOL_NAME,     // 웹 가져오기 — 다시 가져오기 가능
  FILE_EDIT_TOOL_NAME,     // 파일 편집 — 결과가 이미 디스크에 저장됨
  FILE_WRITE_TOOL_NAME,    // 파일 쓰기 — 위와 동일
])
```

`apiMicrocompact.ts`에는 더 세분화된 구분이 정의되어 있음을 주목하라:

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // tool_result 내용 삭제
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // tool_use 입력 삭제
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

이 구분은 매우 정교하다: Read/Grep/Shell의 경우 **출력**(tool_result)을 지우고; Edit/Write의 경우 **입력**(tool_use input)을 지운다. 편집 작업의 입력(diff 내용)은 크기가 크지만 결과는 이미 디스크에 영속화되어 있기 때문이다.

### 8.4.2 두 가지 하위 경로 상세

마이크로 압축에는 두 가지 상호 배타적인 실행 경로가 있으며, `microcompactMessages()` 함수가 통합 스케줄링을 담당한다:

```typescript
// microCompact.ts:287-317 — 스케줄링 로직
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // 경로 A: 시간 트리거 — 최우선순위, 이후 경로 단락
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // 경로 B: 캐시 편집 — 메인 스레드 전용, 특정 모델만 지원
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

**경로 A: Time-based Microcompact(시간 트리거)**

사용자가 세션을 떠난 후 설정된 시간 임계값(기본 60분)을 초과하여 돌아왔을 때 트리거된다. 설계 이유가 `timeBasedMCConfig.ts`에 명확히 기술되어 있다:

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

서버 측 prompt cache의 TTL은 1시간이다. 사용자가 1시간 이상 떠나 있으면 **캐시가 반드시 만료**되어 전체 prompt prefix를 다시 써야 한다. 이때 오래된 도구 결과를 지우는 것은 "무료"다——추가적인 캐시 미스 비용이 발생하지 않는다.

시간 트리거의 핵심 로직:

```typescript
// microCompact.ts:381-389 — 시간 트리거 평가
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

시간 트리거 후 삭제 전략도 LRU(`keepRecent` 기본값 5)를 사용하지만, 경계 보호가 있다:

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

이 `Math.max(1, ...)`는 `keepRecent=0`일 때 `slice(-0)`이 전체 배열을 반환하는 JavaScript 함정을 방지한다——전형적인 "방어적 프로그래밍으로 의미적 모호성 방지" 사례다.

시간 트리거 후에는 캐시 편집 상태도 재설정해야 한다:

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**경로 B: Cached Microcompact(캐시 편집)**

이것은 Anthropic 내부의 고급 최적화 경로(`feature('CACHED_MICROCOMPACT')`)로, API의 `cache_edits` 메커니즘을 활용하여 **로컬 메시지 내용을 수정하지 않고** 서버 측 캐시에서 도구 결과를 삭제한다.

```typescript
// microCompact.ts:327-370 — 캐시 편집 경로 핵심
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // 도구 결과 등록
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
      messages,  // 메시지 불변!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

핵심 설계 결정: **메시지 배열 불변**——`return { messages }`는 원본 참조를 반환한다. 캐시 편집은 API 층에서 일어나며(`cache_edits` 파라미터를 통해), 로컬 상태는 완전히 유지된다. 즉, API 호출이 실패하거나 재시도되어도 로컬 부작용이 없다.

### 8.4.3 캐시 편집의 상태 관리

캐시 편집 경로는 세 가지 핵심 상태를 유지한다:

```typescript
// microCompact.ts:43-49 — 모듈 수준 상태
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

이 세 상태의 생명주기 관리는 미묘하다:

- `pendingCacheEdits`는 일회성이다——`consumePendingCacheEdits()`가 읽은 후 지운다(`microCompact.ts:80-84`). 호출자는 API 요청에 전송한 후 반드시 pin해야 한다.
- `pinnedCacheEdits`는 누적된다——성공한 cache edit마다 특정 user message 위치에 pin되며, 이후 요청은 동일한 위치에서 다시 전송하여 캐시 히트를 보장해야 한다.
- `cachedMCState`는 압축 후(`resetMicrocompactState()`) 또는 시간 트리거 후 재설정된다.

```typescript
// microCompact.ts:78-105 — 상태 소비와 pin
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

### 8.4.4 토큰 추정 보조 함수

마이크로 압축 모듈은 시스템 전체가 공유하는 토큰 추정 함수를 제공한다:

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
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 고정 2000
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
  return Math.ceil(totalTokens * (4 / 3))  // 4/3 보수적 패딩
}
```

`roughTokenCountEstimation`의 핵심 공식은 매우 간결하다: `Math.round(content.length / 4)`(`tokenEstimation.ts:203-207`). 최종적으로 `estimateMessageTokens`는 이 결과에 4/3 보수 계수를 곱하며, 이는 `text.length / 3`과 동일하다. 이 이중 보수 전략은 과소 추정 가능성을 극히 낮춘다.

## 8.5 제2층: 자동 압축(Auto-Compact)

### 8.5.1 임계값 계산 공식

자동 압축의 트리거 임계값은 다음 공식으로 계산된다:

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

구체적인 수치 도출(Claude Opus 200K 기준):

```
contextWindow = 200,000
maxOutputTokens = 16,384 (또는 모델별 값)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (p99.99 = 17,387 기준)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — 핵심 상수
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

`AUTOCOMPACT_BUFFER_TOKENS = 13,000`의 선택은 공학적 트레이드오프다: 너무 작으면 압축이 너무 자주 발생하고(매 압축마다 5~15초와 API 호출 1회가 소모됨), 너무 크면 가용 컨텍스트가 낭비된다. 13K는 약 3~5번의 일반 대화에 해당하는 공간이다.

### 8.5.2 shouldAutoCompact 결정 트리

```typescript
// autoCompact.ts:127-178 — 완전한 결정 체인
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. 재귀 보호: session_memory 및 compact 쿼리 소스는 트리거하지 않음
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. 컨텍스트 폴딩 보호: marble_origami(ctx-agent)는 트리거하지 않음
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. 설정 확인: 사용자가 활성화했는지
  if (!isAutoCompactEnabled()) return false

  // 4. 반응형 모드: 활성화된 경우 능동적 압축 억제
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. 컨텍스트 폴딩 모드: 폴딩이 곧 컨텍스트 관리, 압축이 개입해서는 안 됨
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. 토큰 카운트 + 임계값 비교
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

이 결정 트리는 Claude Code가 병렬로 실험 중인 여러 컨텍스트 관리 전략을 드러낸다:
- **Reactive Compact**(`tengu_cobalt_raccoon`): 능동적으로 압축하지 않고, API가 prompt_too_long을 보고할 때까지 기다림
- **Context Collapse**(`CONTEXT_COLLAPSE`): 90% 제출 / 95% 차단의 스트리밍 방식으로 컨텍스트 관리
- **Auto Compact**(현재 기본값): 임계값에서 능동적으로 압축

세 가지는 상호 배타적이며, feature flag로 제어된다.

### 8.5.3 서킷 브레이커 메커니즘

```typescript
// autoCompact.ts:219-272 — 서킷 브레이커 포함 autoCompactIfNeeded
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // 서킷 브레이커 확인
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // 트립 상태, 직접 건너뜀
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // Session Memory 압축을 우선 시도
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 전통적 압축
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

### 8.5.4 autoCompactIfNeeded 실행 흐름

완전한 실행 순서:

1. **환경 변수 확인**: `DISABLE_COMPACT` → 전역 비활성화
2. **서킷 브레이커 확인**: `consecutiveFailures >= 3` → 건너뜀
3. **임계값 확인**: `shouldAutoCompact()` → 다층 게이트
4. **Session Memory 압축**(우선 경로): 기존 session memory를 활용하여 API 호출 대체
5. **전통적 Fork Agent 압축**(폴백 경로): 완전한 API 기반 요약 생성
6. **실패 처리**: 서킷 브레이커 카운터 증가, 다음 턴으로 전달

## 8.6 제3층: 전통적 압축(Full Compact)

### 8.6.1 Fork Agent 메커니즘

전통적 압축의 핵심은 Fork Agent를 통해 대화 요약을 생성하는 것이다. `streamCompactSummary()` 함수(`compact.ts:1136-1396`)는 두 단계의 폴백 전략을 구현한다:

**첫 번째 단계: Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — 캐시 공유 fork
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

Fork Agent는 메인 대화의 완전한 prompt cache(시스템 프롬프트 + 도구 + 컨텍스트 메시지)를 재사용하고, 요약 요청만 추가한다. 핵심 설계:

1. `maxTurns: 1` — 다중 턴 상호작용 불허
2. `canUseTool: createCompactCanUseTool()` — 모든 도구 호출 거부
3. `skipCacheWrite: true` — 캐시에 쓰지 않음(임시 분기)
4. **maxOutputTokens 미설정** — 주석 설명: 설정하면 thinking config가 변경되어 cache key가 불일치함

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**두 번째 단계: Streaming Fallback**

Fork Agent가 실패하면 직접 스트리밍 API 호출로 폴백하며, 이때 **`maxOutputTokensOverride`를 설정할 수 있다**:

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

스트리밍 폴백은 설정 기반 재시도도 지원한다(`tengu_compact_streaming_retry`), 최대 `MAX_COMPACT_STREAMING_RETRIES = 2`회.

### 8.6.2 전처리 파이프라인

압축 전 메시지는 세 단계의 전처리를 거친다:

```typescript
// compact.ts:1293-1300 — 전처리 체인
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

1. `getMessagesAfterCompactBoundary` — 마지막 압축 이후의 메시지만 가져옴
2. `stripReinjectedAttachments` — `skill_discovery` / `skill_listing` 첨부 파일 제거(압축 후 재주입됨)
3. `stripImagesFromMessages` — 이미지 블록을 `[image]` 텍스트 마커로 교체(`compact.ts:144-199`)

`stripImagesFromMessages`가 존재하는 이유는 실용적이다:

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

CCD(Claude Code Desktop) 사용자는 스크린샷을 자주 첨부하는데, 이미지를 제거하지 않으면 압축 API 호출 자체가 prompt 길이 초과로 실패할 수 있다.

### 8.6.3 요약 출력의 9개 섹션 형식

`prompt.ts`는 요약이 따라야 할 9개의 구조화된 섹션을 정의한다:

```
1. Primary Request and Intent    — 사용자 의도
2. Key Technical Concepts        — 기술 개념
3. Files and Code Sections       — 파일 및 코드 조각
4. Errors and fixes              — 에러 및 수정
5. Problem Solving               — 문제 해결
6. All user messages             — 모든 사용자 메시지(tool result 제외)
7. Pending Tasks                 — 대기 중인 작업
8. Current Work                  — 현재 작업
9. Optional Next Step            — 다음 단계(선택)
```

섹션 6의 설계가 특히 중요하다——"List ALL user messages that are not tool results". 이는 대화가 압축되더라도 사용자의 원본 표현이 완전히 보존됨을 보장한다. 이는 **사용자 피드백 정보 무손실**의 보장이다.

섹션 9에는 정교하게 설계된 제약이 있다:

```
// prompt.ts — 섹션 9의 제약
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

이는 압축 후 모델이 "제멋대로 행동하는 것"을 방지한다——사용자가 명시적으로 요청한 다음 단계만 기록된다.

### 8.6.4 NO_TOOLS_PREAMBLE 우회 방지 설계

Fork Agent는 cache key 매칭을 위해 메인 대화의 완전한 도구 세트를 상속받지만, 압축 agent는 어떤 도구도 사용해서는 안 된다. 이는 모순을 형성한다: 도구가 스키마에 존재하지만 호출해서는 안 된다.

해결책은 **세 겹의 도구 거부**다:

```typescript
// prompt.ts:16-24 — 첫 번째 층: prompt 시작 부분의 강력한 선언
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — 두 번째 층: prompt 끝부분의 반복 알림
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — 세 번째 층: 코드 수준 거부
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

주석은 이 세 겹 설계의 실제 이유를 드러낸다:

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

Sonnet 4.6에서는 prompt 지시만으로는 2.79%의 확률로 여전히 도구 호출을 시도한다(4.5에서는 0.01%에 불과). `createCompactCanUseTool`은 마지막 코드 수준 보장이다.

### 8.6.5 후처리(formatCompactSummary)

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // <analysis> 초안 영역 제거 — 요약 품질을 높이는 중간 추론, 보존 불필요
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // <summary> 내용 추출
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

`<analysis>` 태그 설계는 Chain-of-Thought 기법이다: 모델이 분석 영역에서 먼저 "초안을 작성"한 후 `<summary>`에 최종 결과를 출력하게 한다. 분석 영역의 존재는 요약 품질을 높이지만, 최종 출력에서는 제거된다——중복적인 중간 추론을 포함하여 이후 턴의 컨텍스트 공간을 낭비하기 때문이다.

### 8.6.6 압축 후 메시지 시퀀스와 첨부 파일 재주입

압축이 완료되면 새 메시지 시퀀스가 `buildPostCompactMessages()`에 의해 구축된다:

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // 시스템 메시지: 압축 경계 표시
    ...result.summaryMessages,    // 사용자 메시지: 요약 내용
    ...(result.messagesToKeep ?? []),  // 보존된 원본 메시지
    ...result.attachments,        // 파일 첨부 + 스킬 + 계획
    ...result.hookResults,        // SessionStart 훅 결과
  ]
}
```

첨부 파일 재주입은 복잡한 과정이다(`compact.ts:532-585`):

1. **파일 첨부**: 최근 접근한 최대 5개 파일, 50K 토큰 예산 제약, 각 파일 최대 5K 토큰
2. **계획 파일**: 활성 계획이 있는 경우
3. **계획 모드 지시사항**: plan mode에 있는 경우
4. **스킬 내용**: 호출된 스킬의 내용, 최근 사용순 정렬, 각 최대 5K 토큰, 총 예산 25K 토큰
5. **Deferred Tools Delta**: 지연 로드 도구의 스키마 재선언
6. **Agent Listing Delta**: agent 목록 재선언
7. **MCP Instructions Delta**: MCP 서버 지시사항 재선언

### 8.6.7 PTL 재시도 메커니즘(Prompt-Too-Long Recovery)

압축 API 호출 자체가 prompt 길이 초과로 실패하면, 시스템은 점진적 잘라내기 재시도를 수행한다:

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 이전 재시도에서 남긴 마커 메시지 먼저 지우기
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // 정밀 잘라내기: API가 반환한 토큰 차이에 따라
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 모호 잘라내기: 메시지 그룹의 20%를 버림
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // 최소 한 그룹 보존
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

재시도 상한은 `MAX_PTL_RETRIES = 3`이다. 잘라내기 전략에는 두 가지 경로가 있다:
- **정밀 경로**: API 오류에 토큰 차이가 포함된 경우 → 차이에 따라 그룹별 버림
- **모호 경로**(Vertex/Bedrock 등 비표준 오류 형식): 매번 20% 버림

283행의 경계 처리를 주목하라: 그룹 0을 버린 후 메시지 시퀀스가 assistant 메시지로 시작하면 API 제약을 위반한다(첫 메시지는 반드시 user여야 함). 시스템은 합성 user 마커 메시지를 삽입하여 수정한다.

### 8.6.8 부분 압축(Partial Compact)의 두 방향

`partialCompactConversation()`(`compact.ts:772-1106`)은 두 방향을 지원한다:

```
Direction 'from': 
  [압축 후 보존] | pivot | [요약된 메시지]
  → prompt cache 보존(보존된 것이 앞에, cache prefix 불변)

Direction 'up_to':
  [요약된 메시지] | pivot | [압축 후 보존]
  → prompt cache 무효화(요약이 앞에, prefix 변경)
```

`up_to` 방향에는 추가 정리 로직이 있다——보존된 메시지에서 오래된 compact boundary와 summary를 제거해야 한다:

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

주석은 이유를 설명한다: `up_to` 모드에서 요약이 보존된 메시지 앞에 있어 오래된 boundary가 `findLastCompactBoundaryIndex`의 역방향 스캔을 오도할 수 있다.

## 8.7 제4층: Session Memory 압축

### 8.7.1 핵심 아이디어와 장점

Session Memory 압축(`sessionMemoryCompact.ts`)은 전통적 압축의 최적화 대안이다. 핵심 아이디어: 백그라운드에서 지속적으로 추출되는 session memory(대화의 증분 요약)를 활용하여 실시간 생성되는 Fork Agent 요약을 대체한다.

장점:
- **추가 API 호출 없음**: session memory는 백그라운드 agent가 지속적으로 유지하며, 압축 시 직접 사용
- **더 낮은 지연**: API 응답을 위한 5~15초 대기 불필요
- **더 세밀한 보존**: 최근 메시지를 정확히 얼마나 보존할지 계산 가능

### 8.7.2 calculateMessagesToKeepIndex 알고리즘 상세

이것은 Session Memory 압축의 핵심 알고리즘(`sessionMemoryCompact.ts:262-327`)으로, 압축 후 몇 개의 메시지를 보존할지 결정한다:

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // lastSummarizedIndex + 1부터 시작(session memory가 이전 내용을 이미 다룸)
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // 현재 보존 범위의 토큰과 텍스트 메시지 수 계산
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // 이미 상한에 도달 → 확장 안 함
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 이미 두 가지 최소 요건 충족 → 확장 안 함
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 앞으로 확장하되, 마지막 compact boundary를 넘지 않음
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

설정 파라미터(GrowthBook 원격 설정으로 재정의 가능):

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // 최소 10K 토큰 보존
  minTextBlockMessages: 5,     // 최소 텍스트 있는 메시지 5개 보존
  maxTokens: 40_000,           // 최대 40K 토큰 보존
}
```

알고리즘의 이중 제약 설계(`minTokens` AND `minTextBlockMessages`)는 다음을 보장한다:
- 소수의 초대형 메시지만으로 확장을 멈추지 않음(토큰은 충족했지만 메시지 수 부족)
- 작은 메시지가 너무 많지만 실제 토큰이 부족한 상황을 방지

**Floor 메커니즘**: 앞으로 확장 시 마지막 compact boundary를 넘어서는 안 된다(`floor = lastBoundaryIndex + 1`). 주석이 이유를 설명한다:

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

디스크 저장 층의 메시지 체인은 compact boundary에서 불연속이 있어, 그것을 넘으면 로더의 역방향 순회가 보존된 메시지를 건너뛰게 된다.

### 8.7.3 adjustIndexToPreserveAPIInvariants의 버그 수정

이 함수(`sessionMemoryCompact.ts:172-260`)는 전체 압축 시스템에서 가장 정교한 코드로, 두 가지 API 불변량 문제를 해결한다:

**버그 시나리오 1: 고아 tool_result**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

startIndex = N+2인 경우:
  구 코드는 메시지 N+2의 tool_results만 확인 → 찾지 못함 → N+2 반환
  normalizeMessagesForAPI가 message.id로 병합 후:
    msg[1]: assistant with [tool_use: VALID_ID]  (ORPHAN tool_use 제외!)
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → API 오류: orphan tool_result가 존재하지 않는 tool_use를 참조
```

**버그 시나리오 2: thinking 블록 누락**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

startIndex = N+1인 경우:
  thinking 블록이 N에서 제외됨
  normalizeMessagesForAPI가 병합 불가(동일 ID 메시지 없음)
  → thinking 블록 영구 손실
```

수정 코드는 두 단계 조정을 수행한다:

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // 1단계: tool_use/tool_result 페어 처리
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... 보존 범위 내의 기존 tool_use ID 수집
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // 누락된 tool_use를 앞으로 검색
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 찾은 ID 삭제
      }
    }
  }

  // 2단계: 공유 message.id를 가진 thinking 블록 처리
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

이 코드의 핵심 통찰: Claude API의 스트리밍 응답은 동일한 API 응답을 여러 assistant 메시지로 분할한다(동일한 `message.id`를 공유하지만 UUID는 다름). thinking 블록과 tool_use 블록은 분리되어 있다. `normalizeMessagesForAPI`는 `message.id`로 이 메시지들을 병합하는데——압축이 동일 ID의 메시지 그룹을 잘라내면 병합 후 불일치가 발생한다.

### 8.7.4 trySessionMemoryCompaction 전체 흐름

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. 게이트 확인
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. 원격 설정 초기화(첫 번째만)
  await initSessionMemoryCompactConfig()

  // 3. 진행 중인 session memory 추출 완료 대기
  await waitForSessionMemoryExtraction()

  // 4. session memory 내용 가져오기
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. 경계 결정
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID 없음 → 폴백
  } else {
    // 재개된 세션: 경계 없음 → 끝에서 시작
    lastSummarizedIndex = messages.length - 1
  }

  // 6. 보존 범위 계산
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // 오래된 boundary 필터링

  // 7. session start 훅 실행
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. 결과 구축
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. 임계값 확인(autocompact만)
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // 압축 후에도 임계값 초과 → 전통적 압축으로 폴백
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 설정 파라미터(GrowthBook 원격 설정)

Session Memory 압축의 모든 핵심 파라미터는 GrowthBook 원격 설정으로 재정의할 수 있다:

```typescript
// sessionMemoryCompact.ts:91-109 — 원격 설정 초기화
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // 방어적 코딩: 양수 값만 사용, 0 값은 무시
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

기능 게이트는 두 개의 독립적인 feature flag로 제어된다:

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

## 8.8 컨텍스트 폴딩과 도구 결과 저장

### 8.8.1 collapseReadSearch 메커니즘

`utils/collapseReadSearch.ts`(1,109줄)는 UI 층의 메시지 폴딩을 구현한다——연속된 검색/읽기 작업을 단일 줄 요약으로 접는다(예: "Read 5 files, searched 3 patterns").

핵심 분류 로직(`getToolSearchOrReadInfo`, `collapseReadSearch.ts:142-237`)은 도구 호출을 다음과 같이 분류한다:

| 분류 | 폴딩 가능 | 폴딩 동작 |
|------|-------|---------|
| Read (file_path) | 예 | "Read N files" |
| Search (Grep/Glob) | 예 | "Searched N patterns" |
| Shell (Bash) | 전체 화면 모드에서 예 | "Ran N bash commands" |
| REPL | 예(묵음 흡수) | 내부 도구 독립 계산 |
| Memory Write | 예 | 특수 마커 |
| ToolSearch | 예(묵음 흡수) | 카운터 증가 없음 |
| Edit/Write (memory 제외) | 아니오 | 폴딩 그룹 끊음 |

"묵음 흡수"(`isAbsorbedSilently`)는 정교한 설계다: REPL과 ToolSearch는 카운터를 증가시키지 않지만 현재 폴딩 그룹도 끊지 않는다. 즉, `[Read, ToolSearch, Read]`는 "Read 2 files"로 폴딩되고 ToolSearch에 의해 두 그룹으로 나뉘지 않는다.

폴딩은 **UI 층만의 최적화**다——API에 전송되는 메시지 내용을 변경하지 않고 터미널 표시에만 영향을 미친다.

### 8.8.2 toolResultStorage의 디스크 저장 전략

`utils/toolResultStorage.ts`(1,040줄)는 컨텍스트 관리의 "제0층"이다——도구 결과가 대화 기록에 들어가기 전에 초대형 결과를 처리한다.

**영속화 임계값 분석**:

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Read 도구 특수: Infinity → 영속화하지 않음(Read 자체에 maxTokens 제한 있음)
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // GrowthBook 재정의(tengu_satin_quoll)
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // 기본값: min(도구 선언값, 전역 50K 기본값)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**중복 제거 최적화**: `tool_use_id`는 고유하므로 `flag: 'wx'`(독점 쓰기)를 사용하여 중복 쓰기를 방지한다:

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: 이전 턴에서 이미 영속화됨, 건너뜀
}
```

**빈 결과 처리**:

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

이 수정은 모델 동작 버그를 해결한다: 빈 tool_result가 일부 모델로 하여금 `\n\nHuman:` 패턴을 대화 종료로 매칭하게 만든다.

**Per-Message Aggregate Budget**(`enforceToolResultBudget`, `toolResultStorage.ts:769-908`):

이것은 `toolResultStorage.ts`에서 가장 복잡한 기능으로——API 수준의 user 메시지(`normalizeMessagesForAPI` 병합 후)별로 총 도구 결과 크기 예산을 강제한다.

설계 요점:
- **상태 동결**(`ContentReplacementState`): 한번 tool_result가 "보여진" 후에는(모델에 전송됨), 그 결정(교체/미교체)이 동결되어 생명주기 내에서 절대 변경되지 않는다——이는 prompt cache의 안정성을 보장한다
- **3분할** 전략: `mustReapply`(이전에 교체됨 → 캐시된 교체 내용 재적용), `frozen`(이전에 봤지만 교체 안 됨 → 변경 없음), `fresh`(새로운 것 → 교체 가능)
- **최대 우선**: 교체가 필요할 때, 가장 큰 fresh 결과부터 우선 교체

## 8.9 5층 오류 복구에서 압축의 역할

### 8.9.1 완전한 5층 오류 복구 메커니즘

압축 시스템은 Claude Code의 오류 복구 메커니즘에서 여러 역할을 한다:

| 층 | 트리거 조건 | 압축 동작 | 출처 |
|------|---------|---------|------|
| L1 | API가 prompt_too_long(413) 반환 | Reactive Compact: 잘라내기 + 재요약 | `compactMessages.ts` |
| L2 | 압축 API 자체가 413 반환 | PTL Retry: 가장 오래된 메시지 그룹 잘라내기 × 3회 | `compact.ts:truncateHeadForPTLRetry` |
| L3 | 압축 후에도 임계값 초과 | Re-compaction: 자동으로 다시 압축 | `autoCompact.ts:recompactionInfo` |
| L4 | 연속 3회 압축 실패 | Circuit Breaker: 시도 중단 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent 텍스트 출력 없음 | Streaming Fallback: 직접 스트리밍 API 호출 | `compact.ts:streamCompactSummary` |

### 8.9.2 반응형 압축 vs 능동적 압축

두 전략의 트레이드오프:

**능동적 압축**(Auto-Compact, 현재 기본값):
- 토큰이 임계값에 도달할 때 능동적으로 트리거
- 장점: 사용자 경험이 더 부드럽고, 413 오류 미발생
- 단점: 너무 일찍 압축할 수 있어 사용 가능한 컨텍스트를 낭비

**반응형 압축**(Reactive Compact, `tengu_cobalt_raccoon` 실험):
- API가 prompt_too_long을 보고할 때까지 기다림
- 장점: 컨텍스트 활용률 최대화
- 단점: 사용자 경험에 명확한 중단 있음, 재시도 대기 필요

코드에서 두 가지의 상호 배타적 관계를 볼 수 있다:

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // 반응형 모드에서는 능동적으로 압축하지 않음
  }
}
```

## 8.10 메시지 그룹화와 토큰 추정

### 8.10.1 groupMessagesByApiRound 알고리즘

`grouping.ts`(63줄)는 메시지를 API 라운드별로 그룹화한다——각 그룹은 완전한 API 왕복에 대응한다:

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

그룹 경계의 유일한 판단 기준은 `message.id`의 변화다——동일한 API 응답의 여러 스트리밍 블록은 동일한 `message.id`를 공유하므로 자연스럽게 같은 그룹에 속한다.

이 설계는 이전의 "인간 턴" 기반 그룹화(실제 사용자 메시지에서만 그룹화)를 대체한다. 이전 방식은 SDK/CCR/eval 시나리오에서의 긴 단일 턴 agent 세션을 처리할 수 없었다.

### 8.10.2 roughTokenCountEstimation과 보수적 패딩

토큰 추정은 두 단계의 보수적 전략을 채택한다:

**1단계**: 기본 추정

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

기본값 4 bytes/token, JSON 파일은 2 bytes/token(JSON에 `{`, `}`, `:`, `,` 등 단일 문자 토큰이 많기 때문).

**2단계**: 메시지 수준 패딩

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

조합 효과: 일반 텍스트의 경우 실효 추정은 `text.length / 4 * 4/3 = text.length / 3`.

### 8.10.3 정밀 vs 추정의 혼합 전략

시스템은 서로 다른 시나리오에서 다른 정밀도를 사용한다:

| 시나리오 | 정밀도 | 출처 | 지연 |
|------|------|------|------|
| shouldAutoCompact | 혼합: API 반환 정밀값 우선 | `tokenCountWithEstimation` | 0(이미 캐시됨) |
| estimateMessageTokens | 거친 추정(`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | 거친 추정 | `estimateMessageTokens` | 0 |
| 압축 후 토큰 통계 | 정밀 | `tokenCountFromLastAPIResponse` | 0(API 이미 반환함) |

`tokenCountWithEstimation`의 혼합 전략: 최근 API 응답의 `usage.input_tokens`(정밀값)를 우선 사용하고, 없으면(첫 요청 전 등) 추정으로 폴백한다.

## 8.11 설계 결정 분석

### 8.11.1 점진적 성능 저하 철학

Claude Code의 컨텍스트 관리는 **단계를 건너뛰지 않는** 점진적 성능 저하를 채택한다: 각 층은 최소한의 비용으로 문제를 해결하려 하고, 현재 층이 실패할 때만 다음 층으로 올라간다. 이는 일반적인 "과잉 반응" 문제를 방지한다——예를 들어 큰 파일 Read 결과 하나만으로 전체 압축을 트리거하는 것.

업계 관행과 비교:
- **ChatGPT**: 오래된 메시지 잘라내기(단순하지만 조잡함)
- **GitHub Copilot Chat**: 고정 컨텍스트 창 + 최근 N개 메시지(압축 없음)
- **Claude Code**: 5층 점진(예방 → 미세 조정 → 요약 → 긴급 복구)

### 8.11.2 캐시 우선 설계

Prompt cache는 Claude Code의 생명선이다——200K 토큰 요청에서 180K가 cache read($0.30/M 토큰)이면, 전부 cache miss($3/M 토큰)보다 비용이 10배 낮다. 거의 모든 설계 결정이 이 경제적 제약을 따른다:

1. **Fork Agent 캐시 prefix 공유**: 압축 API 호출이 메인 대화의 캐시를 재사용
2. **Fork에서 maxOutputTokens 미설정**: thinking config 불일치로 인한 cache miss 방지
3. **Cached MC가 로컬 메시지를 수정하지 않음**: prompt prefix 불변 유지
4. **ContentReplacementState가 이미 본 ID 동결**: 동일 tool_result의 교체 결정이 생명주기 내에서 불변
5. **sentSkillNames 미재설정**: ~4K 토큰의 skill_listing 재주입 방지
6. **pinnedCacheEdits를 고정 위치에서 재전송**: cache edit 위치 일관성 보장

### 8.11.3 안전성 보장

시스템은 세 가지 불변량을 유지한다:

**페어 분리 불가**: `adjustIndexToPreserveAPIInvariants`는 tool_use와 tool_result가 절대 양측으로 분리되지 않도록 보장한다. 이것은 기능적 정확성의 요건(API가 오류를 반환함)일 뿐만 아니라 의미적 정확성의 요건이기도 하다(모델은 이전에 호출한 도구의 결과를 봐야 함).

**재귀 보호**: `shouldAutoCompact`의 `querySource` 확인은 session_memory agent, compact agent, context collapse agent가 자동 압축을 트리거하지 않도록 보장한다——이 agent들 자체가 컨텍스트 관리의 일부이므로, 재귀 압축은 교착 상태를 초래할 것이다.

**서킷 브레이커 메커니즘**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`은 실제 데이터(1,279개 세션의 실패 루프)를 기반으로 설정되어, 무한 재시도를 유한 재시도 + 트립으로 바꿨다.

### 8.11.4 API-native Context Management와의 비교

`apiMicrocompact.ts`는 Claude Code가 일부 컨텍스트 관리를 API 층에 오프로드하는 방향을 탐색하고 있음을 드러낸다:

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

이 `context_management.edits` 전략들은 API 요청에서 직접 선언되며, 서버 측에서 실행된다. 장점은 지연이 더 낮고(클라이언트 처리 불필요) 서버 측 토큰 계산과 정확히 일치한다는 것이다. 현재 도구 지우기 전략은 내부 사용자(`USER_TYPE === 'ant'`)에게만 공개되며, 외부 사용자는 thinking 지우기만 사용한다.

## 8.12 이식 가능한 패턴

### 8.12.1 다층 압축 체계의 범용 설계 패턴

Claude Code의 컨텍스트 관리에서 추출한 이식 가능한 범용 패턴:

**패턴 1: 계층적 축출(Tiered Eviction)**
- 다른 유형의 내용에 다른 축출 전략 적용
- 재생성 가능한 내용(도구 출력) 우선 축출, 재생성 불가능한 내용(사용자 입력) 마지막 축출
- 구현 방법: 화이트리스트 + 우선순위 정렬

**패턴 2: 추정-정밀 혼합(Hybrid Estimation)**
- 빠른 결정에는 거친 추정(`text.length / 3`), 정밀 계산에는 API 반환값 사용
- 거친 추정은 항상 보수적(과소 추정으로 API 오류가 나는 것보다 과대 추정으로 조기 압축이 나은 편)

**패턴 3: 동결-재생(Freeze-Replay)**
- 내용이 모델에 "보여진" 후, 처리 결정이 동결됨
- 이후 턴에서 동결된 내용에 대해서는 "재생"(캐시된 교체 재적용)만 하고 새 결정 없음
- prompt prefix의 비트 수준 안정성 보장 → 캐시 히트

**패턴 4: 경계 인식 잘라내기(Boundary-Aware Truncation)**
- 의미 단위 중간에서 절대 잘라내지 않음(tool_use/tool_result 쌍, 동일 ID 메시지 그룹)
- 잘라내기 후 능동적 수정(합성 메시지 삽입, 인덱스 조정)

**패턴 5: 서킷 브레이커 보호(Circuit Breaker Protection)**
- 무한 재시도 가능성이 있는 작업에 실패 카운터 설정
- 직관이 아닌 실제 운영 데이터를 기반으로 임계값 설정

### 8.12.2 Doramagic에서 참고할 수 있는 점

Doramagic의 Soul Extractor 파이프라인에서 추출 과정은 대량의 중간 결과(코드 조각, API 문서, 커뮤니티 토론)를 생성할 수 있다. 참고할 수 있는 패턴:

1. **계층적 추출 캐시**: microcompact의 화이트리스트 메커니즘과 유사하게, 중간 API 응답과 코드 분석 결과를 재생성 가능성에 따라 분류하여 재획득 가능한 내용을 우선 축출
2. **증분 요약**: Session Memory Compact와 유사하게, 전체 기록 대신 추출된 지식의 증분 요약 유지
3. **동결 결정**: 지식 청크가 "가치 있음" 또는 "가치 없음"으로 확인되면 결정이 돌이킬 수 없음——다른 추출 라운드 간에 반복적인 재평가 방지

## 8.13 소스 코드 인덱스

| 파일 | 줄수 | 핵심 역할 |
|------|------|---------|
| `services/compact/compact.ts` | ~1,705 | 전통적 압축 주 로직: Fork Agent, PTL 재시도, 첨부 파일 재주입, 부분 압축 |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Session Memory 압축: calculateMessagesToKeepIndex, adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | 마이크로 압축: 시간 트리거, 캐시 편집, 토큰 추정 |
| `services/compact/prompt.ts` | ~374 | 압축 프롬프트: 9섹션 템플릿, NO_TOOLS_PREAMBLE, formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | 자동 압축: 임계값 계산, shouldAutoCompact 결정 체인, 서킷 브레이커 |
| `services/compact/apiMicrocompact.ts` | ~153 | API-native 컨텍스트 관리: clear_tool_uses, clear_thinking |
| `services/compact/grouping.ts` | ~63 | 메시지 그룹화: groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | 압축 후 정리: 캐시, 모듈 상태, 분류기 재설정 |
| `services/compact/timeBasedMCConfig.ts` | ~43 | 시간 트리거 설정: GrowthBook 원격 설정 |
| `services/compact/compactWarningHook.ts` | ~16 | React hook: compact 경고 억제 상태 구독 |
| `services/compact/compactWarningState.ts` | ~18 | 상태 저장: compact 경고 억제 플래그 |
| `services/cost-tracker.ts` | ~323 | 비용 추적: 토큰 청구, 모델 사용 통계 |
| `utils/collapseReadSearch.ts` | ~1,109 | 컨텍스트 폴딩: UI 층 메시지 그룹화 및 폴딩 |
| `utils/toolResultStorage.ts` | ~1,040 | 도구 결과 저장: 디스크 영속화, per-message 예산, ContentReplacementState |
| `services/tokenEstimation.ts` | ~350+ | 토큰 추정: roughTokenCountEstimation(text.length/4) |
