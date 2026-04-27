# 제 14 장: Feature Flags와 관찰 가능성

## 14.1 개요 및 위치

Claude Code의 관찰 가능성 체계는 컴파일 타임 기능 제거부터 런타임 동작 추적까지 전 과정을 아우르는 다층적, 다목적 시스템입니다. 전체 체계는 세 가지 기둥으로 구성됩니다:

1. **Feature Flag 시스템**: 이중 트랙 설계—컴파일 타임 `feature()` 호출（`bun:bundle`을 통한 Dead Code Elimination）과 런타임 GrowthBook 동적 구성. 전자는 서로 다른 사용자 그룹에 릴리스된 기능 경계를 제어하고, 후자는 재배포 없이 기능 스위치를 조정할 수 있도록 지원합니다.

2. **관찰 가능성 파이프라인**: OpenTelemetry 표준을 기반으로 gRPC/HTTP/Protobuf 세 가지 내보내기 프로토콜을 지원하고, Metrics, Logs, Traces 세 가지 신호를 통합 수집하며, Perfetto 추적 형식으로 내부 디버깅 레이어를 제공합니다.

3. **Analytics 수집**: 이중 라우팅—Datadog（외부 모니터링）+ First-Party 1P 이벤트 로깅（내부 BigQuery/proto）. 이벤트 이름 접두사 `tengu_*`로 모든 비즈니스 이벤트를 식별하고, GrowthBook 동적 샘플링 구성으로 데이터 볼륨을 제어합니다.

이 체계의 핵심 설계 원칙은 **계층 격리**입니다: 사용자 개인 정보 우선（기본적으로 코드 내용 및 파일 경로 기록 없음）, 내부/외부 빌드 차별화（ant-only vs external）, graceful degradation（각 레이어에 kill-switch 있음）.

---

## 14.2 이론적 기반

### Feature Flag 주도 개발

Feature Flag（기능 플래그）는 팀이 동일한 코드베이스에서 다양한 단계의 기능을 병렬로 개발하고 필요에 따라 활성화할 수 있게 합니다. Claude Code는 두 레이어 flag 메커니즘을 채택합니다:

- **컴파일 타임 flag**: `bun:bundle`이 제공하는 `feature()` 호출을 통해 번들 시 Dead Code Elimination 수행. 외부 버전에 존재하지 않는 전체 코드 블록이 완전히 삭제되어 패키지 크기를 줄일 뿐만 아니라 내부 논리의 역공학도 방지합니다.
- **런타임 flag**: GrowthBook SDK를 통해 서버에서 동적으로 가져와 A/B 테스트, 점진적 롤아웃, 긴급 kill-switch 등의 시나리오를 지원합니다.

### 관찰 가능성의 세 가지 기둥

OpenTelemetry 커뮤니티는 관찰 가능성을 세 가지 신호（Three Pillars of Observability）로 정의합니다:

- **Metrics（지표）**: 시계열 수치 데이터, 예: API 지연, 토큰 소비량. Claude Code는 `@opentelemetry/sdk-metrics`를 사용하여 PeriodicExportingMetricReader로 60초마다 내보냅니다.
- **Logs（로그）**: 구조화된 이벤트 기록. 모든 `logEvent()` 호출은 최종적으로 OTel `LoggerProvider` + `BatchLogRecordProcessor`를 통해 일괄 내보내기됩니다.
- **Traces（추적）**: 분산 호출 체인. Claude Code는 `sessionTracing.ts`를 통해 Interaction → LLM Request → Tool Call의 계층적 Span 트리를 구축하고, 멀티 Agent 시나리오에서 부모-자식 관계 추적을 지원합니다.

### CLI 도구에서의 A/B 테스트 적용

웹 제품과 달리 CLI 도구의 A/B 테스트는 고유한 도전에 직면합니다: 브라우저 지문 없음, 멀티 플랫폼 멀티 배포 채널, 오프라인 실행 시나리오. Claude Code의 대응 전략:

- 사용자 차원 타겟팅: `GrowthBookUserAttributes`에 `platform`, `subscriptionType`, `rateLimitTier` 등의 속성이 포함되어 계층별 실험 지원.
- 로컬 디스크 캐시: 서버에서 특성 값을 성공적으로 가져올 때마다 `~/.claude/config.json`의 `cachedGrowthBookFeatures`에 기록하여 오프라인에서도 마지막으로 알려진 값을 사용 가능.
- 노출 중복 제거: 동일 session 내에서 각 feature의 실험 노출 이벤트는 한 번만 기록됩니다（`loggedExposures` Set）.

