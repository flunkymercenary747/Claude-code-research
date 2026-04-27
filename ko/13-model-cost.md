# 제 13 장: 모델 선택과 비용 제어

> 데이터 소스: Claude Code TypeScript 소스 코드 스냅샷 (2026-03-31, ~512K LOC)
> 핵심 파일: `services/api/claude.ts` (3,419줄), `services/api/withRetry.ts`, `cost-tracker.ts` (323줄), `utils/effort.ts`, `utils/modelCost.ts`, `utils/model/model.ts`, `migrations/` 디렉토리 (11개 파일)

---

## 13.1 개요 및 위상

Claude Code의 모델 선택과 비용 제어에 관한 설계 철학은 세 문장으로 요약할 수 있습니다:

1. **사용자 의도 우선**: 우선순위 체인은 `/model` 명령 → `--model` 플래그 → 환경 변수 → 설정 파일 순으로 내려가며, 각 계층은 상위 계층에 의해 재정의될 수 있지만, 하위 계층에 의해 예기치 않게 대체되지 않습니다.
2. **비용 완전 투명**: 세션 종료 시 모델별 토큰 사용량과 달러 비용을 강제로 출력하며, 비활성화할 수 없습니다 (`hasConsoleBillingAccess()`가 true일 때만).
3. **무비밀 강등**: Overload Fallback (Opus → Sonnet) 발생 시 반드시 사용자에게 경고 메시지를 표시하며, 조용히 전환하지 않습니다.

이 챕터는 소스 코드 수준에서 이 하위 시스템에 대한 cc-notebook의 주장을 하나씩 검증하고 분석을 심화합니다.

---

## 13.2 이론적 기초

### 다중 모델 시스템의 라우팅 전략

다중 모델 시스템에서 라우팅 전략은 일반적으로 **능력** (capability), **비용** (cost), **지연** (latency) 세 가지 차원에서 균형을 맞춥니다. Claude Code의 선택은 메인 대화(main loop)를 가장 강력한 가용 모델로 라우팅하고, 백그라운드 보조 작업을 가장 빠르고 저렴한 모델로 라우팅하며, 메인 모델 불가 시 투명한 강등을 제공합니다.

### AI 시스템에서의 비용 효율 분석

`modelCost.ts`를 보면 Claude Code에 정확한 가격표가 내장되어 있음을 알 수 있습니다:

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

Haiku 4.5의 가격이 가장 낮고 ($1/$5 per Mtok), Opus 4.6 Fast Mode의 가격이 가장 높습니다 ($30/$150 per Mtok). 둘 사이의 가격 차이는 30배로, 시스템이 백그라운드 작업을 Haiku에 할당하는 핵심 경제적 논리입니다.

### 우아한 강등 (Graceful Degradation) 패턴

전통적인 소프트웨어에서 우아한 강등은 기능이 불가할 때 차선을 택하지만 충돌하지 않는 것입니다. LLM 시스템에서 대안은 "더 저렴하거나 더 가용한 모델로 전환"입니다. Claude Code는 카운터 보호가 있는 트리거 메커니즘을 구현했습니다: 연속 3회의 529 오류 후 모델 전환이 트리거되며, 즉시 전환하지 않습니다 (일시적 overload로 인한 불필요한 품질 강등 방지).

---

## 13.3 모델 선택 아키텍처

### 모델 우선순위 계층

`utils/model/model.ts`의 `getUserSpecifiedModelSetting()` 함수가 우선순위 순서를 정확히 정의합니다:

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

`getMainLoopModel()`은 여기에 5번째 우선순위인 내장 기본값을 추가합니다:

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

완전한 5단계 우선순위 체인:
| 우선순위 | 출처 | 설명 |
|--------|------|------|
| 1 (최고) | `/model` 명령 | 세션 내 즉시 적용, 메모리 override에 저장 |
| 2 | `--model` 시작 플래그 | 시작 시 메모리 override에 씀 |
| 3 | `ANTHROPIC_MODEL` 환경 변수 | 프로세스 수준 |
| 4 | `settings.json` 설정 파일 | 지속 사용자 선호 |
| 5 (최저) | 내장 기본값 | 구독 유형에 따라 결정 |

