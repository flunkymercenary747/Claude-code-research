# 3장: 도구 시스템

## 3.1 개요 및 위치

Claude Code의 도구 시스템은 전체 제품의 실행 계층이다. LLM이 추론과 결정을 담당하지만, 실제 부작용—파일 읽기, 명령 실행, 코드 검색, 네트워크 접근—은 모두 도구 시스템을 통해 완성된다. 도구 시스템은 LLM의 의도와 실제 세계 사이의 유일한 채널이다.

규모로 보면, 이것은 상당히 큰 서브시스템이다:
- 소스 스냅샷에서 도구 디렉토리는 **40+ 개의 서브디렉토리**를 포함하며, 파일 조작, 코드 실행, Agent 조율, MCP 통합, 작업 관리 등의 카테고리를 포함
- 핵심 추상화 파일 `Tool.ts`는 792행, 도구 등록 파일 `tools.ts`는 389행, 도구 실행 엔진 `services/tools/toolExecution.ts`는 1,745행
- 도구 결과 저장 모듈 `utils/toolResultStorage.ts`는 1,040행으로, token 예산 문제를 독립적으로 처리

이 규모가 한 가지 사실을 보여준다: **도구 시스템은 Claude Code의 부속품이 아니라 핵심 공학 자산이다**. 전체 제품의 신뢰성, 보안성, 확장성은 크게 도구 시스템의 설계 품질에 의해 결정된다.

경쟁 제품 분석(cc-notebook)에는 독립적인 도구 시스템 장이 없으며, 이것은 명확한 분석 공백이다—본 장이 이 공백을 채운다.

---

## 3.2 이론적 기반

### 자기 기술 도구 (Self-Describing Tools) 패턴

전통적인 API 호출에서는 호출자가 인터페이스 사양을 미리 알아야 한다. Claude Code의 도구 시스템은 다른 설계 철학을 채택했다: **각 도구가 스스로의 능력, 입력 형식, 사용 제약을 자기 기술한다**.

이것은 `Tool` 타입의 여러 핵심 필드에 반영된다:

```typescript
// Tool.ts:300-310 (단순화)
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // 3-10 단어의 능력 요약, ToolSearch 키워드 매칭용
  description(input, options): Promise<string>   // 동적으로 설명 생성
  prompt(options): Promise<string>               // 도구의 완전한 시스템 프롬프트
  inputSchema: Input           // Zod schema, 문서이자 검증기
  outputSchema?: z.ZodType
  // ...
}
```

`description()`과 `prompt()`는 비동기 메서드로, 이것은 도구의 자기 기술이 **동적으로 생성**될 수 있음을 의미한다—현재 권한 컨텍스트, 설치된 도구, 환경 상태에 따라 프롬프트 내용을 조정한다. 이것은 정적 문서가 아니라 런타임에 생성되는 컨텍스트 인식 설명이다.

### 플러그인 아키텍처와 의존성 주입

도구 시스템은 본질적으로 플러그인 아키텍처다. 각 도구는 `buildTool()` 팩토리 함수로 구성되어, 통합된 `Tool` 인터페이스를 구현하지만 서로 완전히 분리되어 있다. 새 도구를 추가하려면:

1. 도구 디렉토리 생성 (예: `tools/MyTool/`)
2. `ToolDef` 인터페이스 구현
3. `tools.ts`의 `getAllBaseTools()`에 등록

도구 자체는 서로 의존하지 않으며(순환 의존은 lazy require로 해소), 모두 `ToolUseContext`에 의존한다—이것은 전체 실행 체인에 걸쳐 전달되는 컨텍스트 객체로, 권한 상태, 메시지 이력, 앱 상태 등을 포함한다.

```typescript
// Tool.ts:167-172 (단순화)
export type ToolUseContext = {
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  contentReplacementState?: ContentReplacementState
  // ...
}
```

`ToolUseContext`의 설계는 전형적인 의존성 주입이다: 도구 실행 시 필요한 모든 외부 의존성이 context를 통해 전달되며, 도구 자체는 상태 없는 순함수 컴포넌트다. 이로써 테스트, 격리, 서브에이전트 실행이 모두 가능해진다.

### LLM에서 Function Calling의 역할

Claude Code는 Anthropic API의 Function Calling 프로토콜을 따른다. LLM은 추론 과정에서 `tool_use` 블록을 출력하여 호출할 도구 이름과 파라미터를 지정하고, 실행 결과가 `tool_result` 블록 형식으로 LLM에 반환되어 다음 라운드 추론의 입력이 된다.