---

## 14.3 Feature Flag 시스템

### GrowthBook 통합

GrowthBook은 오픈소스 Feature Flag 및 A/B 테스트 플랫폼입니다. Claude Code는 공식 `@growthbook/growthbook` SDK로 통합하며, 파일 위치는 `src/services/analytics/growthbook.ts`（1155줄）입니다.

**초기화 흐름**:

```typescript
// growthbook.ts:529-600（간략화）
export const initializeGrowthBook = memoize(
  async (): Promise<GrowthBook | null> => {
    let clientWrapper = getGrowthBookClient()
    // ...
    await clientWrapper.initialized
    setupPeriodicGrowthBookRefresh()
    return clientWrapper.client
  },
)
```

핵심 설계: `memoize`는 전체 프로세스 생명주기 내에서 GrowthBook 클라이언트가 한 번만 초기화되도록 보장합니다. auth가 변경될 때（로그인/로그아웃）는 `refreshGrowthBookAfterAuthChange()`를 통해 클라이언트를 파괴하고 재빌드하며, `apiHostRequestHeaders` 업데이트를 시도하지 않습니다（SDK가 초기화 후 업데이트를 지원하지 않음）.

**사용자 속성 모델**（`growthbook.ts:31-46`）:

```typescript
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

**새로 고침 전략**:
- 외부 사용자: 6시간마다 새로 고침（`6 * 60 * 60 * 1000`）
- 내부 직원（ant）: 20분마다 새로 고침

**캐시 아키텍처**（세 가지 우선순위 레벨）:
1. 메모리의 `remoteEvalFeatureValues` Map（프로세스 내 최신 값）
2. 디스크 캐시 `~/.claude/config.json`의 `cachedGrowthBookFeatures`（크로스 프로세스 영속성）
3. 구 버전 `cachedStatsigGates`（마이그레이션 호환 레이어, 점차 폐지 중）

**API 호환 Workaround**（`growthbook.ts:320-390`）: 서버가 반환하는 remoteEval 응답은 `value` 필드를 사용하지만 SDK는 `defaultValue`를 기대하므로, 코드에 명시적인 형식 변환 로직이 있고 서버 수정을 기다리는 TODO 주석이 달려 있습니다.

**환경 변수 오버라이드**（ant 내부 사용자만）:
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### 컴파일 타임 Feature Flag 완전 목록（80+개）

`bun:bundle`의 `feature()` 호출로 dead code elimination을 구현합니다. 다음은 소스 코드에서 추출한 모든 컴파일 타임 flag입니다:

| Flag 이름 | 위치 | 제어 기능 |
|-----------|---------|---------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Perfetto 추적（ant-only） |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | 향상된 원격 측정 beta |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | 자동 모드/스케줄 작업 시스템 |
| `KAIROS_BRIEF` | `commands.ts` | KAIROS 간소화 모드 |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | KAIROS 채널 지원 |
| `KAIROS_DREAM` | `commands.ts` | KAIROS 드림 모드 |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | GitHub webhook 구독 |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | KAIROS 푸시 알림 |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Agent 트리거（스케줄 작업） |
| `AGENT_TRIGGERS_REMOTE` | — | 원격 Agent 트리거 |
| `AGENT_MEMORY_SNAPSHOT` | — | Agent 메모리 스냅샷 |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | 대화 분류기 |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | 검증 에이전트 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | 내장 탐색/계획 에이전트 |
| `COORDINATOR_MODE` | `builtInAgents.ts` | 코디네이터 모드 |
| `FORK_SUBAGENT` | `commands.ts` | Fork 자식 에이전트 |
| `BUDDY` | `commands.ts` | Buddy 기능 |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Unix Domain Socket 수신함 |
| `BRIDGE_MODE` | `commands.ts` | 브릿지 모드（CCR） |
| `DAEMON` | `commands.ts` | 데몬 모드 |
| `VOICE_MODE` | `commands.ts` | 음성 모드 |
| `ULTRAPLAN` | `commands.ts` | UltraPlan 명령 |
| `ULTRATHINK` | — | UltraThink 기능 |
| `TORCH` | `commands.ts` | TORCH 명령（동적 로딩） |
| `MCP_SKILLS` | `commands.ts` | MCP 기술 지원 |
| `CHICAGO_MCP` | `metadata.ts` | Chicago MCP 내장 서버（computer-use） |
| `WORKFLOW_SCRIPTS` | `commands.ts` | 워크플로우 스크립트 |
| `CCR_REMOTE_SETUP` | `commands.ts` | CCR 원격 설정 명령 |
| `CCR_AUTO_CONNECT` | — | CCR 자동 연결 |
| `CCR_MIRROR` | — | CCR 미러 모드 |
| `PROACTIVE` | `commands.ts` | 능동적 모드 |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | 실험적 기술 검색 |
| `HISTORY_SNIP` | `commands.ts` | 이력 스니펫 기능 |
| `HISTORY_PICKER` | — | 이력 선택기 |
| `WEB_BROWSER_TOOL` | — | 웹 브라우저 도구 |
| `QUICK_SEARCH` | — | 빠른 검색 |
| `MONITOR_TOOL` | — | 모니터링 도구 |
| `OVERFLOW_TEST_TOOL` | — | 오버플로우 테스트 도구 |
| `BREAK_CACHE_COMMAND` | — | 강제 캐시 중단점 명령 |
| `TREE_SITTER_BASH` | — | Tree-sitter Bash 파싱 |
| `TREE_SITTER_BASH_SHADOW` | — | Tree-sitter 그림자 비교 |
| `BASH_CLASSIFIER` | — | Bash 안전 분류기 |
| `TERMINAL_PANEL` | — | 터미널 패널 |
| `NATIVE_CLIPBOARD_IMAGE` | — | 네이티브 클립보드 이미지 지원 |
| `NATIVE_CLIENT_ATTESTATION` | — | 네이티브 클라이언트 증명 |
| `AUTO_THEME` | — | 자동 테마 |
| `POWERSHELL_AUTO_MODE` | — | PowerShell 자동 모드 |
| `TOKEN_BUDGET` | — | 토큰 예산 표시 |
| `STREAMLINED_OUTPUT` | — | 간소화된 출력 모드 |
| `CONNECTOR_TEXT` | — | 커넥터 텍스트 |
| `CONTEXT_COLLAPSE` | — | 컨텍스트 접힘 |
| `COMPACTION_REMINDERS` | — | 압축 알림 |
| `CACHED_MICROCOMPACT` | — | 캐시된 미세 압축 |
| `REACTIVE_COMPACT` | — | 반응형 압축 |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Prompt Cache 중단점 감지 |
| `EXTRACT_MEMORIES` | — | 자동 메모리 추출 |
| `LODESTONE` | — | Lodestone 기능 |
| `TEAMMEM` | — | 팀 메모리 |
| `TEMPLATES` | — | 템플릿 기능 |
| `FILE_PERSISTENCE` | — | 파일 영속성 |
| `BG_SESSIONS` | — | 백그라운드 세션 |
| `DOWNLOAD_USER_SETTINGS` | — | 사용자 설정 다운로드 |
| `UPLOAD_USER_SETTINGS` | — | 사용자 설정 업로드 |
| `NEW_INIT` | — | 새 버전 초기화 흐름 |
| `HARD_FAIL` | — | 하드 실패 모드 |
| `SLOW_OPERATION_LOGGING` | — | 느린 작업 로깅 |
| `SHOT_STATS` | — | 요청 통계 |
| `MEMORY_SHAPE_TELEMETRY` | — | 메모리 형태 원격 측정 |
| `COWORKER_TYPE_TELEMETRY` | — | 협업자 타입 원격 측정 |
| `ANTI_DISTILLATION_CC` | — | 반증류 보호 |
| `RUN_SKILL_GENERATOR` | — | 기술 생성기 |
| `SKILL_IMPROVEMENT` | — | 기술 개선 |
| `REVIEW_ARTIFACT` | — | 코드 리뷰 산출물 |
| `MESSAGE_ACTIONS` | — | 메시지 작업 |
| `AWAY_SUMMARY` | — | 자리 비움 요약 |
| `COMMIT_ATTRIBUTION` | — | 커밋 귀속 |
| `UNATTENDED_RETRY` | — | 무인 재시도 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | libc 타입 감지（빌드 시 주입） |

### 런타임 Feature Flag vs 컴파일 타임 Feature Flag

| 차원 | 컴파일 타임（`feature()`） | 런타임（GrowthBook） |
|------|----------------------|---------------------|
| 실행 시기 | 번들 단계 | 프로세스 시작 후 비동기 로딩 |
| 코드 보존 | 삭제된 분기는 산출물에 존재하지 않음 | 코드는 존재하지만 논리가 flag 값으로 제어됨 |
| 업데이트 방법 | 새 버전 배포 | 서버 측 푸시, 최대 20분 내 효력 발생 |
| 전형적인 용도 | ant-only 기능, 실험적 도구, 플랫폼 차이 코드 | A/B 테스트, 점진적 롤아웃, kill-switch, 동적 구성 |
| 오버라이드 방법 | 빌드 변수 | `CLAUDE_INTERNAL_FC_OVERRIDES` 환경 변수（ant만） |

### Dead Code Elimination 메커니즘

`bun:bundle`의 `feature()`는 Bun 번들러의 특수 내장 함수로, 빌드 단계에서 빌드 타임 정의에 따라 `feature('X')`를 직접 `true` 또는 `false`로 대체하고, 상수 폴딩 및 dead code elimination으로 항상 거짓인 분기를 제거합니다.

예시（`perfettoTracing.ts:216-220`）:
```typescript
// 외부 빌드에서 이 전체 if 블록은 완전히 삭제됩니다
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... 모든 Perfetto 초기화 코드
}
```

이 메커니즘은 패키지 크기를 줄일 뿐만 아니라 내부 도구 코드가 외부 산출물에 노출되는 것을 방지합니다.

### 알려진 중요한 런타임 Feature Flags

다음은 일부 알려진 `tengu_*` GrowthBook flag와 기능 설명입니다:

| Flag 이름 | 타입 | 기능 설명 |
|-----------|------|---------|
| `tengu_auto_mode_config` | Object | 자동 모드 구성（enabled/opt-in） |
| `tengu_1p_event_batch_config` | Object | 1P 이벤트 일괄 내보내기 구성 |
| `tengu_event_sampling_config` | Object | 이벤트 샘플링률 구성 사전 |
| `tengu_log_datadog_events` | Boolean | Datadog 이벤트 보고 스위치 |
| `tengu_frond_boric` | Object | Analytics sink kill-switch（sink 이름으로 비활성화） |
| `tengu_quartz_lantern` | Boolean | FileWriteTool 원자적 쓰기 동작 제어 |
| `tengu_hive_evidence` | Boolean | 작업 업데이트/Todo 쓰기 동작 제어 |
| `tengu_plum_vx3` | Boolean | WebSearchTool이 Haiku 모델 사용 여부 스위치 |
| `tengu_kairos_cron` | Object | KAIROS 스케줄 작업 구성 |
| `tengu_kairos_cron_durable` | Boolean | 영구적인 스케줄 작업 지원 |
| `tengu_agent_list_attach` | Boolean | AgentTool 목록 추가 동작 |
| `tengu_amber_stoat` | Boolean | 내장 에이전트 가용성 제어 |
| `tengu_slim_subagent_claudemd` | Boolean | 간소화된 자식 에이전트 CLAUDE.md 로딩 |
| `tengu_glacier_2xr` | Boolean | ToolSearch 모드 결정 제어 |
| `tengu_max_version_config` | Object | 최대 버전 제한（강제 업그레이드 kill-switch） |
| `tengu_prompt_cache_1h_config` | Object | Prompt Cache 1시간 구성 |
| `tengu_sm_compact_config` | Object | Session Memory 압축 구성 |
| `tengu_ant_model_override` | String | ant 전용 모델 오버라이드 |
| `enhanced_telemetry_beta` | Boolean | 향상된 원격 측정 beta 스위치 |

---

## 14.4 관찰 가능성 시스템

### OpenTelemetry 통합

Claude Code는 OpenTelemetry 세 가지 신호 지원을 완전히 구현하며, 핵심 진입점은 `src/utils/telemetry/instrumentation.ts`（825줄）입니다.

**초기화 부트스트랩**（`instrumentation.ts:bootstrapTelemetry()`）:
ant 빌드에서 `ANT_OTEL_*` 접두사 변수에서 구성을 읽어 표준 `OTEL_*` 변수로 매핑합니다. 외부 사용자는 표준 OTel 환경 변수 구성 규범을 따르며, 기본 temporality는 `delta`（누적이 아닌 증분）로 설정됩니다.

**세 가지 신호 내보내기 구성**（지연 로딩 설계）:

```typescript
// instrumentation.ts:169-190（간략화）
// OTLP/Prometheus 내보내기는 동적 import 지연 로딩 사용
// @grpc/grpc-js (~700KB)가 필요하지 않을 때 로딩되지 않도록 방지
case 'grpc': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-grpc'
  )
  exporters.push(new OTLPMetricExporter())
  break
}
case 'http/protobuf': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-proto'
  )
  exporters.push(new OTLPMetricExporter(httpConfig))
  break
}
```

세 가지 전송 프로토콜 모두 지원: `grpc`, `http/json`, `http/protobuf`. `OTEL_EXPORTER_OTLP_PROTOCOL` 환경 변수로 선택합니다.

**리소스 속성**: 서비스 이름은 `claude-code`이며, 플랫폼 아키텍처, WSL 버전, 구독 타입, 서비스 버전 등의 속성을 포함하고 `envDetector`, `hostDetector`, `osDetector`로 자동 채웁니다.

### gRPC 데이터 전송

gRPC는 기업 시나리오에서 권장되는 전송 프로토콜로, 양방향 스트리밍 전송과 강타입 protobuf 인코딩을 제공합니다. Claude Code에서:

- gRPC 내보내기（`@opentelemetry/exporter-metrics-otlp-grpc`）는 지연 로딩 의존성으로 시작 시간에 영향을 미치지 않음
- mTLS 구성은 `getMTLSConfig()`로 지원되며, 기업 내부망 시나리오에서 자서명 인증서 사용 가능
- 프록시 지원은 `getProxyUrl()` + `HttpsProxyAgent`로 투명하게 처리

자식 프로세스는 OTEL 관련 환경 변수를 상속하지 않습니다（`subprocessEnv.ts`）:
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Perfetto 추적

Perfetto는 Google이 개발한 고성능 시스템 수준 추적 프레임워크입니다. Claude Code는 Chrome Trace Event 형식의 호환 레이어를 구현했습니다（`src/utils/telemetry/perfettoTracing.ts`, 1120줄, ant-only）.

**활성화 방법**:
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # ~/.claude/traces/trace-<session-id>.json에 씀
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # 지정된 경로에 씀
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # 60초마다 주기적으로 씀
```

