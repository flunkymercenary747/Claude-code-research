# 제 4 장: Agent 오케스트레이션과 멀티 에이전트 아키텍처

## 4.1 개요 및 위치

Claude Code의 멀티 에이전트 시스템은 전체 제품 아키텍처에서 가장 복잡한 서브시스템으로, 핵심 코드 약 8,700줄에 걸쳐 12개의 핵심 모듈을 아우르고 있습니다. 이 시스템은 근본적인 공학 문제를 해결합니다: **단일 스레드 REPL 애플리케이션이 어떻게 안전하고 효율적으로 여러 LLM Agent의 동시 실행을 오케스트레이션하는가**.

시스템은 세 가지 단계적 협업 모드를 제공합니다:

| 모드 | 트리거 방법 | 동시성 | 통신 메커니즘 | 격리 수준 |
|------|-----------|--------|-------------|---------|
| **Subagent（기본）** | AgentTool 호출 | 동기/비동기 | 함수 반환값 | 프로세스 내 AsyncGenerator |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | 완전 비동기 | `<task-notification>` XML | 독립 AbortController |
| **Team Mode** | `spawnTeammate()` + TeamFile | 영구적 병렬 | 파일 메일박스 + 폴링 | Tmux Pane / InProcess / Remote |

이 세 가지 모드는 독립적인 구현이 아니라 동일한 `runAgent()` 핵심 엔진(`runAgent.ts`)을 공유하며, 파라미터 조합으로 서로 다른 동작 특성을 구현합니다. 이것이 전체 시스템에서 가장 우아한 설계 결정 중 하나입니다.

**소스 코드 규모 통계:**

| 파일 | 줄수 | 역할 |
|------|------|------|
| `AgentTool.tsx` | 1,397 | 통합 진입점, 라우팅 결정, 생명주기 관리 |
| `runAgent.ts` | 973 | Agent 실행 엔진, query() 루프 |
| `loadAgentsDir.ts` | 755 | Agent 정의 파싱（Markdown/JSON/Plugin） |
| `agentToolUtils.ts` | 686 | 도구 필터링, 권한, 결과 직렬화 |
| `UI.tsx` | 871 | Agent 진행 상황 및 결과 렌더링 |
| `coordinatorMode.ts` | 369 | Coordinator 시스템 프롬프트 및 컨텍스트 |
| `SendMessageTool.ts` | 917 | 5가지 메시지 라우팅 |
| `spawnMultiAgent.ts` | 1,093 | Teammate 생성（Tmux/InProcess） |
| `inProcessRunner.ts` | 1,552 | InProcess 백엔드 완전 구현 |
| `teammateMailbox.ts` | 1,183 | 파일 메일박스 프로토콜 |
| `worktree.ts` | 1,519 | Git Worktree 격리 |

## 4.2 이론적 기반

### 4.2.1 Actor 모델과 Agent 오케스트레이션의 관계

Claude Code의 멀티 에이전트 아키텍처는 LLM 오케스트레이션 영역에서 Actor 모델의 실용적인 변형입니다. 고전적 Actor 모델（Hewitt, 1973）의 세 가지 핵심 프리미티브—**메시지 수신, 새 Actor 생성, 메시지 전송**—는 코드에서 명확한 대응 관계를 가집니다:

| Actor 프리미티브 | Claude Code 구현 | 소스 코드 위치 |
|-----------|-----------------|---------|
| 메시지 수신 | `waitForNextPromptOrShutdown()` 폴링 루프 | `inProcessRunner.ts:689-868` |
| Actor 생성 | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| 메시지 전송 | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

그러나 순수 Actor 모델에서 두 가지 핵심 이탈이 있습니다:

1. **비대칭 계층 구조**: Leader는 전역 뷰（AppState）를 가지고, Worker는 자신의 ToolUseContext만 가집니다. 이것은 대등한 Actor가 아니라, 명확한 Leader-Worker 계층 구조를 가진 감독 트리（supervision tree）입니다.
2. **공유 상태 채널**: InProcess 백엔드의 Teammate는 `setAppStateForTasks`를 통해 루트 AppState store를 공유합니다（`runAgent.ts:336-337`）. 순수 메시지 전달이 아니라 Actor 모델에 대한 실용적인 타협입니다. 단일 프로세스 내에서는 공유 상태가 메시지 직렬화보다 더 효율적입니다.

### 4.2.2 메시지 전달 vs 공유 메모리 동시성 모델

시스템은 격리 수준에 따라 두 가지 동시성 모델을 동시에 사용합니다:

**메시지 전달 모델**（Team Mode - Tmux Pane 백엔드）:
```
Leader → writeToMailbox("worker-1", {...}) → 파일 시스템 → readMailbox() → Worker
```
통신은 JSON 파일 + 파일 잠금으로 구현되며, `teammateMailbox.ts`의 `LOCK_OPTIONS`는 지수 백오프 재시도（10회 재시도, 5-100ms）를 구성하여 동시 쓰기를 직렬화합니다:

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**공유 메모리 모델**（InProcess 백엔드）:
```
Leader → setAppState(prev => {...}) → 동일한 AppState store ← getAppState() ← Worker
```
InProcess Teammate는 `toolUseContext.setAppStateForTasks`를 통해 루트 store를 직접 읽고 씁니다. 경쟁 조건은 React 스타일의 `setAppState(prev => {...})` 함수형 업데이트 의미론으로 방지합니다（하부가 React는 아니지만 동일한 CAS 패턴을 채택합니다）.