이 루프의 핵심 제약은: **도구 정의(이름 + 입력 schema)가 system prompt에서 LLM에 전송되어야 하며**, 이것이 귀중한 컨텍스트 token을 소비한다는 점이다. 도구 수가 40+으로 늘어나고 MCP 서드파티 도구까지 추가되면 이 오버헤드가 무시할 수 없게 된다—이것이 3.6절에서 설명하는 ToolSearch 지연 로딩 메커니즘을 직접적으로 낳았다.

---

## 3.3 아키텍처와 데이터 구조

### buildTool() 통합 추상화

`buildTool()`은 도구 시스템의 핵심 팩토리 함수로, `Tool.ts:756-769`에 정의되어 있다:

```typescript
// Tool.ts:756-769
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

이 함수는 한 가지를 한다: 사용자가 제공한 `ToolDef`(선택적 필드 생략 허용)와 `TOOL_DEFAULTS`(안전한 기본값)를 합쳐 완전한 `Tool`을 반환한다.

기본값 (`Tool.ts:729-742`)은 **실패 안전** (fail-closed) 설계 철학을 반영한다:

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // 기본적으로 동시 실행 안전하지 않음
  isReadOnly: (_input?) => false,            // 기본적으로 쓰기가 있다고 가정
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // 기본 허용, 일반 권한 시스템이 처리
  toAutoClassifierInput: (_input?) => '',    // 기본적으로 보안 분류기 건너뜀
  userFacingName: (_input?) => '',
}
```

`isConcurrencySafe` 기본값이 `false`라는 점에 주목하라—시스템이 두 도구를 직렬로 실행하더라도 부작용이 있을 수 있는 작업을 동시에 실행하는 위험을 감수하지 않겠다는 의미다. `isConcurrencySafe: () => true`를 명시적으로 선언한 도구(예: GrepTool, GlobTool 등의 읽기 전용 도구)만 병렬로 스케줄된다.

### 도구의 핵심 타입 정의

`Tool` 인터페이스의 메서드는 몇 가지 기능 도메인으로 분류할 수 있다 (`Tool.ts:297-580`):

**실행 도메인**
- `call(args, context, canUseTool, parentMessage, onProgress)` — 핵심 실행 메서드, `Promise<ToolResult<Output>>` 반환
- `validateInput(input, context)` — 실행 전 검증, `ValidationResult` 반환
- `checkPermissions(input, context)` — 권한 확인, 일반 권한 시스템과 독립

**설명 도메인** (도구 자기 기술 능력)
- `description(input, options)` — 간단한 설명, API의 tools 목록에 사용
- `prompt(options)` — 완전한 시스템 프롬프트, 모델에게 이 도구 사용 방법 알림
- `searchHint` — 3-10 단어의 능력 요약, ToolSearch 키워드 매칭 전용

**렌더링 도메인** (React 컴포넌트, REPL 모드만)
- `renderToolUseMessage(input, options)` — 도구 호출 시작 시의 UI
- `renderToolResultMessage(content, progressMessages, options)` — 도구 결과의 UI
- `renderToolUseProgressMessage(progressMessages, options)` — 실행 중 진행 UI
- `renderToolUseRejectedMessage(input, options)` — 거부 시 UI

**메타데이터 도메인**
- `isConcurrencySafe(input)` — 동시 실행 가능 여부 선언
- `isReadOnly(input)` — 읽기 전용 여부 선언 (권한 판단에 영향)
- `isDestructive(input)` — 되돌릴 수 없는 작업 여부 선언 (삭제, 덮어쓰기, 전송)
- `shouldDefer` — 지연 로딩 여부 (ToolSearch가 필요에 따라 로딩)
- `alwaysLoad` — 항상 prompt에 로딩 (지연 없음)
- `maxResultSizeChars` — 도구 결과를 디스크에 지속화하는 트리거 임계값

`ToolResult<T>`의 구조 (`Tool.ts:289-298`)도 주목할 만하다:

```typescript
// Tool.ts:289-298
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

`contextModifier`는 도구 실행 후 컨텍스트를 수정할 수 있게 한다(하지만 주석에 명확히 적혀 있다: **동시 실행 안전하지 않은 도구만 contextModifier를 실행한다**—이것은 중요한 동시성 안전 제약이다).

### 도구 등록과 발견 메커니즘

`tools.ts`의 `getAllBaseTools()`는 도구 등록의 단일 진실 소스다 (`tools.ts:108-186`). 이 함수는 현재 환경에서 사용 가능한 모든 내장 도구를 반환하며, 다층 조건으로 도구의 가용성을 제어한다:

**환경 조건** (process.env):
- `USER_TYPE === 'ant'` — Anthropic 내부 도구 (ConfigTool, TungstenTool, REPLTool)
- `NODE_ENV === 'test'` — 테스트 도구 (TestingPermissionTool)
- `ENABLE_LSP_TOOL` — LSP 통합 도구
- `CLAUDE_CODE_VERIFY_PLAN` — 계획 검증 도구

**Feature Flag 조건** (`feature()` from `bun:bundle`):
- `PROACTIVE` / `KAIROS` — SleepTool (능동적 동작)
- `AGENT_TRIGGERS` — ScheduleCronTool 등 스케줄 도구
- `COORDINATOR_MODE` — 조율자 모드 관련 도구
- `WEB_BROWSER_TOOL` — 브라우저 도구
- `WORKFLOW_SCRIPTS` — 워크플로 도구

**런타임 조건**:
- `isToolSearchEnabledOptimistic()` — ToolSearchTool 추가 여부
- `isTodoV2Enabled()` — 작업 관리 도구 집합 추가 여부
- `isAgentSwarmsEnabled()` — 팀 협업 도구 추가 여부
- `hasEmbeddedSearchTools()` — 내장 bfs/ugrep이 있으면 GlobTool/GrepTool 추가 안 함

도구의 **중복 제거와 정렬** (`assembleToolPool()`, `tools.ts:218-248`)은 정교하게 설계된 전략을 채택한다: 내장 도구와 MCP 도구를 각각 정렬 후 이어 붙이며, 내장 도구가 접두사, MCP 도구가 뒤에 추가된다. 이것은 시스템 프롬프트의 안정성(prompt cache stability)을 위해서다—Anthropic 서버가 고정된 위치에 캐시 브레이크포인트를 설정하므로, 내장 도구와 MCP 도구를 혼합 정렬하면 새로운 MCP 도구가 추가될 때마다 캐시가 깨진다.

---

## 3.4 도구 분류 목록

`tools/` 디렉토리 구조와 `tools.ts` 등록 로직을 기반으로 완전한 도구 목록을 정리했다:

### 파일 조작 (File Operations)

| 도구 | 디렉토리 | 기능 | 동시 실행 안전 |
|------|------|------|---------|
| FileReadTool | `FileReadTool/` | 파일 읽기, PDF/이미지/Notebook 지원, 페이지 읽기 | 예 |
| FileEditTool | `FileEditTool/` | 정밀 문자열 교체, replace_all 지원 | 아니오 |
| FileWriteTool | `FileWriteTool/` | 파일 쓰기/생성 | 아니오 |
| GlobTool | `GlobTool/` | glob 패턴으로 파일 찾기 | 예 |
| GrepTool | `GrepTool/` | ripgrep 정규식 내용 검색 | 예 |
| NotebookEditTool | `NotebookEditTool/` | Jupyter Notebook 셀 편집 | 아니오 |

### 코드 실행 (Execution)

| 도구 | 디렉토리 | 기능 | 비고 |
|------|------|------|------|
| BashTool | `BashTool/` | Shell 명령 실행, 백그라운드 작업, 샌드박스 지원 | 핵심 도구 |
| PowerShellTool | `PowerShellTool/` | Windows PowerShell 실행 | 조건부 활성화 |
| REPLTool | `REPLTool/` | 격리 VM 환경의 REPL 실행 | Ant 내부 |

### Agent 조율 (Agent Orchestration)

| 도구 | 디렉토리 | 기능 |
|------|------|------|
| AgentTool | `AgentTool/` | 서브에이전트 시작, 병렬 실행 지원 |
| SendMessageTool | `SendMessageTool/` | 다른 Agent에 메시지 전송 |
| TeamCreateTool | `TeamCreateTool/` | Agent 팀 생성 |
| TeamDeleteTool | `TeamDeleteTool/` | Agent 팀 삭제 |
| TaskCreateTool | `TaskCreateTool/` | 백그라운드 작업 생성 |
| TaskGetTool | `TaskGetTool/` | 작업 상태 가져오기 |
| TaskUpdateTool | `TaskUpdateTool/` | 작업 상태 업데이트 |
| TaskListTool | `TaskListTool/` | 모든 작업 나열 |
| TaskStopTool | `TaskStopTool/` | 작업 중지 |
| TaskOutputTool | `TaskOutputTool/` | 작업 출력 가져오기 |

### 컨텍스트와 도구 발견 (Context & Discovery)

| 도구 | 디렉토리 | 기능 |
|------|------|------|
| SkillTool | `SkillTool/` | Skill 로딩 및 실행 (~/.claude/skills/) |
| ToolSearchTool | `ToolSearchTool/` | 지연 로딩 도구 검색 |
| MCPTool (동적 생성) | `MCPTool/` | MCP 서버 도구 (런타임 동적 등록) |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | MCP 리소스 나열 |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | MCP 리소스 읽기 |
| LSPTool | `LSPTool/` | LSP 언어 서버 통합 |

### 계획 및 상태 (Planning & State)

| 도구 | 디렉토리 | 기능 |
|------|------|------|
| EnterPlanModeTool | `EnterPlanModeTool/` | 계획 모드 진입 (읽기 전용, 실행 안 함) |
| ExitPlanModeTool | `ExitPlanModeTool/` | 계획 모드 종료 |
| EnterWorktreeTool | `EnterWorktreeTool/` | git worktree 격리 환경 진입 |
| ExitWorktreeTool | `ExitWorktreeTool/` | worktree 환경 종료 |
| TodoWriteTool | `TodoWriteTool/` | Todo 목록 쓰기 (사이드바에 표시) |
| BriefTool | `BriefTool/` | 세션 요약 생성 |

### 네트워크 접근 (Network)

| 도구 | 디렉토리 | 기능 |
|------|------|------|
| WebFetchTool | `WebFetchTool/` | HTTP 가져오기, HTML→Markdown 변환, 도메인 보안 확인 |
| WebSearchTool | `WebSearchTool/` | 네트워크 검색 |

### 시스템 및 스케줄링 (System & Scheduling)

| 도구 | 디렉토리 | 기능 | 조건 |
|------|------|------|------|
| ConfigTool | `ConfigTool/` | 설정 읽기/쓰기 | Ant 내부 |
| SleepTool | `SleepTool/` | 대기 (능동적 모드) | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | 합성 출력 (특수 목적) | — |
| ScheduleCronTool | `ScheduleCronTool/` | 스케줄 작업 생성/삭제/나열 | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | 원격 트리거 | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | 사용자에게 질문 (인터랙티브) | — |

---

## 3.5 도구 실행 흐름

### LLM tool_use에서 도구 실행까지의 전체 흐름

도구 실행의 진입점은 `services/tools/toolExecution.ts`의 `runToolUse()` 함수 (`toolExecution.ts:298-428`)이며, 이것은 async generator다:

```
LLM이 tool_use 블록 출력
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — 도구 찾기, 별칭 지원 (이름이 변경된 도구의 하위 호환)
    ↓