**추적되는 Span 타입**:

| Span 이름 | 분류 | 포함 정보 |
|-----------|------|---------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | attempt 번호 |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**메모리 관리**（이벤트 상한 100,000개）:
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// 상한에 도달하면 가장 오래된 절반을 삭제, 분할 상환 O(1) 비용
// trace_truncated 마커를 삽입하여 ui.perfetto.dev에서 갭을 볼 수 있게 함
```

**멀티 Agent 계층 추적**: 각 Agent（자식 Agent 포함）는 독립적인 process ID로 매핑되고, `parent_agent` metadata 이벤트로 계층 관계를 기록하여 Perfetto UI에서 독립적인 수영 레인으로 표시됩니다.

**쓰기 전략**（세 가지 보장）:
1. `cleanup registry` 비동기 콜백（정상 종료）
2. `process.on('beforeExit')` 핸들러（백업）
3. `process.on('exit')` 동기 쓰기（마지막 방어선, 이때는 async를 사용할 수 없음）

### OpenTelemetry Session Tracing

`src/utils/telemetry/sessionTracing.ts`（927줄）는 외부 사용자를 위한 향상된 원격 측정 진입점으로, Perfetto 형식이 아닌 표준 OTel Span 기반입니다.

**활성화 조건**（`sessionTracing.ts:170-185`）:
```typescript
export function isEnhancedTelemetryEnabled(): boolean {
  if (feature('ENHANCED_TELEMETRY_BETA')) {
    const env = process.env.CLAUDE_CODE_ENHANCED_TELEMETRY_BETA
      ?? process.env.ENABLE_ENHANCED_TELEMETRY_BETA
    if (isEnvTruthy(env)) return true
    if (isEnvDefinedFalsy(env)) return false
    return (
      process.env.USER_TYPE === 'ant' ||
      getFeatureValue_CACHED_MAY_BE_STALE('enhanced_telemetry_beta', false)
    )
  }
  return false
}
```

**AsyncLocalStorage 컨텍스트 전파**: 각 Interaction과 Tool Call은 독립적인 ALS 저장소 SpanContext를 사용하여 멀티 Agent 동시 시나리오에서 Span이 혼재되지 않도록 합니다. WeakRef 약한 참조로 오래 살아있는 span의 메모리 누출을 방지하고, 60초 간격으로 30분 이상 된 고아 Span을 정리합니다.

**logEvent 이벤트 체계**

모든 비즈니스 이벤트는 `src/services/analytics/index.ts`의 `logEvent()` 함수를 통해 통합 디스패치됩니다:

```typescript
// index.ts（간략화）
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // boolean | number | undefined만 허용
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