### 구독 유형별 기본 모델 계층화

`getDefaultMainLoopModelSetting()`은 구독 차이를 드러냅니다:

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (내부 직원) 기본 Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max 사용자와 Team Premium 사용자 기본 Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG, Enterprise, Team Standard, Pro 기본 Sonnet 4.6
  return getDefaultSonnetModel()
}
```

이 설계는 다음을 의미합니다: 사용자가 아무것도 설정하지 않아도, Max/Team Premium 사용자가 여는 것은 Opus 4.6이고, Pro/Sonnet 사용자가 여는 것은 Sonnet 4.6입니다. **기본값 자체가 제품 차별화 전략입니다.**

### 모델 Alias 시스템

`parseUserSpecifiedModel()`은 단축 별칭 파싱을 지원하여 사용자가 전체 Model ID를 기억할 필요가 없게 합니다:

```typescript
// utils/model/model.ts — parseUserSpecifiedModel 발췌
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan 모드는 Sonnet, 비-plan 모드는 Opus
```

`[1m]` 접미사는 임의의 alias에 붙일 수 있으며 (예: `opus[1m]`), 시스템이 자동으로 1M context window 변형으로 파싱합니다.

### 모델 능력 탐지

`utils/model/modelCapabilities.ts`는 내부 직원(`USER_TYPE === 'ant'`)에게만 활성화되는 캐싱 메커니즘을 구현합니다:

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

외부 사용자는 모델 능력 목록을 요청하지 않으며, 능력 정보는 `modelSupportsEffort()`, `modelSupports1M()` 등의 함수에 하드코딩되어 추가 API 호출 오버헤드를 방지합니다.

---

## 13.4 Haiku의 백그라운드 용도

cc-notebook은 Haiku에 6가지 백그라운드 용도가 있다고 주장합니다. `queryHaiku` 함수 호출 위치(`grep -rn 'queryHaiku\b'`)와 `getSmallFastModel()` 호출 위치에 대한 전수 검색을 통해 **소스 코드 검증**한 결과는 다음과 같습니다:

### 백그라운드 용도 요약 (소스 코드 검증)

| 번호 | 용도 | 파일 | 트리거 조건 |
|------|------|------|-----------|
| 1 | Web Fetch 내용 추출 | `tools/WebFetchTool/utils.ts:503` | 웹 페이지 크롤링 후 Haiku를 사용해 Markdown을 사용자가 지정한 내용으로 필터링 |
| 2 | Shell 명령 접두사 추출 | `utils/shell/prefix.ts:220` | Bash 도구 실행 전, Haiku를 사용해 명령에 권한 프롬프트가 필요한지 판단 |
| 3 | 세션 제목 생성 | `utils/sessionTitle.ts:87` | 세션 시작 후 자동으로 짧은 제목 생성 (JSON schema 출력) |
| 4 | MCP DateTime 파싱 | `utils/mcp/dateTimeParser.ts:68` | 자연어 시간 설명을 ISO 8601 형식으로 파싱 |
| 5 | 도구 호출 요약 생성 | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | 배치 도구 호출 완료 후 한 줄 요약 레이블 생성 |
| 6 | 세션 이름 변경 | `commands/rename/generateSessionName.ts:20` | `/rename` 명령으로 kebab-case 이름 생성 |

**추가 발견** (cc-notebook이 언급하지 않은 것으로, `getSmallFastModel()` 검색으로 발견):

| 번호 | 용도 | 파일 | 트리거 조건 |
|------|------|------|-----------|
| 7 | API Key 검증 | `services/api/claude.ts:544` | API Key 유효성 검증 (소스 코드 주석: "WARNING: if you change this to use a non-Haiku model, this request will fail in 1P") |
| 8 | Away 모드 요약 | `services/awaySummary.ts:49` | 사용자 자리 비움 시 컨텍스트 요약 생성 (AFK 모드) |
| 9 | Web 검색 보조 | `tools/WebSearchTool/WebSearchTool.ts:280` | 일부 Web 검색 시나리오에서 Haiku로 결과 처리 |
| 10 | Quota 상태 확인 | `services/claudeAiLimits.ts:200` | 가장 작은 Haiku 요청으로 현재 할당량 상태 탐지 |
| 11 | 토큰 수 추정 | `services/tokenEstimation.ts:277` | context window 사용량 추정 |
| 12 | Prompt/Exec Hook 실행 | `utils/hooks/execPromptHook.ts:79`, `execAgentHook.ts:118` | Hook 콜백은 기본적으로 Haiku를 사용 (hook 설정으로 재정의 가능) |
| 13 | Skill 개선 분석 | `utils/hooks/skillImprovement.ts:169` | Skill 실행 후 자동으로 개선 제안 분석 |

**결론**: cc-notebook의 "6가지 백그라운드 용도"는 **과소평가**입니다. 소스 코드에서 `queryHaiku` 또는 `getSmallFastModel()`의 호출 위치는 최소 13곳으로, 세션 생명 주기의 각 단계(시작 검증, 실행 중 보조, 세션 정리)를 포함합니다. Haiku/SmallFastModel은 전체 시스템의 백그라운드 "기반 서비스 레이어"이며, 가끔 등장하는 최적화 수단이 아닙니다.

핵심 설계 세부 사항: `queryHaiku`는 비스트리밍 호출(`queryModelWithoutStreaming`)을 사용하며, Tool permission context를 포함하지 않습니다 (`getEmptyToolPermissionContext()`):

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

## 13.5 Overload Fallback 메커니즘

cc-notebook은 "529 Overload Fallback, Opus → Sonnet 대체"가 존재한다고 주장합니다. 소스 코드에서 **완전히 검증**되었으며, 세부 사항은 더 풍부합니다.

### 529 오류 인식

`services/api/withRetry.ts`의 `is529Error()` 함수:

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // 529 상태 코드, 또는 스트리밍 시 SDK가 상태 코드를 올바르게 전달하지 못하는 경우 확인
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

이중 감지에 주목하세요: 상태 코드 `529`와 오류 메시지의 `overloaded_error` 문자열. SDK가 스트리밍 중 때로는 529 상태 코드를 올바르게 전달하지 못하기 때문입니다.

### 트리거 조건: 연속 3회 529

```typescript
// services/api/withRetry.ts — withRetry 함수 발췌
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
      // 특수 오류를 던져 상위 계층의 모델 전환 트리거
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