abortController.signal.aborted? → CANCEL_MESSAGE 반환
    ↓
streamedCheckPermissionsAndCallTool() [AsyncIterable 반환]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — Zod 타입 검증
  2. tool.validateInput(input, context)  — 도구 자체 검증
  3. runPreToolUseHooks()                — PreToolUse 훅 실행
  4. canUseTool()                        — 권한 확인 (UI 확인 팝업 가능)
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — 대용량 결과 지속화
  7. runPostToolUseHooks()               — PostToolUse 훅 실행
    ↓
yield MessageUpdateLazy (tool_result 포함)
    ↓
다음 라운드 LLM 추론
```

중요한 하위 호환 설계 (`toolExecution.ts:350-360`): 도구 이름이 변경될 때 이전 이름이 `aliases`로 유지된다. `options.tools`에서 도구를 찾을 수 없을 때 시스템이 `getAllBaseTools()`에서 별칭 매칭을 검색하여, 이전 transcript의 이전 도구 이름도 여전히 실행될 수 있도록 보장한다.

### 스트리밍 도구 실행 (Streaming Tool Execution)

도구 실행은 `Stream<MessageUpdateLazy>`를 통해 스트리밍된다 (`toolExecution.ts:500-535`):

```typescript
// toolExecution.ts:500-535 (단순화)
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // 진행 메시지
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // 최종 결과
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

스트리밍 설계의 의미: UI가 도구가 아직 실행 중일 때 실시간으로 진행 상황을 표시할 수 있다(예: BashTool의 실시간 출력, AgentTool의 서브에이전트 진행 상황). 진행 메시지와 최종 결과가 동일한 `Stream` 파이프를 통해 전달되어 소비자 코드를 단순화한다.

### 동시 도구 실행

Claude Code는 LLM이 단일 응답에서 여러 `tool_use` 블록을 출력하고 병렬로 실행하는 것을 지원한다. 동시 실행의 전제는: **모든 도구가 `isConcurrencySafe: () => true`를 선언해야 한다**.

병렬 실행 시 `contextModifier`는 실행되지 않는다(`ToolResult` 주석에 명시: "contextModifier is only honored for tools that aren't concurrency safe"). 이것은 중요한 보안 제약이다: 전역 컨텍스트를 수정하는 작업은 동시 환경에서 진행될 수 없다.