핵심 설계: metadata 타입이 의도적으로 `string`을 제외하여, 개발자가 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 타입 변환을 사용하도록 강제합니다. 타입 레벨에서 코드 내용이나 파일 경로를 실수로 기록하는 것을 방지합니다.

---

## 14.5 Analytics 수집

### 이중 라우팅 아키텍처

모든 이벤트는 `sink.ts`를 통해 두 가지 백엔드로 라우팅됩니다:

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog（tengu_log_datadog_events gate가 열려 있는 경우）
    │     DATADOG_ALLOWED_EVENTS 화이트리스트 이벤트만 전송
    │     _PROTO_* 키 제거（PII 마커 필드）
    └─→ 1P First-Party Logger（OpenTelemetry BatchLogRecordProcessor）
          /api/event_logging/batch로 전송
          _PROTO_* 키 보존（BigQuery 보호된 열로 라우팅）
```

**Datadog 통합**（`datadog.ts`）:
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- 일괄 전송: 100개/배치, 15초 새로 고침 간격
- 네트워크 타임아웃: 5초
- 화이트리스트 메커니즘: 약 50개의 핵심 이벤트（`DATADOG_ALLOWED_EVENTS` Set）
- 비활성화 조건: Bedrock/Vertex/Foundry 서드파티 클라우드, 테스트 환경, 사용자가 no-telemetry 선택

**1P Event Logging（FirstPartyEventLoggingExporter）**:
- OpenTelemetry 표준 `LogRecordExporter` 인터페이스 사용
- 일괄 내보내기: 기본 200개/배치, 5초 스케줄 지연
- 실패 재시도: 지수 백오프（기본 500ms, 최대 30초, 최대 8회）
- 영구적인 실패 큐: 실패한 이벤트를 `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl`에 기록, 다음 시작 시 재시도
- Proto 직렬화: 생성된 `ClaudeCodeInternalEvent` protobuf 타입 사용

### 사용자 행동 추적

약 400+ 개의 `tengu_*` 이벤트 이름이 완전한 사용자 상호작용 생명주기를 커버합니다. 핵심 이벤트 범주:

**세션 생명주기**: `tengu_started`, `tengu_init`, `tengu_exit`, `tengu_cancel`

**API 호출**: `tengu_api_query`, `tengu_api_success`, `tengu_api_error`, `tengu_api_retry`

**도구 사용**: `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_in_prompt_permanent`

**권한 요청**: `tengu_internal_bash_tool_use_permission_request`, `tengu_tool_use_show_permission_request`, `tengu_tool_use_granted_by_classifier`

**OAuth 인증**: `tengu_oauth_flow_start`, `tengu_oauth_success`, `tengu_oauth_token_refresh_*`（완전한 잠금 상태 기계 추적）

**MCP 서버**: `tengu_mcp_server_connection_succeeded`, `tengu_mcp_server_connection_failed`, `tengu_mcp_oauth_flow_*`

**업데이트 메커니즘**: `tengu_binary_download_attempt`, `tengu_native_update_complete`, `tengu_binary_download_failure`

### 성능 지표 수집

`sessionTracing.ts`의 API Call Span은 다음 파생 지표를 계산합니다:

```typescript
// perfettoTracing.ts（endLLMRequestPerfettoSpan 간략화）
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second（입력 토큰 처리 속도）

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second（샘플링 속도）

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // Cache 적중률（백분율）
```

### 이벤트 샘플링 제어

GrowthBook 동적 구성 `tengu_event_sampling_config`를 통해 각 이벤트의 샘플링률을 제어합니다:

```typescript
// firstPartyEventLogger.ts（shouldSampleEvent 간략화）
// null 반환 = 100% 샘플링（구성 없음）
// 0 반환 = 완전히 버림
// rate (0-1) 반환 = 무작위 샘플링, metadata에 sample_rate 기록
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // 예시: 10% 샘플링
}
```

### 오류 보고

다층적인 오류 이벤트 체계:
- `tengu_uncaught_exception`, `tengu_unhandled_rejection`: 프로세스 수준 미포착 오류
- `tengu_api_error`, `tengu_query_error`: API 호출 오류
- `tengu_streaming_error`: 스트리밍 응답 오류
- `tengu_atomic_write_error`: 파일 쓰기 오류
- `tengu_compact_failed`: 세션 압축 실패

---

## 14.6 진단 및 디버깅

### /doctor 명령

`src/commands/doctor/index.ts`가 `/doctor` 명령을 등록합니다:

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

이 명령은 `local-jsx` 타입으로 실행되어（REPL 내에서 직접 React 컴포넌트 렌더링）, 다음을 확인합니다: 설치 무결성, MCP 서버 연결 상태, 키 바인딩 구성 유효성, 환경 의존성（ripgrep 등）.

### 진단 추적 시스템

IDE 통합 시나리오에서 Claude Code는 Language Server Protocol을 통해 코드 진단 정보를 수신합니다. 파일이 저장되면（`didSave` 이벤트）TypeScript Server가 새로운 진단 메시지를 전송하고, 시스템은 이를 `<new-diagnostics>` XML 태그로 모델에 전달합니다:

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### 힙 메모리 진단

`src/utils/heapDumpService.ts`는 프로세스 수준 메모리 진단 능력을 제공합니다. 힙 덤프 트리거 시 메모리 사용 스냅샷을 동기적으로 수집하고, `~/Desktop/<session-id>-diagnostics.json`에 출력하며, `heapUsed`, `external`, `rss` 및 분석 제안을 포함합니다. 해당 analytics 이벤트: `tengu_heap_dump`.

### 오류 복구 로그

`src/utils/telemetry/bigqueryExporter.ts`는 BigQuery 지표 내보내기를 구현하여 OTEL Metrics 파이프라인과 통합하고, ant 내부의 장기 성능 모니터링 및 용량 계획에 사용됩니다. `1p_failed_events` 영구적인 큐는 네트워크 장애가 발생해도 중요한 이벤트가 손실되지 않도록 보장합니다.

---

## 14.7 설계 결정 분석

### 컴파일 타임 Flag의 장단점

**장점**:
1. **제로 런타임 오버헤드**: 삭제된 코드 분기는 산출물에 존재하지 않아 조건 판단 오버헤드가 없음
2. **보안 격리**: ant-only 기능 코드는 외부 사용자에게 완전히 보이지 않아 역공학 불가
3. **패키지 크기 최적화**: 대형 모듈（예: `@grpc/grpc-js` ~700KB）은 필요한 빌드에만 존재
4. **타입 안전성**: TypeScript의 타입 검사는 번들 전에 작용하여 런타임에 영향 없음

**단점**:
1. **배포 의존성**: flag 상태 변경은 새 버전 배포가 필요하여 핫 업데이트 불가
2. **테스트 매트릭스 폭발**: N개의 컴파일 타임 flag는 이론적으로 2^N가지 빌드 조합 테스트가 필요
3. **디버깅 복잡성**: 외부 사용자가 문제를 보고할 때, 그들의 빌드에는 일부 코드 경로가 전혀 존재하지 않을 수 있음

### 개인 정보와 관찰 가능성의 균형

Claude Code는 개인 정보 보호에 여러 방어선을 채택합니다:

1. **타입 시스템 보호**: `LogEventMetadata`는 `boolean | number | undefined`만 허용하고 문자열 직접 보고를 금지합니다. 문자열을 기록하려면 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`를 명시적으로 선언해야 합니다. 이것은 `never` 타입으로 실제로 값을 보유할 수 없습니다—단지 개발자가 해당 문자열이 코드나 경로를 포함하지 않음을 수동으로 검증했다는 타입 주석을 작성하도록 강제합니다.