핵심 제약:
- 기본적으로 **비 ClaudeAI 구독 사용자**의 **Opus 계열 모델**에 대해서만 트리거됩니다 (`isNonCustomOpusModel()`)
- 환경 변수 `FALLBACK_FOR_ALL_PRIMARY_MODELS`로 모든 주요 모델로 확장 가능
- 스트리밍 요청의 529도 카운터에 포함됩니다 (`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`), 비스트리밍 재시도와 협력하여 계산

### FallbackTriggeredError 신호 전파

`FallbackTriggeredError`는 `originalModel`과 `fallbackModel` 필드를 포함하는 전용 오류 클래스로, 호출 스택을 타고 `query.ts`까지 전파됩니다:

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

### query.ts에서의 모델 전환과 사용자 알림

`query.ts:894-946`이 이 오류를 포착하여 실제 모델 전환을 실행합니다:

```typescript
// query.ts — FallbackTriggeredError 처리 발췌
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // warning 수준으로 사용자에게 표시 — verbose 모드 여부에 관계없이 표시됨
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // toolUseContext의 메인 루프 모델 동기 업데이트
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // 새 모델로 전체 요청 재시도
}
```

**사용자 알림 메커니즘**: 전환 메시지는 `'warning'` 수준을 사용합니다. 이는 사용자가 verbose 모드를 활성화했는지 여부에 관계없이 인터페이스에서 알림을 볼 수 있음을 의미합니다. **cc-notebook의 "무비밀 강등" 주장이 완전히 검증됩니다.**