전형적인 동시 실행 안전 도구: GrepTool, GlobTool, FileReadTool(모두 `isConcurrencySafe: () => true` 선언).

---

## 3.6 ToolSearch — 지연 로딩 메커니즘

### 왜 ToolSearch가 필요한가 (프롬프트 팽창 문제)

각 도구의 정의(이름 + JSON Schema + 설명)가 LLM에 전송될 때 token을 소비한다. 도구 수가 특정 임계값(실험 결과 약 40-60개)을 초과하면:

1. **token 비용 상승**: 매 API 호출이 많은 양의 도구 정의를 전달
2. **어텐션 희석**: LLM이 수십 개의 도구를 마주할 때 각 도구에 대한 어텐션이 떨어질 수 있음
3. **prompt cache 무효화 위험**: 도구 목록 변경(예: MCP 도구가 동적으로 추가)이 캐시를 무효화

ToolSearch의 해결책은 **필요에 따라 로딩**: 대부분의 도구가 `shouldDefer: true`로 표시되어 초기 프롬프트에 완전한 schema가 전송되지 않으며, 검색으로 발견된 후에야 로딩된다.

### deferred tools의 등록과 발견

도구는 `shouldDefer` 필드로 지연 로딩 여부를 선언한다 (`Tool.ts:456-462`):

```typescript
// Tool.ts:456-462
readonly shouldDefer?: boolean

/**
 * When true, this tool is never deferred — its full schema appears in the
 * initial prompt even when ToolSearch is enabled. For MCP tools, set via
 * `_meta['anthropic/alwaysLoad']`. Use for tools the model must see on
 * turn 1 without a ToolSearch round-trip.
 */
readonly alwaysLoad?: boolean
```

`isDeferredTool()` 함수(`tools/ToolSearchTool/prompt.ts`에 정의)가 도구의 지연 여부를 판단한다: `shouldDefer: true`가 설정되고 `alwaysLoad: true`가 없는 도구는 deferred로 표시된다.

ToolSearchTool 자체는 **절대 지연되지 않는다**—첫 번째 라운드에 반드시 사용 가능해야 다른 도구를 발견할 수 있다.

### 필요에 따른 로딩의 구현

ToolSearchTool의 `call()` 메서드 (`ToolSearchTool.ts:221-302`)는 두 가지 쿼리 모드를 지원한다:

**직접 선택 모드** (`select:` 접두사):
```
query: "select:NotebookEdit"     → NotebookEditTool 직접 반환
query: "select:Read,Edit,Grep"   → 여러 도구 일괄 선택
```

**키워드 검색 모드**:
```
query: "jupyter notebook"        → 키워드 매칭, NotebookEditTool 등 반환
query: "mcp__github"             → MCP 서버 접두사 매칭
```

검색 점수 알고리즘 (`ToolSearchTool.ts:155-198`):

```
도구명 정확 매칭 부분 (MCP): +12 점
도구명 정확 매칭 부분 (일반): +10 점
도구명 부분 포함 키워드 (MCP): +6 점
도구명 부분 포함 키워드 (일반): +5 점
도구명 전체 매칭 fallback: +3 점
searchHint 단어 경계 매칭: +4 점 (정교하게 설계된 능력 요약, 신호가 강함)
설명 텍스트 단어 경계 매칭: +2 점
```

`searchHint` 필드의 가중치(+4점)가 설명 텍스트(+2점)보다 높아, 도구 개발자가 정확한 능력 요약을 제공하도록 장려한다. 예를 들어 GrepTool의 `searchHint: 'search file contents with regex (ripgrep)'`, FileEditTool의 `searchHint: 'modify file contents in place'`.

검색 결과는 `tool_reference` 블록을 통해 LLM에 반환된다 (`ToolSearchTool.ts:330-352`). 이것은 Anthropic API의 특별 확장으로, 서버에 "이 도구들의 완전한 schema를 현재 대화의 도구 목록에 주입해 달라"고 알린다.

---

## 3.7 도구 결과 저장

### 디스크 저장 전략

도구 실행 결과가 매우 클 수 있다(예: 10MB 로그 파일 읽기, 많은 출력을 생성하는 명령 실행). 대용량 결과를 직접 메시지 이력에 넣으면 token이 낭비되고 후속 요청의 context가 팽창한다.

`utils/toolResultStorage.ts`는 **필요에 따른 지속화** 전략을 구현한다:

1. 결과 크기 계산 (`contentSize()`)
2. 도구의 `maxResultSizeChars` 임계값과 비교 (`getPersistenceThreshold()`로 파싱)
3. 임계값을 초과하는 결과를 `~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`에 쓰기
4. 파일 경로 + 미리보기가 포함된 메시지로 대체