2. **MCP 도구 이름 마스킹**: MCP 도구 이름 형식 `mcp__<server>__<tool>`이 사용자의 비공개 서비스 구성을 누출할 수 있어 기본적으로 `mcp_tool`로 마스킹됩니다. `cowork` 진입점, 공식 MCP 레지스트리의 서버, 또는 내장으로 명시적으로 선언된 서버 이름만 원본 이름을 보존합니다.

3. **PII 마커 필드**: `_PROTO_*` 접두사 metadata 키는 PII 민감 필드를 나타내어 1P 보호된 BigQuery 열로만 라우팅되며, `sink.ts`는 Datadog에 전달하기 전에 이 필드를 제거합니다.

4. **서드파티 클라우드 비활성화**: Bedrock/Vertex/Foundry를 사용하는 기업 고객은 Anthropic 측 analytics（Datadog 및 1P 포함）가 기본적으로 꺼집니다.

### 원격 측정의 지연 로딩 이유

OTLP 관련 패키지（gRPC 약 700KB, proto 약 300KB）는 동적 `import()`로 지연 로딩됩니다. 이유:

1. **시작 시간 민감성**: CLI 도구의 첫 번째 성능 지표는 Time-to-First-Output으로, 불필요한 초기화는 모두 지연해야 함
2. **프로토콜 상호 배타성**: 하나의 프로세스는 하나의 전송 프로토콜만 사용하므로, 모든 변형을 정적으로 import하는 것（6개 패키지）은 순수한 낭비
3. **Bun 최적화 호환성**: 지연 로딩은 Bun의 모듈 파싱 최적화 패턴과 일치하여 필요에 따라 번들