### 백그라운드 작업의 529 전략: 즉시 포기

포어그라운드가 아닌 작업 (summary, title, suggestions 등)은 529 시 **재시도하지 않고** 즉시 삭제됩니다:

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

// 비포어그라운드 작업의 529는 즉시 던지고 재시도하지 않음
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

이것은 아키텍처 차원의 비용 제어 결정입니다: 백그라운드 작업 재시도는 용량 부족 시 3-10배의 게이트웨이 증폭 효과를 만들고, 사용자는 이런 작업 실패를 전혀 인지하지 못합니다.

---

## 13.6 Effort Level 메커니즘

cc-notebook은 Effort Level 시스템이 존재한다고 주장합니다. 소스 코드에서 **완전히 검증**되었으며, 세부 사항은 설명보다 훨씬 풍부합니다.

### 네 가지 Effort Level

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

각 수준의 의미 (`getEffortLevelDescription()` 기반):
- **low**: Quick, straightforward implementation with minimal overhead
- **medium**: Balanced approach with standard implementation and testing
- **high**: Comprehensive implementation with extensive testing and documentation
- **max**: Maximum capability with deepest reasoning (**Opus 4.6 전용**)

### 모델 지원 매트릭스

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // Opus 4.6와 Sonnet 4.6만 effort 파라미터를 지원
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku, 구버전 Opus/Sonnet은 지원하지 않음
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P 기본 true, 3P 기본 false
  return getAPIProvider() === 'firstParty'
}

// max effort는 Opus 4.6만 사용 가능
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### 우선순위 체인: env → appState → 모델 기본값

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' 또는 'auto' → effort 파라미터를 보내지 않음
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API가 비 Opus 4.6의 max를 거부 → 자동으로 high로 강등
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### 기본 Effort 차별화

Opus 4.6의 기본 effort는 구독 유형에 따라 다릅니다:

```typescript
// utils/effort.ts — getDefaultEffortForModel 발췌
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // Pro 사용자 기본 medium (할당량 절약)
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team도 GrowthBook 설정으로 medium으로 유도 가능
  }
}
```

흥미로운 점은 `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT`의 `dialogDescription`에 "We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits."라고 명시되어 있다는 것입니다. 이는 기본 medium이 의도적인 할당량 관리 전략임을 보여주며, 성능 우선이 아닙니다.

### max의 지속성 제한

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max는 비 ant 사용자에게는 세션 수준이며 지속되지 않음
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

외부 사용자가 설정한 `max` effort는 `settings.json`에 쓰이지 않고 현재 세션에서만 유효합니다.

---

## 13.7 비용 추적 시스템

### cost-tracker.ts의 핵심 역할

`cost-tracker.ts` (323줄)는 세 가지 역할을 담당합니다:
1. **실시간 누적**: API 응답마다 `addToTotalSessionCost()` 호출
2. **지속화**: 세션 종료 시 프로젝트 설정 파일에 씀 (`saveCurrentSessionCosts()`)
3. **복원**: 재시작 시 설정 파일에서 이전 비용 데이터 읽기 (`restoreCostStateForSession()`)

### 모델별 토큰 통계

`addToTotalModelUsage()`는 모델 이름별로 5차원 데이터를 누적합니다:

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

세션 종료 시 형식화하여 표시합니다 (`formatModelUsage()`): 짧은 이름으로 집계(여러 API 엔드포인트가 같은 모델을 다른 형식으로 반환)하여 다음과 같이 표시:

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Fast Mode의 비용 마킹

`addToTotalSessionCost()`에는 Fast Mode에 대한 특별 처리가 있습니다:

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