```typescript
// toolResultStorage.ts:168-177
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  let message = `${PERSISTED_OUTPUT_TAG}\n`
  message += `Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}\n\n`
  message += `Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):\n`
  message += result.preview
  message += result.hasMore ? '\n...\n' : '\n'
  message += PERSISTED_OUTPUT_CLOSING_TAG
  return message
}
```

`PREVIEW_SIZE_BYTES = 2000` (약 2KB), 미리보기는 마지막 줄 바꿈까지 잘라내어 행 중간에 잘리는 것을 방지한다.

중요한 **멱등성** 설계 (`toolResultStorage.ts:145-158`): 쓸 때 `{ flag: 'wx' }`(배타적 생성)를 사용하여, 파일이 이미 존재하면 쓰기 에러를 무시하고 기존 파일로 미리보기를 생성한다. 이것은 microcompact가 이전 메시지를 재생할 때 중복 쓰기가 되지 않으며, EEXIST 에러도 발생하지 않도록 보장한다.

FileReadTool에는 특별한 처리가 있다: `maxResultSizeChars: Infinity`—읽기 도구의 결과는 절대 디스크에 지속화되지 않는다. 이유는 주석에 설명되어 있다: "persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits" (지속화하면 순환이 발생: Read가 파일을 읽고, 결과가 너무 커서 파일로 지속화되고, 모델이 다시 Read로 그 파일을 읽고...).

### Token 예산 관리