---

## 14.8 이전 가능한 패턴

다음 설계 패턴은 다른 프로젝트에서 높은 참고 가치를 가집니다:

### 1. 타입 시스템으로 PII 누출 방지

`never` 타입의 marker type을 통해 컴파일 타임에 개발자가 민감하지 않은 정보임을 명시적으로 확인하도록 강제합니다. 비용은 제로（runtime 오버헤드 없음）이며, 보호 효과는 100%（우회에는 명시적 타입 단언 필요）. 데이터 보고 요구사항이 있는 모든 시스템에 적용 가능합니다.

### 2. 이중 레벨 Feature Flag 아키텍처

컴파일 타임（코드 계층화）+ 런타임（동작 제어）이중 트랙으로, 서로 다른 배포 속도 요구사항에 해당합니다:
- 구조적 기능（전체 모듈의 유무）→ 컴파일 타임
- 동작 튜닝（파라미터, 비율, 알고리즘 선택）→ 런타임

### 3. Sink Kill-Switch 패턴

`tengu_frond_boric` GrowthBook 구성은 이름（`datadog`, `firstParty`）으로 임의의 analytics 백엔드를 독립적으로 끌 수 있어 새 버전 배포가 불필요합니다. 이것은 여러 downstream sink가 있는 이벤트 시스템 모두에 적합한 범용 긴급 회로 차단기 패턴입니다.