`speed: 'fast'` 마킹은 과금에 영향을 미칩니다 — Fast Mode에서 Opus 4.6은 `COST_TIER_30_150` ($30/$150)를 사용하고, 표준 `COST_TIER_5_25` ($5/$25)를 사용하지 않습니다.

### Advisor 중첩 비용 추적

`addToTotalSessionCost()`는 Advisor 도구의 사용량을 재귀적으로 처리합니다:

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

Advisor는 메인 모델 응답에 숨겨진 보조 모델 호출로, 비용이 별도로 추적되어 총 비용에 합산됩니다.

### 비용 표시 트리거 메커니즘

`costHook.ts` (22줄)는 프로세스 종료 이벤트를 감시하는 React hook입니다:

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

`hasConsoleBillingAccess()`는 비용 표시 여부를 제어하여, 과금 정보에 액세스할 수 없는 환경(CCR/Remote 모드 등)에서 비용을 표시하지 않습니다. 동시에 `saveCurrentSessionCosts()`는 무조건 실행됩니다 — 표시 여부에 관계없이 지속화됩니다.

---

## 13.8 API 호출 레이어

### claude.ts 요청 구성의 핵심 파라미터

`services/api/claude.ts` (3,419줄)는 API 호출의 통합 진입점입니다. 핵심 파라미터는 여러 시스템에서 수렴합니다:

```typescript
// services/api/claude.ts — 요청 파라미터 조합 (개략)
{
  model: normalizeModelStringForAPI(options.model),  // [1m] 접미사 제거
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // Effort 파라미터 (지원 모델만)
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()`는 API 전송 전에 `[1m]`과 `[2m]` 접미사를 제거합니다. 이 접미사들은 1M context window를 표시하는 클라이언트 내부 규약으로, API 레이어는 이를 인식하지 못합니다.

### 스트리밍 응답과 비스트리밍 폴백

메인 대화는 스트리밍 전송(Server-Sent Events)을 사용하지만, 스트리밍 전송이 실패하면 비스트리밍으로 폴백합니다:

```typescript
// services/api/claude.ts:2535-2559
// 스트리밍 자체가 529로 실패하면 이 횟수를 연속 529 카운터에 포함
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

비스트리밍 폴백에는 최대 토큰 제한이 있습니다:

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Beta Headers 동적 주입

다른 기능들은 다른 Beta Header에 해당하며, 요청 시 동적으로 첨부됩니다:

```typescript
// constants/betas.ts (참조)
EFFORT_BETA_HEADER        // effort 파라미터 지원
CONTEXT_1M_BETA_HEADER    // 1M context window
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // 예산 제어
```

---

## 13.9 설계 결정 분석

### 무비밀 강등의 설계 철학

`query.ts`의 `'warning'` 수준 전환 알림과 `FallbackTriggeredError` 전용 오류 클래스 설계를 통해 이것이 의도적인 아키텍처 선택임을 알 수 있습니다:

**왜 조용히 전환할 수 없는가?** Claude Code는 코드 작성 도구이기 때문에 모델 품질이 출력 품질에 직접 영향을 미칩니다. 사용자는 "나는 Opus가 아닌 Sonnet을 사용하고 있다"는 것을 알 권리가 있으며, 이를 통해 계속 기다릴지 아니면 다른 전략을 사용할지 결정할 수 있습니다. 소비자용 채팅 제품과 달리, 코드 도구의 사용자는 더 전문적이고 모델 차이에 더 민감합니다.

### 비용 투명성의 설계 고려

`costHook.ts`의 `hasConsoleBillingAccess()` 설계가 주목할 만합니다: 표시하지 않아도 비용 데이터는 지속화됩니다. 이는 비용 추적의 주요 목적이 **세션 복원** (다음 시작 시 이전 지출 표시)이지 실시간 경고가 아님을 보여줍니다. 이것은 "오프라인 인식" 설계입니다: 사용자는 매 API 호출 후 방해받는 것이 아니라 세션 종료 후 전체 지출을 볼 수 있습니다.

### 모델 기본값 차별화의 제품 논리

Opus를 Max/Team Premium의 기본 모델로, Sonnet을 Pro/PAYG의 기본 모델로 삼는 것에는 명확한 제품 논리가 있습니다: Max 구독의 가치 제안 중 하나가 "가장 강력한 모델 이용"이며, 기본값 자체가 이 가치 제안의 구현입니다.

동시에, Max 사용자라도 Opus 4.6의 기본 effort는 `medium` (GrowthBook으로 제어됨) 입니다. 이는 Anthropic이 effort 시스템을 통해 **품질과 할당량 사이의 균형을 맞추고** 있으며, Max 사용자에게 무조건 최고 설정을 제공하는 것이 아님을 보여줍니다.

---

## 13.10 모델 마이그레이션 (migrations)의 필요성

`migrations/` 디렉토리의 11개 마이그레이션 파일은 제품 진화의 흔적을 드러냅니다. 각 마이그레이션은 하나의 제품 결정에 대응합니다:

| 마이그레이션 파일 | 트리거 시점 | 핵심 로직 |
|---------|---------|---------|
| `migrateFennecToOpus.ts` | 내부 직원 (ant) | fennec 코드명 alias → opus alias (내부 코드명 정리) |
| `migrateLegacyOpusToCurrent.ts` | 1P 사용자, settings에 `opus-4-0`/`4-1` 존재 | 구버전 Opus model ID → `opus` alias (Opus 4.0/4.1 종료) |
| `migrateOpusToOpus1m.ts` | Max/Team Premium (1P), settings에 `opus` 존재 | `opus` → `opus[1m]` (1M 경험 통합) |
| `migrateSonnet1mToSonnet45.ts` | `sonnet[1m]` 사용자 | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` (4.5로 고정, 4.6 1M은 대상 다름) |
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium (1P), Sonnet 4.5로 고정 | Sonnet 4.5 문자열 → `sonnet` alias (4.6으로 업그레이드) |
| `resetProToOpusDefault.ts` | Pro 1P 사용자, 커스텀 모델 없음 | 마이그레이션 타임스탬프 기록, REPL이 한 번 업그레이드 알림 표시 |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode 활성화, 구버전 OptIn 대화 사용자 | `skipAutoPermissionPrompt` 초기화, 새 버전 권한 대화 재표시 |
| `migrateAutoUpdatesToSettings.ts` | 자동 업데이트를 명시적으로 비활성화한 사용자 | `autoUpdates: false`를 settings.json의 환경 변수로 마이그레이션 |
| `migrateBypassPermissionsAcceptedToSettings.ts` | 전역 설정에 `bypassPermissionsModeAccepted` 존재 | settings.json의 `skipDangerousModePermissionPrompt`로 마이그레이션 |
| `migrateSonnet45ToSonnet46.ts` | 상동 | 전술한 동명 마이그레이션 |
| `migrateEnableAllProjectMcpServersToSettings.ts` | MCP 관련 설정 | MCP 서버 설정 구조 조정 |

**아키텍처 인사이트**: 각 마이그레이션은 `userSettings`(사용자 수준 settings.json)만 조작하며, `projectSettings`(프로젝트 수준)나 `policySettings`(기업 정책 수준)는 건드리지 않습니다. 이것은 의도적인 설계입니다:

1. **멱등성**: 같은 데이터 소스를 읽고 쓰므로, 재실행해도 부작용이 없음
2. **최소 권한**: 사용자가 프로젝트 수준에서 고정한 것을 (교체해서는 안 되고) 교체할 수 없음
3. **전역 승격 방지**: 사용자가 특정 프로젝트에서 구버전 Opus를 고정했다면, 마이그레이션이 이를 전역 설정으로 승격시키지 않음

이 마이그레이션 시스템의 존재 자체가 보여줍니다: **AI 시스템의 스키마 마이그레이션은 전통적인 데이터베이스 마이그레이션보다 훨씬 복잡합니다** — 구독 유형 변경, 모델 종료, context window 업그레이드 등 여러 차원을 고려해야 하며, 단순하게 사용자 의도를 덮어쓸 수 없습니다.