`toolResultStorage.ts`는 더 거시적인 **메시지 수준의 도구 결과 예산(per-message aggregate budget)**도 구현한다. 이것은 `ContentReplacementState` 메커니즘에 의해 구동된다 (`toolResultStorage.ts:395-440`):

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // 예산 확인을 거친 tool_use_id (결과 동결)
  replacements: Map<string, string>  // 대체된 ID → 대체 후 내용 문자열
}
```

핵심 제약: **결과가 한 번 판단되면(대체되든 아니든) 절대 변경되지 않는다** (`seenIds` 세트로 보장). 이것은 prompt cache 안정성을 위해서다—같은 tool_use_id의 처리 결과가 세션 전체에서 일관되어야 캐시가 내용 변경으로 인해 무효화되지 않는다.

예산 제한은 GrowthBook feature flag `tengu_hawthorn_window`로 동적으로 제어되며, 하나의 메시지에서 도구 결과 총량이 예산을 초과할 때 시스템이 가장 큰 도구 결과를 디스크 지속화 버전으로 대체하여 총량이 예산 이하로 떨어질 때까지 반복한다.

---

## 3.8 설계 결정 분석

### 자기 기술 vs 외부 등록의 tradeoff

Claude Code는 **자기 기술** 모드(각 도구가 자신의 schema, 설명, 프롬프트, 렌더링 로직을 가짐)를 선택했으며, 이런 정보를 중앙 레지스트리에 집중시키지 않았다.

장점:
- **도구가 완전히 자체 포함**: 새 도구를 추가할 때 디렉토리 하나만 필요하며, 중앙 레지스트리의 로직을 수정할 필요 없음
- **설명이 동적으로 생성 가능**: `description()`과 `prompt()`는 비동기 함수여서, 환경, 권한, 설치 상태에 따라 내용을 동적으로 조정 가능
- **렌더링 로직이 도구와 공존**: React 렌더링 컴포넌트가 도구 파일 바로 옆에 있어, 도구 동작과 UI를 바꾸는 것이 같은 PR

단점:
- **도구 인터페이스 비대화**: `Tool` 타입에 40+ 개의 메서드/필드가 있어, 새 도구 작성자가 많은 인터페이스 세부 사항을 알아야 함
- **코드 중복**: 각 도구에 `renderToolUseMessage`, `renderToolResultMessage` 등의 렌더링 메서드가 있으며, 패턴이 매우 유사
- **`buildTool()`이 완전히 제거할 수 없음**: 기본값을 제공하지만, 여전히 많은 메서드를 각 도구가 직접 구현해야 함

실제로 Claude Code는 **공유 UI 컴포넌트**(예: `tools/shared/`)와 **패턴 추출**(예: `lazySchema()`)로 코드 중복을 완화하지만, 근본적인 인터페이스 복잡도는 여전히 존재한다.

### 왜 일부 도구를 지연 로딩하는가

ToolSearch의 지연 로딩 결정은 하나의 원칙을 따른다: **첫 번째 라운드 대화에서 필요하지 않을 수 있는 도구는 모두 지연해야 한다**.

`alwaysLoad` 도구(절대 지연 안 함)는: 첫 번째 라운드부터 모델이 존재를 알아야 한다. 전형적인 예는 AgentTool, BashTool, FileReadTool—이것들은 모든 프로그래밍 작업의 기초 도구다.

`shouldDefer` 도구(지연 로딩)는 보통: 특정 시나리오에서만 필요한 도구(NotebookEditTool은 Jupyter 작업에서만 필요), 많은 MCP 도구(사용자가 수십 개의 MCP 서버를 설치했지만 각 대화에서 일부만 사용).

MCP 도구는 기본적으로 도구 수에 따라 ToolSearch 메커니즘이 트리거되지만, 도구 메타데이터에 `_meta['anthropic/alwaysLoad']`를 설정하면 강제로 지연하지 않을 수 있다.

### 도구 권한의 계층적 설계

도구 권한은 **3층 방어** 설계를 채택한다:

1. **Zod 타입 검증** (`checkPermissionsAndCallTool` 첫 번째 단계): 도구의 inputSchema가 파라미터 타입을 엄격히 검증하며, LLM이 잘못된 타입의 파라미터를 생성하면 거부되고 에러 메시지 반환
2. **도구 자체 검증** (`validateInput()`): 도구가 자신의 비즈니스 로직 검증을 구현, 예를 들어 FileEditTool이 old_string과 new_string이 다른지 확인하고 파일 크기가 1GiB를 초과하지 않는지 확인
3. **일반 권한 시스템** (`canUseTool()` + `checkPermissions()`): 사용자가 설정한 allow/deny 규칙, 도구의 읽기 전용 여부, 파괴적 작업 여부 등을 기반으로 최종 판단, 인터랙티브 확인 팝업이 뜰 수 있음

이 3층은 순서대로 실행되며, 어느 층에서 실패해도 단락(short-circuit)되어 다음 층에 진입하지 않는다.

---

## 3.9 이식 가능한 패턴

### 자기 기술 도구 패턴의 범용 설계

Claude Code의 도구 시스템에서 추출된 가장 이식 가치가 높은 패턴은: **도구가 곧 자체 포함 플러그인**이다.

핵심 원칙:
1. **Schema가 곧 문서가 곧 검증기**: Zod schema로 입력을 정의하여, LLM을 위한 JSON Schema를 자동 생성하고, 런타임에 LLM의 출력을 검증
2. **팩토리 함수 + 안전 기본값**: `buildTool()`이 실패 안전 기본 동작을 제공(기본적으로 동시 실행 안전하지 않음, 기본적으로 읽기 전용 아님), 도구 개발자가 자신의 예외만 선언
3. **searchHint 간결 요약**: 3-10 단어의 능력 설명, 키워드 검색에 특화되어 완전한 설명과 분리
4. **능력 선언이 런타임 판단보다 우선**: `isReadOnly()`, `isConcurrencySafe()`, `isDestructive()`로 스케줄러가 도구를 실행하지 않고도 스케줄링 결정을 내릴 수 있음

### Doramagic의 도구 시스템 (Brick 시스템)이 참고할 수 있는 점

Doramagic의 Brick 시스템(278+ 적목)과 Claude Code의 도구 시스템은 깊은 유사성이 있지만, 본질적인 차이도 있다:

**유사점**:
- 모두 "플러그인식" 아키텍처: 각 Brick/Tool이 자체 포함 기능 단위
- 모두 설명 메커니즘이 필요: LLM이 언제 어떤 도구/적목을 사용해야 하는지 알게 함
- 모두 분류 체계가 있음: 기능 영역별로 조직

**참고할 구체적 패턴**:

1. **`searchHint` 유사 Brick의 tags**: Claude Code는 각 도구에 3-10 단어의 간결한 능력 설명을 제공하여 검색 매칭에 특화했다. Doramagic의 적목은 현재 tags와 categories로 조직되어 있는데, `hint` 필드를 추가하여 모델의 적목 발견 효율을 최적화할 수 있다.

2. **지연 로딩 → 적목 필요에 따라 활성화**: Claude Code의 deferred tools 메커니즘은 모든 적목을 시스템 프롬프트에 넣는 것이 좋은 방법이 아님을 보여준다. Doramagic은 `shouldDefer`의 설계를 참고하여, 자주 사용하지 않는 적목(도메인 특화 적목)을 지연 로딩으로 설정하고 모델이 명시적으로 필요로 할 때만 활성화할 수 있다.

3. **`maxResultSizeChars` → 적목 출력 예산**: 각 도구가 자신의 결과 최대 token 예산을 선언하고, 초과하면 압축한다. Doramagic의 적목 출력(추출된 지식 JSON)도 매우 클 수 있으므로, 이 메커니즘을 참고하여 "요약 우선, 세부 사항 필요에 따라" 출력 전략을 구현할 수 있다.

4. **`isConcurrencySafe` → 적목 병렬 선언**: Doramagic의 지식 추출 파이프라인에서 여러 적목이 동시에 같은 코드베이스에 작동할 수 있다. 적목의 동시 실행 안전성을 명확히 선언하면, 스케줄러가 어떤 적목을 병렬로 실행하고 어떤 것을 직렬로 실행할지 자동으로 결정할 수 있다.

5. **3층 권한 방어 → 적목 실행 보안**: Doramagic이 OpenClaw Skill로 실행되는 시나리오에서, Brick 실행의 합법성 검증이 이 3층 설계를 참고할 수 있다: schema 검증 → 비즈니스 검증 → 플랫폼 권한 확인.

**본질적 차이**: Claude Code의 도구는 주로 **확정적 작업**(파일 읽기, 명령 실행)에 면하며, 출력이 정밀하게 정의될 수 있다. Doramagic의 적목은 **지식 추출**에 면하며, 출력이 의미론적이다. 즉 Doramagic은 Claude Code처럼 Zod schema로 적목 출력을 엄격히 검증할 수 없다—이것이 바로 Doramagic의 "코드는 사실을 말하고, AI는 이야기를 말한다"는 아키텍처 원칙의 의미: 확정적인 뼈대(facts 추출)는 schema로 제약할 수 있지만, 비확정적인 해석(stories 생성)은 그럴 필요가 없다.

---

## 3.10 소스 인덱스

| 파일 | 행수 | 핵심 내용 |
|------|------|---------|
| `src/Tool.ts` | 792 | `Tool` 타입 정의, `buildTool()`, `ToolUseContext`, `ToolResult`, `TOOL_DEFAULTS` |
| `src/tools.ts` | 389 | `getAllBaseTools()`, `getTools()`, `assembleToolPool()`, `getMergedTools()`, `filterToolsByDenyRules()` |
| `src/services/tools/toolExecution.ts` | 1,745 | `runToolUse()`, `checkPermissionsAndCallTool()`, `streamedCheckPermissionsAndCallTool()`, `buildSchemaNotSentHint()` |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 471 | `searchToolsWithKeywords()`, `parseToolName()`, 키워드 점수 알고리즘, `select:` 접두사 직접 선택 |
| `src/utils/toolResultStorage.ts` | 1,040 | `persistToolResult()`, `buildLargeToolResultMessage()`, `ContentReplacementState`, `enforceToolResultBudget()` |
| `src/tools/BashTool/BashTool.tsx` | ~1,800+ | `isSearchOrReadBashCommand()`, 샌드박스, 백그라운드 작업, 진행 표시 |
| `src/tools/FileEditTool/FileEditTool.ts` | ~500+ | 문자열 교체, 대용량 파일 보호 (1GiB 한계), 비밀 감지 |
| `src/tools/FileReadTool/FileReadTool.ts` | ~600+ | 다중 형식 지원 (PDF/이미지/Notebook), token 계산, `maxResultSizeChars: Infinity` |
| `src/tools/GrepTool/GrepTool.ts` | ~400+ | ripgrep 통합, head_limit/offset 페이지 매김, `DEFAULT_HEAD_LIMIT = 250` |
| `src/tools/WebFetchTool/utils.ts` | ~450+ | 도메인 블랙리스트 확인, LRU 캐시 (50MB/15분), HTML→Markdown 변환 |
| `src/tools/MCPTool/classifyForCollapse.ts` | ~350 | MCP 도구의 search/read 분류 (Slack/GitHub/Linear/Jira 등 20+ 서비스 사전 규칙) |

**핵심 상수** (여러 파일에 분산):
- `PREVIEW_SIZE_BYTES = 2000` (toolResultStorage.ts) — 대용량 결과 미리보기 크기
- `DEFAULT_HEAD_LIMIT = 250` (GrepTool.ts) — grep 기본 결과 상한
- `MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024` (WebFetchTool/utils.ts) — 네트워크 가져오기 10MB 상한
- `FETCH_TIMEOUT_MS = 60_000` (WebFetchTool/utils.ts) — HTTP 요청 60초 타임아웃
- `CACHE_TTL_MS = 15 * 60 * 1000` (WebFetchTool/utils.ts) — URL 캐시 15분
- `PROGRESS_THRESHOLD_MS = 2000` (BashTool.tsx) — 2초 초과하면 진행 표시
- `MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024` (FileEditTool.ts) — 파일 편집 1GiB 한계