### 4. 실패 이벤트 영구적 재시도

1P 이벤트 내보내기 실패 시 로컬 JSONL 파일에 기록하고 다음 시작 시 재시도합니다. 이는 네트워크 장애 시 중요한 원격 측정 데이터가 손실되지 않도록 보장하며, 오프라인이나 불안정한 네트워크 환경에서 실행되는 도구에 특히 적합합니다.

### 5. 실험 노출 중복 제거

GrowthBook 실험 노출 이벤트（A/B 테스트 결과 분석용）는 session 수준 Set으로 중복 제거되어, 동일 feature의 노출이 분석 측에서 한 번만 기록됩니다. 동일 flag를 여러 번 호출하여 노출 카운트가 과장되는 것을 방지합니다.

---

## 14.9 소스 코드 색인

| 파일 경로（`src/` 상대） | 줄수 | 핵심 역할 |
|------------------------|------|---------|
| `services/analytics/growthbook.ts` | 1155 | GrowthBook SDK 통합, Feature Flag 읽기, A/B 노출 기록 |
| `services/analytics/index.ts` | 173 | logEvent 공개 API, 이벤트 큐, Sink 인터페이스 정의 |
| `services/analytics/sink.ts` | 114 | 이중 라우팅 구현（Datadog + 1P）, 초기화 |
| `services/analytics/datadog.ts` | 307 | Datadog 일괄 로그 전송, 화이트리스트 필터링 |
| `services/analytics/firstPartyEventLogger.ts` | 449 | OpenTelemetry LoggerProvider 초기화, 샘플링 제어 |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | 1P 이벤트 HTTP 내보내기, 영구적 재시도, proto 직렬화 |
| `services/analytics/metadata.ts` | 973 | 이벤트 메타데이터 보강, MCP 도구 이름 마스킹, PII 처리 |
| `services/analytics/config.ts` | 38 | isAnalyticsDisabled() 공유 로직 |
| `services/analytics/sinkKillswitch.ts` | 25 | Sink 수준 Kill-Switch（tengu_frond_boric） |
| `utils/telemetry/instrumentation.ts` | 825 | OTel SDK 초기화, 세 가지 신호（Metrics/Logs/Traces）구성 |
| `utils/telemetry/sessionTracing.ts` | 927 | OTel Span 관리, AsyncLocalStorage 컨텍스트 전파 |
| `utils/telemetry/perfettoTracing.ts` | 1120 | Perfetto Chrome Trace 형식 추적（ant-only） |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Beta 추적 확장 속성 |
| `utils/telemetry/bigqueryExporter.ts` | 252 | BigQuery 지표 내보내기 |
| `utils/telemetry/pluginTelemetry.ts` | 289 | 플러그인 원격 측정 캡슐화 |
| `utils/telemetry/events.ts` | 75 | OTel 이벤트 타입 정의 |
| `commands/doctor/index.ts` | 12 | /doctor 명령 등록 |
| `commands.ts` | — | 컴파일 타임 feature() 집중 호출 위치 |