### 4.2.3 분산 시스템의 Coordinator 패턴

Coordinator Mode 설계는 분산 시스템의 고전적 Coordinator 패턴（Master-Worker라고도 함）을 매핑하지만, 독특한 제약을 추가합니다: **Coordinator 자체가 LLM Agent이며, 그 "조정 논리"는 하드코딩되지 않고 system prompt로 프로그래밍됩니다**.

`coordinatorMode.ts:126-369`에 정의된 `getCoordinatorSystemPrompt()` 함수는 약 5,000자의 구조화된 prompt를 반환하며, 완전한 Worker 스케줄링 전략을 포함합니다:

```typescript
// coordinatorMode.ts:161-167 — 핵심 스케줄링 규칙
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

"prompt로 조정 논리 프로그래밍"이라는 이 패턴은 Coordinator의 동작을 prompt 수정으로 조정할 수 있음을 의미합니다. 연구→합성→구현→검증의 4단계 워크플로우는 코드로 강제되지 않고 LLM의 지시 준수 능력으로 구현됩니다. 이는 전통적인 분산 Coordinator의 하드코딩된 스케줄링 논리와 뚜렷하게 대조됩니다.

## 4.3 아키텍처 및 데이터 구조

### 4.3.1 전체 아키텍처 다이어그램（Leader-Worker）

```
                    ┌─────────────────────────────────────────┐
                    │           Human User (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ user input
                    ┌──────────────▼──────────────────────────┐
                    │         Main REPL (query() loop)         │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — 라우팅 결정│     │
                    │    └──┬─────────┬─────────┬───────┘     │
                    │       │         │         │              │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐     │
                    │  │ Sync   │ │ Async  │ │Teammate │     │
                    │  │Agent   │ │Agent   │ │Spawn    │     │
                    │  │(block) │ │(fire&  │ │         │     │
                    │  │        │ │forget) │ │         │     │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘     │
                    │      │          │         │              │
                    │      └────┬─────┘    ┌────▼──────────┐  │
                    │           │          │  spawnMulti-   │  │
                    │      ┌────▼────┐     │  Agent.ts      │  │
                    │      │runAgent │     └────┬───────────┘  │
                    │      │  .ts    │          │              │
                    │      │         │     ┌────▼──────────┐  │
                    │      │ query() │     │  3 Backends:   │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    통신 레이어:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → parent collects      │
    │  Async Agent:   <task-notification> XML → user msg   │
    │  Teammate:      파일 메일박스 (.claude/teams/*/inboxes/) │
    │  InProcess:     AppState 공유 + mailbox fallback     │
    │  Remote (ant):  teleportToRemote() → CCR session     │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 AgentDefinition 타입 시스템

Agent 정의는 3층 유니온 타입 설계를 채택합니다:

```typescript
// loadAgentsDir.ts — 핵심 타입 계층

// 기본 타입: 모든 Agent가 공유하는 필드
type BaseAgentDefinition = {
  agentType: string              // 라우팅 키（예: "Explore", "worker"）
  whenToUse: string              // LLM이 Agent를 선택하는 근거
  tools?: string[]               // 화이트리스트（undefined = 전체）
  disallowedTools?: string[]     // 블랙리스트
  model?: string                 // 'inherit' | 구체적인 모델명
  effort?: EffortValue           // 추론 노력 수준
  permissionMode?: PermissionMode // 권한 상속 전략
  maxTurns?: number              // 최대 대화 턴 수
  background?: boolean           // 항상 백그라운드 실행
  isolation?: 'worktree' | 'remote' // 격리 모드
  memory?: AgentMemoryScope      // 영구 메모리
  omitClaudeMd?: boolean         // CLAUDE.md 생략（~5-15 Gtok/week 절약）
  // ...
}

// Built-in Agent: 동적 prompt, 정적 systemPrompt 없음
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent: Markdown/JSON에서 로드
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent: 플러그인 시스템에서 옴
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// 최종 유니온 타입
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

이 설계의 정교함은 `getSystemPrompt` 메서드에 있습니다: Built-in Agent는 `toolUseContext` 파라미터를 받아（현재 도구 세트에 따라 prompt를 동적으로 조정 가능）, Custom/Plugin Agent는 파싱 시점의 Markdown 내용을 클로저로 캡처합니다. 이는:

- **Built-in Agent의 prompt는 동적**: 매 호출마다 다를 수 있음
- **Custom Agent의 prompt는 정적**: Markdown 파일로 정의되지만, `memory`가 활성화된 경우 런타임에 메모리 내용이 추가됨（`loadAgentsDir.ts:335-340`）

Agent 정의의 로딩 우선순위는 오버라이드 체인을 따릅니다: `builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`. `getActiveAgentsFromList()`에서 Map으로 나중에 오는 것이 앞의 것을 덮어씁니다（`loadAgentsDir.ts:169-186`）.

### 4.3.3 세 가지 실행 백엔드의 통합 추상화

세 가지 백엔드는 동일한 `runAgent()` AsyncGenerator 인터페이스를 공유하지만, 프로세스 모델과 통신 메커니즘은 완전히 다릅니다:

| 차원 | Tmux Pane | InProcess | Remote（ant-only） |
|------|-----------|-----------|-------------------|
| **프로세스 모델** | 독립적인 Claude CLI 프로세스 | 동일 프로세스 AsyncLocalStorage 격리 | CCR 원격 세션 |
| **시작 방법** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **통신** | 파일 메일박스 폴링（500ms） | 공유 AppState + 파일 메일박스 fallback | HTTP API |
| **권한** | 독립적인 권한 컨텍스트 | Leader UI 큐 브릿징 | 원격 독립 |
| **리소스 오버헤드** | 높음（완전한 프로세스） | 낮음（공유 V8 힙） | 매우 높음（원격 인스턴스） |
| **생존 기간** | Leader 독립 | Leader 프로세스에 바인딩 | 독립 |
| **감지 로직** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

백엔드 감지와 폴백은 `spawnMultiAgent.ts:339-375`에서 우아한 폴백 체인으로 구현됩니다:

```
iTerm2 (it2 backend) → Tmux → InProcess fallback
```

iTerm2가 감지되었지만 it2 CLI가 설치되지 않은 경우, 시스템은 대화형 setup prompt（`It2SetupPrompt`）를 팝업하여 사용자가 it2 설치 또는 Tmux로 폴백을 선택할 수 있게 합니다.

### 4.3.4 통신 프로토콜 데이터 구조

**파일 메일박스 메시지 형식**（`teammateMailbox.ts:42-49`）:

```typescript
type TeammateMessage = {
  from: string       // 발신자 이름
  text: string       // 메시지 내용（순수 텍스트 또는 JSON 구조화 메시지）
  timestamp: string  // ISO 타임스탬프
  read: boolean      // 읽음 표시
  color?: string     // 발신자 색상 식별
  summary?: string   // UI 미리보기 요약（5-10 단어）
}
```

메일박스 경로는 고정 형식을 따릅니다: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**구조화된 메시지 타입**（`text` 필드의 JSON 인코딩으로 전달）:

| 메시지 타입 | 방향 | 용도 |
|---------|------|------|
| `shutdown_request` | Leader → Worker | 종료 요청 |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | 종료 응답 |
| `idle_notification` | Worker → Leader | 유휴 알림 |
| `permission_request` | Worker → Leader | 권한 요청 |
| `permission_response` | Leader → Worker | 권한 응답 |
| `plan_approval_request` | Worker → Leader | Plan Mode 승인 |
| `plan_approval_response` | Leader → Worker | 승인 응답 |
| `sandbox_permission_request` / `_response` | 양방향 | 네트워크 샌드박스 권한 |
| `task_assignment` | Leader → Worker | 작업 할당 |
| `team_permission_update` | Leader → Workers | 권한 브로드캐스트 |

## 4.4 핵심 알고리즘 및 흐름

### 4.4.1 AgentTool 라우팅 결정 트리（완전판）

`AgentTool.call()`은 시스템의 통합 진입점이며, 라우팅 논리는 `AgentTool.tsx:238-764`에 구현됩니다. 완전한 결정 트리는 다음과 같습니다:

```
AgentTool.call(input) 진입점
│
├─ [1] Team name + name 파라미터가 모두 존재하는가?
│   ├─ YES: Teammate가 중첩 생성을 시도하는가?
│   │   ├─ YES: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NO: → spawnTeammate() (return teammate_spawned)
│   └─ NO: 계속
│
├─ [2] effectiveType (subagent_type) 파싱
│   ├─ 명시적 지정 → 지정된 값 사용
│   ├─ 미지정 + Fork Gate ON → undefined (Fork 경로)
│   └─ 미지정 + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (Fork 경로)
│   ├─ YES: 재귀 Fork 검사
│   │   ├─ 이미 Fork 자식 프로세스 내 → throw
│   │   └─ 통과 → selectedAgent = FORK_AGENT
│   └─ NO: activeAgents에서 검색
│       ├─ 찾음 → selectedAgent = found
│       ├─ permission deny → throw（deny rule 정보 포함）
│       └─ 존재하지 않음 → throw（사용 가능한 agent 목록 표시）
│
├─ [4] effectiveIsolation 파싱
│   ├─ 'remote'（ant-only） → teleportToRemote() → return remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → 이후 단계에서 worktreePath 사용
│
├─ [5] system prompt 및 prompt 메시지 빌드
│   ├─ Fork 경로: 부모 prompt 상속 + buildForkedMessages()
│   └─ Normal: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] shouldRunAsync 판정
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — BUT NOT isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → return { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → while(true) 루프 진입
│       ├─ Race: nextMessage vs backgroundSignal
│       │   ├─ background wins → 비동기 실행으로 전환（wasBackgrounded=true）
│       │   └─ message wins → yield message, 진행 상황 추적
│       └─ 루프 종료 → finalizeAgentTool() → return AgentToolResult
```

### 4.4.2 runAgent() AsyncGenerator 실행 흐름

`runAgent()`는 전체 멀티 에이전트 시스템의 핵심 엔진입니다（`runAgent.ts:247-860`）. `AsyncGenerator<Message, void>`로, 매 Message를 yield할 때마다 호출자가 처리할 수 있습니다（기록, 표시 또는 백그라운드 큐에 추가）.

**실행 흐름의 핵심 단계:**

1. **도구 파싱**: `resolveAgentTools()`가 Agent 정의의 `tools` 화이트리스트를 실제 Tool 객체로 파싱하고, `disallowedTools` 블랙리스트를 적용합니다（`runAgent.ts:500-502`）

2. **System Prompt 빌드**: `override?.systemPrompt` 또는 `getAgentSystemPrompt()`에 따라 빌드. Explore/Plan Agent는 `claudeMd`와 `gitStatus`를 건너뛰어 fleet-wide ~5-15 Gtok/week를 절약합니다（`runAgent.ts:389-409`）

3. **AbortController 전략**（`runAgent.ts:524-528`）:
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // 외부 제어（async 경로）
     : isAsync
       ? new AbortController()      // 비동기: 독립적인 controller
       : toolUseContext.abortController  // 동기: 부모 controller 공유
   ```

4. **권한 오버라이드**（`runAgent.ts:414-497`）: Agent의 `permissionMode`가 부모 수준을 오버라이드하지만, `bypassPermissions`, `acceptEdits`, `auto` 세 가지 부모 수준 모드는 항상 우선합니다. 이는 관리자가 설정한 보안 정책이 자식 Agent에 의해 다운그레이드되지 않도록 보장합니다.

5. **핵심 루프**—직접 `query()`를 호출하고 yield합니다（`runAgent.ts:748-806`）:
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... stream_event, attachment, recordable messages 처리
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **정리 finally 블록**（`runAgent.ts:816-858`）: MCP 정리, session hooks 정리, prompt cache 추적, 파일 상태 캐시 해제, Perfetto 등록 해제, AppState todos 정리, 백그라운드 bash 작업 kill—총 9가지 정리 작업으로 리소스 누출을 방지합니다.

### 4.4.3 비동기 Agent 생명주기（fire-and-forget）

비동기 Agent의 완전한 생명주기는 `runAsyncAgentLifecycle()`（`agentToolUtils.ts:322-497`）이 구동합니다:

```
registerAsyncAgent() → AppState에 작업 등록
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — 모든 메시지 수집
   │   ├─ agentMessages.push(message)
   │   ├─ task.retain이면 → AppState.tasks[taskId].messages에 추가
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — SDK 진행 이벤트
   │
   ├─ finalizeAgentTool() — 최종 결과 추출
   │
   ├─ completeAsyncAgent() — 완료 표시（FIRST, 느린 작업 전에）
   │   │                      ↑ 핵심 설계: gh-20236 수정
   │   │                        classifyHandoff와 worktree cleanup이 hang될 수 있음
   │   │                        상태 전환을 차단해서는 안 됨
   │
   ├─ classifyHandoffIfNeeded() — 안전 분류기 검사（선택사항）
   │
   ├─ getWorktreeResult() — worktree 정리
   │
   └─ enqueueAgentNotification() — <task-notification> XML로 부모에 알림
```

**gh-20236 수정**은 기록할 가치가 있는 설계 결정입니다: `completeAsyncAgent()`가 `classifyHandoffIfNeeded()`와 `getWorktreeResult()` 전에 호출됩니다. 주석에 명확히 이유를 설명합니다:

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 도구 필터링 및 권한 상속

도구 필터링은 3층 필터 체인입니다（`agentToolUtils.ts:66-115`）:

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS — 모든 Agent가 사용 금지된 도구
Layer 2: CUSTOM_AGENT_DISALLOWED_TOOLS — Custom Agent만 추가로 금지된 도구
Layer 3: ASYNC_AGENT_ALLOWED_TOOLS — 비동기 Agent의 화이트리스트（역전 논리）
```

특수 예외:
- MCP 도구（`mcp__` 접두사）는 항상 허용
- `ExitPlanMode`는 Plan Mode에서 항상 허용
- InProcess Teammate는 Agent Swarms 모드에서 `AgentTool`（동기 자식 Agent 생성）과 Task 도구（공유 작업 목록 조정）를 사용할 수 있음

도구 파싱은 와일드카드（`'*'` 또는 `undefined` = 모든 도구）와 Agent 범위 제한（`AgentTool(worker, researcher)` 구문, `agentToolUtils.ts:165-172`）도 지원합니다.

### 4.4.5 Coordinator Mode 4단계 워크플로우

Coordinator Mode의 핵심 논리는 `coordinatorMode.ts:126-369`의 `getCoordinatorSystemPrompt()`에서 prompt로 정의됩니다. 모든 작업을 4단계로 분해합니다:

**Phase 1: Research**（Worker 병렬 실행）
- 여러 Worker가 동시에 코드베이스를 탐색
- 핵심 prompt 지시: *"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Phase 2: Synthesis**（Coordinator가 직접 수행）
- 이것이 가장 중요한 단계—Coordinator는 연구 결과를 직접 읽고 이해해야 함
- 명시적으로 금지된 안티패턴: *"Never write 'based on your findings'"*
- 구체적인 파일 경로, 줄 번호, 수정 내용이 포함된 synthesized spec 출력 필요

**Phase 3: Implementation**（Worker 실행）
- Coordinator가 continue（`SendMessageTool`）할지 spawn fresh（`AgentTool`）할지 결정
- 결정 기준은 컨텍스트 중첩도（prompt에 완전한 결정 테이블 포함）

**Phase 4: Verification**（독립적인 Worker）
- 독립적인 검증을 명시적으로 요구: *"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- 검증 기준: *"proving the code works, not confirming it exists"*

### 4.4.6 Team Mode 영구적 협업

Team Mode는 TeamFile（`.claude/teams/{team_name}/team.json`）을 기반으로 영구적인 팀 상태를 구현합니다. Coordinator Mode의 fire-and-forget Worker와 달리, Teammate는 **장기 실행 프로세스**입니다:

1. **생성**: `spawnTeammate()`가 Tmux pane 또는 InProcess 작업을 생성
2. **실행**: Teammate가 prompt를 실행 → 완료 → `idle_notification` 전송 → 다음 prompt 대기
3. **통신**: 모든 메시지는 파일 메일박스를 통해（모든 백엔드가 파일 시스템으로 통신 가능）
4. **종료**: Leader가 `shutdown_request` 전송 → Teammate의 LLM이 approve 또는 reject 결정

InProcess Runner의 주 루프（`inProcessRunner.ts:883-1464`）는 완전한 영구적 의미론을 구현합니다:

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. 현재 prompt 실행（runAgent() 호출）
  // 2. idle로 표시
  // 3. Leader에게 idle_notification 전송
  // 4. waitForNextPromptOrShutdown() — 메일박스 폴링
  //    ├─ shutdown_request → LLM에 전달하여 결정
  //    ├─ new_message → 다음 턴 prompt로 설정
  //    └─ aborted → shouldExit = true
}
```

주목할 만한 것은 메시지 우선순위 전략입니다（`inProcessRunner.ts:760-804`）:
1. 최고 우선순위: `shutdown_request`（Leader의 종료 지시가 묻히지 않음）
2. 다음: `team-lead`에서 온 메시지（Leader가 사용자 의도를 대표）
3. 마지막: FIFO 큐의 peer 메시지

### 4.4.7 파일 메일박스 통신 프로토콜

파일 메일박스는 모든 백엔드의 통신 기반입니다. 설계는 성능보다 **단순성**을 선택했습니다:

**쓰기 프로토콜**（`teammateMailbox.ts:133-191`）:
```
1. ensureInboxDir() — 디렉터리 존재 보장
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — 원자적 생성（존재하지 않는 경우）
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — 파일 잠금 획득
4. readMailbox() — 잠금 내에서 재읽기（더티 읽기 방지）
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — 쓰기 반환
7. release() — 잠금 해제
```

**읽기 프로토콜**（`teammateMailbox.ts:83-107`）:
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. TeammateMessage[] 반환
```

읽기는 **잠금 없음**입니다—이것은 의도적인 설계입니다. 읽기 측은 최종 일관성만 필요하며, 쓰기 측은 `lockfile`을 통해 원자성을 보장합니다.

### 4.4.8 SendMessage 5가지 라우팅

`SendMessageTool.call()`은 5가지 독립적인 메시지 라우팅 경로를 구현합니다（`SendMessageTool.ts`）:

```
input.to 값
│
├─ [라우팅1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — 크로스 머신 Remote Control
│   （안전 검사 필요: 크로스 머신 메시지는 사용자의 명시적 동의 필요）
│
├─ [라우팅2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — 로컬 Unix Domain Socket
│
├─ [라우팅3] agentNameRegistry 또는 toAgentId 매칭
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task stopped/evicted → resumeAgentBackground()
│       （디스크 transcript에서 중지된 Agent를 자동으로 복구）
│
├─ [라우팅4] to === '*'
│   → handleBroadcast() — TeamFile.members를 순회하며 개별 메일박스에 쓰기
│
└─ [라우팅5] 기타
    ├─ 순수 텍스트 → handleMessage() — 메일박스에 쓰기
    └─ 구조화된 메시지 → 해당 handler로 분배:
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

라우팅 3의 **자동 복구** 메커니즘이 특히 정교합니다: 이미 중지된 Agent에 메시지를 보낼 때, 시스템이 디스크 transcript에서 자동으로 복구하여 백그라운드에서 실행합니다. 이는 Coordinator가 `SendMessage`를 통해 이전에 완료한 Worker를 계속 진행시킬 수 있음을 의미하며, 실행 중인지 여부를 신경 쓸 필요가 없습니다.

### 4.4.9 권한 위임 완전 흐름

InProcess Teammate의 권한 처리는 전체 시스템에서 가장 복잡한 부분 중 하나입니다（`inProcessRunner.ts:127-449`）. 핵심 도전은: **백그라운드 Agent가 어떻게 인간의 인가를 요청하는가?**

해결책은 2층 폴백입니다:

**주 경로: Leader UI 큐 브릿징**
```
Worker가 권한이 필요한 도구 트리거
  → createInProcessCanUseTool() 호출
  → hasPermissionsToUseTool() returns { behavior: 'ask' }
  → Bash classifier 자동 승인 확인（사용 가능한 경우）
  → getLeaderToolUseConfirmQueue() — Leader의 UI 확인 큐 획득
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ Worker 신원 표시
     └→ Leader 터미널에 Worker badge가 있는 권한 대화상자 표시
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Fallback 경로: 메일박스 권한 요청**
```
Worker가 권한이 필요한 도구 트리거
  → Leader UI 큐 사용 불가
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → 자신의 메일박스 폴링（500ms 간격）
  → Leader가 permission_response를 쓰기 반환 대기
  → processMailboxPermissionResponse()
```

권한 업데이트의 전파도 중요합니다: Leader가 권한을 승인하고 "Always allow"를 선택하면, `persistPermissionUpdates()`가 디스크에 기록하고, `getLeaderSetToolPermissionContext()`가 업데이트를 Leader의 공유 컨텍스트에 다시 씁니다—그러나 `preserveMode: true`로, Worker의 `acceptEdits` 모드가 Coordinator로 누출되는 것을 방지합니다（`inProcessRunner.ts:275-277`）.

### 4.4.10 Worker 완전 생명주기

```
탄생
  │
  ├─ Sync Agent 경로:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → return AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ Async Agent 경로:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ InProcess Teammate 경로:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — 영구 루프:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → 새 메시지/shutdown/abort → 계속 여부 결정
          }

실행 중
  │
  ├─ query() 루프 → API 호출 → tool_use → canUseTool 검사
  │   ├─ allow → 도구 실행
  │   ├─ deny → 도구 거부됨
  │   └─ ask → 권한 대화상자（sync）또는 메일박스 권한（async/teammate）
  │
  ├─ 진행 추적:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (SDK 이벤트)
  │
  └─ 자동 백그라운드화（Sync Agent만）:
      backgroundPromise race → 사용자가 Ctrl+Z를 누르면
      → wasBackgrounded = true → 백그라운드에서 계속 실행

통신
  │
  ├─ Sync Agent: yield message → 부모가 직접 수집
  ├─ Async Agent: <task-notification>을 부모 user messages에 주입
  └─ Teammate: writeToMailbox() → Leader가 폴링으로 읽기

종료
  │
  ├─ 정상 완료: finalizeAgentTool() → 최종 텍스트 추출 → completed 표시
  ├─ 사용자 Kill: AbortError → killAsyncAgent() → partialResult 추출 → 알림
  ├─ 오류: catch → failAsyncAgent() → 오류 알림
  └─ 정리: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Worktree 격리의 생성 및 정리

Git Worktree는 Agent에게 파일 시스템 수준의 격리를 제공합니다（`worktree.ts`）. 핵심 흐름:

**생성**（`worktree.ts:234-374`）:
```
1. validateWorktreeSlug(slug) — 경로 순회 공격 방지
2. 빠른 복구 검사: readWorktreeHeadSha() — worktree가 이미 존재하면 fetch 건너뜀
3. 존재하지 않는 경우:
   a. 로컬 origin/<default> ref 읽기 시도（`git fetch`의 6-8초 오버헤드 방지）
   b. 로컬에 없으면 → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. 선택사항: sparse-checkout（지정된 경로만 체크아웃）
4. performPostCreationSetup():
   - settings.local.json 복사
   - git hooks 구성（husky의 core.hooksPath 문제 처리）
   - node_modules 등 대형 디렉터리에 심볼릭 링크
   - .worktreeinclude에 지정된 gitignored 파일 복사
```

**정리 결정**（`AgentTool.tsx:644-685`）:
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // Hook 기반은 항상 보존
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // 변경 없음, worktree 삭제
    }
  }
  return { worktreePath, worktreeBranch }; // 변경 있음, 보존
};
```

핵심 보안 조치:
- `validateWorktreeSlug()`가 각 `/`로 구분된 세그먼트가 `[a-zA-Z0-9._-]+`와 일치하는지 검증하여 `../../../` 경로 순회 방지
- `flattenSlug()`가 중첩된 slug를 평탄화（`user/feature` → `user+feature`）하여 git ref D/F 충돌과 디렉터리 중첩 문제 방지
- `GIT_NO_PROMPT_ENV`가 모든 git 자격증명 프롬프트를 비활성화하여 CLI hang 방지

## 4.5 설계 결정 분석

### 4.5.1 IPC 대신 파일 메일박스를 선택한 이유

파일 메일박스는 "원시적인" 선택처럼 보입니다—왜 Unix Domain Socket, Named Pipe, gRPC를 사용하지 않았을까요?

**핵심 이유: 백엔드 독립성**. 파일 시스템은 세 가지 백엔드（Tmux, InProcess, Remote）의 최대공약수입니다:
- Tmux Pane은 독립적인 프로세스로, 공유 메모리가 없음
- InProcess는 동일 프로세스지만 AsyncLocalStorage 격리 사용
- Remote는 네트워크를 거치지만 네트워크 파일 시스템 공유 가능

파일 메일박스의 추가 장점:
1. **관찰 가능성**: 직접 `cat ~/.claude/teams/*/inboxes/*.json`으로 디버깅 가능
2. **영속성**: 프로세스 크래시 후에도 메시지 손실 없음
3. **단순성**: 복잡한 연결 관리, 하트비트, 재연결 불필요
4. **동시성 안전**: `proper-lockfile`이 제공하는 파일 잠금으로 충분히 신뢰할 수 있음

대가는 **지연**: 500ms 폴링 간격은 최악의 경우 메시지 전달에 500ms 지연을 의미합니다. 그러나 LLM Agent 시나리오에서는 각 도구 호출 자체가 수 초가 걸리므로 500ms는 무시할 수 있습니다.

### 4.5.2 InProcess vs Pane 백엔드의 트레이드오프

| 차원 | InProcess | Tmux Pane |
|------|-----------|-----------|
| **메모리** | 공유 V8 힙（낮음） | 독립적인 프로세스 힙（높음） |
| **시작 지연** | ~0ms | ~2-3초（CLI 시작） |
| **격리** | AsyncLocalStorage（약함） | OS 프로세스（강함） |
| **권한** | Leader UI 브릿징（실시간） | 메일박스 폴링（지연） |
| **디버깅** | 공유 로그（복잡） | 독립 터미널（직관적） |
| **생존 기간** | Leader에 바인딩 | 독립적 |

InProcess 백엔드의 가장 큰 장점은 **권한 브릿징**입니다—`getLeaderToolUseConfirmQueue()`를 통해 Worker의 권한 대화상자가 Leader의 터미널에 Worker 배지와 함께 직접 표시됩니다. 이는 사용자가 권한을 승인하기 위해 Worker의 터미널로 전환할 필요가 없음을 의미합니다.

그러나 InProcess에는 근본적인 제한이 있습니다: **Worker는 백그라운드 Agent를 생성할 수 없습니다**（`AgentTool.tsx:277-278`）. 생명주기가 Leader 프로세스에 바인딩되어 있고, 백그라운드 Agent는 독립적인 AbortController가 필요하기 때문입니다.

### 4.5.3 권한은 항상 인간이 통제한다는 설계 철학

전체 멀티 에이전트 시스템의 권한 설계는 타협할 수 없는 원칙을 따릅니다: **인간은 항상 최종 권한 부여자입니다**.

이 원칙이 코드에서 구현된 방식:
1. **자식 Agent는 권한을 승격할 수 없음**: `runAgent.ts:419` — `bypassPermissions`, `acceptEdits`, `auto` 모드의 부모 설정이 자식 Agent의 `permissionMode`보다 항상 우선
2. **Leader의 permission이 Worker로 누출되지 않음**: `runAgent.ts:467-477` — `allowedTools`가 지정되면 session 수준의 allow 규칙을 지우고 CLI 인수 수준의 규칙만 유지
3. **크로스 머신 메시지는 명시적 동의 필요**: `SendMessageTool.ts:checkPermissions` — `bridge:` 주소로 전송에는 `safetyCheck`가 필요하며 `classifierApprovable: false`（안전 분류기가 자동 승인 불가）
4. **Plan Mode 승인**: Teammate를 `plan_mode_required`로 설정하면, Leader가 계획을 승인하기 전까지 실행 불가

### 4.5.4 query() 루프 재사용의 재귀적 설계

`runAgent()`의 핵심은 `query()` 함수를 호출하는 것입니다—그리고 `query()`는 주 REPL 루프가 사용하는 것과 동일한 함수입니다. 이는 **자식 Agent와 주 Agent가 완전히 동일한 API 호출 및 도구 실행 파이프라인을 사용함**을 의미합니다.

```typescript
// runAgent.ts:748-757 — Agent의 query() 호출
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```

이 설계의 심오한 영향:
- **도구 일관성**: Agent가 사용하는 도구는 사용자가 사용하는 것과 완전히 동일합니다（필터링만 거침）
- **재귀 능력**: Agent의 도구 풀에 `AgentTool`이 포함될 수 있으므로 Agent가 자식 Agent를 생성할 수 있습니다（InProcess Teammate는 동기 자식 Agent 생성 허용）
- **Prompt Cache 재사용**: Fork 경로는 `useExactTools`를 통해 자식 Agent의 API 요청 접두사가 부모 Agent와 바이트 단위로 일치하게 하여 prompt cache 적중률을 극대화합니다

그러나 재귀는 위험도 가져옵니다—무한 재귀 fork. 해결책은 이중 검사입니다（`AgentTool.tsx:331-333`）:
1. `querySource === 'agent:builtin:fork'` — 컴파일 타임 내구성（context.options는 autocompact의 영향을 받지 않음）
2. `isInForkChild(messages)` — 메시지 스캔 폴백

### 4.5.5 LangGraph / AutoGen / CrewAI와의 비교

| 차원 | Claude Code | LangGraph | AutoGen | CrewAI |
|------|------------|-----------|---------|--------|
| **오케스트레이션 모델** | Leader-Worker（prompt-programmed） | DAG/StateGraph | Agent Chat | Sequential/Hierarchical |
| **통신** | 파일 메일박스 + 공유 AppState | State channels | Python function calls | Shared memory |
| **격리** | 3수준（InProcess/Pane/Remote） | 없음 | 없음 | 없음 |
| **권한** | Human-in-the-loop, 항상 | 선택사항 | 선택사항 | 없음 |
| **영속성** | 디스크 transcript + TeamFile | 선택적 checkpointing | 없음 | 없음 |
| **도구 공유** | 통합 Tool 풀 + 필터링 | 각 노드 독립 바인딩 | 각 Agent 독립 | 각 Agent 독립 |
| **모델 이기종** | `model` 파라미터로 Agent별 지정 | 지원 | 지원 | 지원 |

Claude Code의 가장 큰 차별화는 두 가지입니다:

1. **Coordinator 논리는 prompt-programmed** — 다른 프레임워크의 오케스트레이션 논리는 코드로 하드코딩된 DAG 또는 상태 기계입니다. Claude Code의 Coordinator는 자연어 prompt로 프로그래밍되어 있어, 오케스트레이션 전략을 코드 변경 없이 prompt 수정으로 조정할 수 있습니다.
2. **파일 시스템을 통신 기반으로** — 원시적으로 보이지만, 크로스 프로세스, 크로스 머신 통합 통신 능력과 완전한 관찰 가능성을 제공합니다. 다른 프레임워크는 프로세스 내 Python function call에 의존하며, 멀티 머신 시나리오에서는 추가적인 RPC 레이어가 필요합니다.

## 4.6 이전 가능한 패턴

### 4.6.1 Agent 오케스트레이션의 범용 패턴

Claude Code 구현에서 5가지 범용 Agent 오케스트레이션 패턴을 추출할 수 있습니다:

**패턴 1: AsyncGenerator를 Agent 인터페이스로**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGenerator는 풀 기반（pull-based）메시지 스트림 의미론을 제공합니다—호출자가 다음 메시지를 소비할 시기를 결정하며, 백그라운드 전환（yield 지점에서 race 삽입）과 진행 추적을 자연스럽게 지원합니다.

**패턴 2: Foreground → Background 원활한 전환**

Claude Code의 Sync Agent는 실행 중에 `Promise.race([nextMessage, backgroundSignal])`를 통해 백그라운드로 전환될 수 있습니다. 이 패턴은 "장기 작업을 중간에 백그라운드화할 수 있어야 하는" 모든 시나리오에 적용됩니다. 핵심은 foreground와 background 사이에서 전달되는 안정적인 taskId를 갖는 것입니다.

**패턴 3: 파일 시스템을 Agent 간 통신의 "최소공배수"로**

여러 백엔드（프로세스 내/크로스 프로세스/크로스 머신）가 통합 통신이 필요할 때, 파일 시스템이 가장 단순한 선택입니다. JSON 파일 + 파일 잠금이 충분한 일관성 보장을 제공합니다.

**패턴 4: Prompt-Programmed Coordination**

오케스트레이션 논리를 코드가 아닌 system prompt에 작성하면 조정 전략이 "구성"이 아닌 "구현"이 됩니다. 이것은 Agent 오케스트레이션이 빠르게 반복되는 단계에서 특히 가치가 있습니다—prompt 변경 비용이 코드 변경 비용보다 훨씬 낮습니다.

**패턴 5: 알림 장식보다 안전한 상태 전환 우선**

gh-20236 수정 패턴: 비동기 흐름에서 먼저 핵심 상태 전환을 완료하고（`completeAsyncAgent`）, 그다음 hang될 수 있는 장식 작업을 수행합니다（classifier check, worktree cleanup）. 차단될 수 있는 모든 작업이 핵심 상태 변경을 gate해서는 안 됩니다.

### 4.6.2 Doramagic FlowController에서 참고할 수 있는 것

Claude Code의 Agent 아키텍처와 Doramagic의 FlowController（lease 시스템 + staging/delivery 격리 + 12 상태 기계）에는 몇 가지 참조할 만한 대응 관계가 있습니다:

1. **상태 기계 vs Prompt-Programmed**: Doramagic은 12 상태 기계로 흐름 제어를 하드코딩하고, Claude Code는 prompt로 Coordinator를 프로그래밍합니다. 둘 다 적합한 시나리오가 있습니다—결정론적 흐름에는 상태 기계, 유연한 판단이 필요한 흐름에는 prompt 프로그래밍.

2. **파일 메일박스의 직접 이전 가능성**: Doramagic의 staging/delivery 디렉터리 격리는 Claude Code의 `.claude/teams/*/inboxes/` 구조와 이치가 같습니다. Doramagic의 FlowController는 파일 메일박스 패턴을 직접 채택하여 skill 간 느슨하게 결합된 통신을 구현할 수 있습니다.

3. **권한 모델 참고**: Claude Code의 "자식 Agent는 권한을 승격할 수 없다" 원칙을 Doramagic의 skill 권한에 매핑할 수 있습니다—호출된 skill은 호출자보다 높은 시스템 액세스 권한을 가질 수 없습니다.

4. **Worktree 격리 아이디어**: Doramagic의 병렬 skill 실행（여러 soul extractor가 서로 다른 프로젝트를 병렬로 추출하는 경우）에서 Worktree의 파일 시스템 격리 패턴을 참고하여 각 병렬 실행을 위한 독립적인 작업 디렉터리를 만들 수 있습니다.

## 4.7 소스 코드 색인

| 파일 | 경로 | 핵심 내보내기 |
|------|------|---------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool`（buildTool 정의）, `inputSchema`, `outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | `runAgent()` AsyncGenerator, `filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition` 타입 유니온, `getAgentDefinitionsWithOverrides()`, `parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`, `resolveAgentTools()`, `finalizeAgentTool()`, `runAsyncAgentLifecycle()`, `classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`, `getCoordinatorSystemPrompt()`, `getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool`（5가지 라우팅）, `handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`, `handleSpawnSplitPane()`, `resolveTeammateModel()`, `buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`, `createInProcessCanUseTool()`, `waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`, `writeToMailbox()`, `markMessageAsReadByIndex()`, 모든 구조화된 메시지 타입 |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`, `createAgentWorktree()`, `removeAgentWorktree()`, `validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | `TaskState` 유니온（7가지 task 타입）, `isBackgroundTask()` |

**TaskState 유니온 타입**（`tasks/types.ts`）:
```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

---

*이 장은 Claude Code TypeScript 소스 코드 스냅샷（2026-03-31, ~512K LOC）분석을 기반으로 합니다. 모든 코드 참조는 구체적인 파일명과 줄 번호 범위가 표시되어 있습니다.*