---

## 13.11 이전 가능한 패턴

이 챕터 분석에서 자신의 시스템에 사용할 수 있는 5가지 설계 패턴을 추출합니다:

### 패턴 1: 다단계 Override 체인
```
session_override > startup_flag > env_var > config_file > builtin_default
```
어느 계층이든 상위 계층에 의해 재정의될 수 있지만, 하위 계층이 몰래 상위 계층에 영향을 줄 수 없습니다. allowlist 검사와 결합하여 불법 model ID 주입을 방지합니다.

### 패턴 2: 포어그라운드/백그라운드 529 전략 분리
포어그라운드 작업 (사용자가 결과를 기다림): N회 재시도, 초과 시 fallback 트리거.
백그라운드 작업 (사용자가 인지하지 못함): 첫 번째 529에서 즉시 포기하여 용량 위기 시 재시도 증폭 효과 방지.

### 패턴 3: FallbackTriggeredError 신호화
retry 내부에서 조용히 모델을 전환하지 않고, 전용 오류를 던져 더 상위 계층의 호출자가 전환 로직을 처리하게 합니다. 이렇게 전환 로직이 한 곳(query.ts)에 집중되고, 반드시 사용자 알림을 수반합니다.

### 패턴 4: toPersistableEffort 지속성 필터
세션 수준 설정 (예: `max` effort)은 settings.json에 쓰기 전에 필터링됩니다. "세션 간 지속되어서는 안 되는 상태"와 "지속되어야 하는 사용자 선호"가 데이터 모델 수준에서 구분됩니다.

### 패턴 5: 모델별 비용 버킷 추적
총 비용만 추적하지 않고, 모델 이름(정규화 후)별로 버킷을 나눕니다. 이렇게만 세션 종료 시 "Opus는 얼마, Haiku는 얼마"를 표시하여 사용자가 어떤 기능이 가장 비싼지 이해할 수 있습니다.

---

## 13.12 소스 코드 인덱스

| 파일 | 줄 수 | 핵심 내용 |
|------|-------|---------|
| `services/api/claude.ts` | 3,419 | API 호출 레이어, queryHaiku, 요청 구성, 스트리밍 처리 |
| `services/api/withRetry.ts` | ~600 | 재시도 로직, 529 처리, FallbackTriggeredError |
| `cost-tracker.ts` | 323 | 비용 추적, 지속화, 형식화 표시 |
| `costHook.ts` | 22 | React hook, 프로세스 종료 감시 비용 표시 트리거 |
| `utils/effort.ts` | ~350 | Effort Level 정의, 우선순위 체인, 모델 지원 탐지 |
| `utils/modelCost.ts` | ~200 | 가격표, 비용 계산 함수 |
| `utils/model/model.ts` | ~450 | 모델 우선순위 체인, alias 파싱, 기본 모델 로직 |
| `utils/model/modelCapabilities.ts` | ~100 | 모델 능력 캐시 (내부 사용자만) |
| `query.ts` | ~1000 | FallbackTriggeredError 포착, 사용자 알림, 모델 전환 |
| `migrations/*.ts` | 11개 파일 | 모델 버전 마이그레이션 스크립트 |
| `tools/WebFetchTool/utils.ts:503` | — | Haiku 용도 1: Web Fetch 내용 추출 |
| `utils/shell/prefix.ts:220` | — | Haiku 용도 2: Shell 명령 접두사 판단 |
| `utils/sessionTitle.ts:87` | — | Haiku 용도 3: 세션 제목 생성 |
| `utils/mcp/dateTimeParser.ts:68` | — | Haiku 용도 4: DateTime 파싱 |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Haiku 용도 5: 도구 호출 요약 |
| `commands/rename/generateSessionName.ts:20` | — | Haiku 용도 6: 세션 이름 변경 |
| `services/api/claude.ts:544` | — | Haiku 용도 7: API Key 검증 |

---
