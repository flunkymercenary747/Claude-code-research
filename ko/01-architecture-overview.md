# 1장: 아키텍처 총람과 시작 흐름

> 데이터 출처: Claude Code TypeScript 소스 스냅샷 (2026-03-31)
> 소스 경로 (mini): `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 개요 및 위치

**Claude Code란 무엇인가:** Claude Code는 터미널에서 실행되는 AI 프로그래밍 어시스턴트로, React/Ink로 인터랙티브한 TUI(Terminal User Interface)를 렌더링하며, REPL 루프를 통해 Claude API를 구동하여 코드 편집, 명령 실행, 파일 조작 등의 개발 작업을 수행한다.

### 기술 스택 개요

| 계층 | 기술 | 용도 |
|------|------|------|
| 런타임 | Bun(주요) / Node.js 18+(호환) | JavaScript 실행 환경 |
| 언어 | TypeScript | 전 프로젝트 엄격한 타입 |
| UI 프레임워크 | React + Ink | 터미널 TUI 렌더링 |
| CLI 프레임워크 | Commander.js (`@commander-js/extra-typings`) | 커맨드라인 인수 파싱 |
| API 클라이언트 | `@anthropic-ai/sdk` | Claude API 호출 |
| MCP 통합 | `@modelcontextprotocol/sdk` | MCP 서버 프로토콜 |
| 기능 플래그 | GrowthBook + `bun:bundle` feature flags | A/B 테스트와 DCE |
| 원격 측정 | OpenTelemetry (지연 로딩 ~400KB) | 지표/로그/추적 |
| 검증 | Zod v4 | 런타임 schema 검증 |

### 코드 규모 통계

- **총 행수**: 512,664 행 (`.ts` + `.tsx` 파일)
- **파일 수**: 1,884 개의 TypeScript 파일
- **최상위 디렉토리 수**: 35 개

주요 디렉토리별 LOC 비율:

```
utils/       180,472 행  (35.2%)  — 유틸리티 함수, 권한, 인증, 설정 등
components/   81,546 행  (15.9%)  — React UI 컴포넌트
services/     53,680 행  (10.5%)  — API, MCP, 분석, 메모리 등 서비스
tools/        50,828 행  (9.9%)   — 30 개의 도구 구현 (Bash/File/Agent 등)
commands/     26,428 행  (5.2%)   — slash 명령어 구현
screens/       5,977 행  (1.2%)   — REPL 등 최상위 화면
bootstrap/     ~5,000 행  (1.0%)  — 전역 상태 (state.ts 1,758 행)
entrypoints/   ~3,000 행  (0.6%)  — CLI/SDK/MCP 진입점
main.tsx       4,683 행  (0.9%)   — 주 진입점 조정자
setup.ts         477 행  (0.1%)   — 초기화 설정
```

---

## 1.2 이론적 기반

### 커맨드라인 애플리케이션의 아키텍처 패턴

Claude Code는 두 가지 고전적 CLI 아키텍처 패턴을 융합했다:

**REPL Loop (Read-Eval-Print Loop)**
전통적인 REPL은 동기 루프에서 입력을 읽고, 평가하고, 출력을 인쇄한다. Claude Code는 이를 비동기 이벤트 구동 REPL로 업그레이드했다: 입력은 React 컴포넌트가 캡처하고, "평가"는 Claude API round-trip(다수의 도구 호출 포함)이며, 출력은 React/Ink reconciler를 통해 터미널에 렌더링된다.

**Event-Driven Architecture**
시작 시 모든 초기화가 완료될 때까지 차단하여 기다리지 않는다—MDM 읽기, Keychain 사전 가져오기, MCP 연결, plugin hook 로딩이 모두 fire-and-forget 방식으로 병렬 트리거된다(1.4절 참조). 이는 TTFR(Time To First Render)을 최소화하며, 웹 애플리케이션의 Critical Rendering Path 최적화 사고방식과 일치한다.

### 터미널 UI 프레임워크의 설계 철학: React in Terminal

Ink는 React의 컴포넌트 모델, 선언적 상태, reconciliation 메커니즘을 터미널로 이식했다. 핵심 아이디어:

- **가상 DOM → 가상 터미널 버퍼**: 매 state 변화가 diff를 트리거하고, 변화된 문자 행만 다시 그려 깜박임을 방지
- **Flexbox → 터미널 레이아웃**: CSS Yoga 엔진으로 열 너비와 줄 바꿈을 계산하여 터미널 UI를 JSX로 선언적으로 기술할 수 있게 함
- **컴포넌트 재사용**: Loading spinner, 확인 팝업, Diff 표시 등의 UI 로직을 테스트 가능한 React 컴포넌트로 캡슐화

이로 인해 Claude Code의 UI 코드가 웹 프론트엔드 코드와 인지적 프레임워크를 공유하며, `components/` 디렉토리의 81,546행 코드를 익숙한 React 패턴으로 이해할 수 있다.

### 플러그인 아키텍처의 이론적 기반

Claude Code의 플러그인 시스템은 "능력 등록" 모델(Capability Registration Pattern)을 기반으로 한다:

- 도구(Tools), 명령(Commands), Hooks가 모두 시작 시 전역 레지스트리에 등록됨
- 플러그인이 파일 시스템 관례(`~/.claude/plugins/`)를 통해 도구/명령 목록을 확장
- `bun:bundle`의 `feature()` 함수가 컴파일 시점에 Dead Code Elimination(DCE)을 수행하여, 실험적 기능이 외부 빌드 산출물에 나타나지 않도록 함

---

## 1.3 전체 아키텍처 다이어그램

### 계층 아키텍처 (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                    진입점 계층 (Entry Layer)               │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (CLI 인터랙션)  (Commander.js 라우팅)  (MCP 서버 모드)     │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  부트스트랩 계층 (Bootstrap Layer)          │
│    setup.ts      │    entrypoints/init.ts                 │
│  (세션 초기화)     │    bootstrap/state.ts                 │
│  (worktree/tmux)  │    (전역 상태 싱글톤)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  UI 계층 (Ink/React TUI)                  │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (주 인터랙션 화면)  │  components/ (81K LOC)               │
│  replLauncher.tsx  │  (입력/출력/팝업/대기 애니메이션)        │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  엔진 계층 (Engine Layer)                  │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (세션 생명주기 관리)  │  (API 호출)  │  (React 상태 트리)    │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  도구 계층 (Tool Layer)                    │
│  tools/ (30 개 도구, 50K LOC)                             │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  서비스 계층 (Service Layer)               │
│  services/ (53K LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (MCP 클라이언트)  (GrowthBook/OTel)         │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (언어 서버)    (세션 메모리)     (엔터프라이즈 관리 설정)      │
└─────────────────────────────────────────────────────────┘
```

### 모듈 의존 관계 개요

```
main.tsx
  ├── entrypoints/init.ts       (memoized, 한 번만 초기화)
  ├── entrypoints/cli.tsx       (Commander 하위 명령 라우팅)
  ├── bootstrap/state.ts        (전역 상태, 순환 의존 엄금)
  ├── setup.ts                  (매 세션 호출)
  ├── QueryEngine.ts            (headless/SDK 경로)
  ├── replLauncher.tsx          (interactive 경로)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (MCP 도구/리소스 로딩)
```

**bootstrap/state.ts의 특별한 위치**: 코드에 명시적 주석 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`가 있으며, ESLint 규칙 `custom-rules/bootstrap-isolation`이 이 파일이 비-리프 모듈에서 임포트되는 것을 방지하여 순환 의존을 막는다.

### 세 가지 진입점 비교

| 진입점 | 파일 | 트리거 방법 | 특징 |
|------|------|----------|------|
| CLI 인터랙션 | `entrypoints/cli.tsx` | `claude` 명령어 | 완전한 REPL + React TUI |
| SDK 헤드리스 | `QueryEngine.ts` | `-p` 플래그 / SDK API | UI 없음, 단일 또는 스트리밍 출력 |
| MCP 서버 | `entrypoints/mcp.ts` | `claude --mcp` | 도구 집합을 MCP server로 노출 |

---

## 1.4 시작 흐름 상세

### main.tsx 전체 시작 시퀀스

`main.tsx`의 4,683행은 순서대로 실행되지 않는다—파일 상단의 import 부작용은 정교하게 편성된 병렬 예열 시퀀스다.

**단계 0: 모듈 로딩 시기 (import 부작용, ~135ms)**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. 성능 기준점

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. 병렬: MDM 자식 프로세스 (plutil/reg query)

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. 병렬: macOS Keychain 사전 읽기 (OAuth + API key)

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // 모든 import 완료
```

주석은 이 세 가지 병렬 작업의 이점을 정확히 설명한다: MDM 읽기는 ~135ms의 모듈 평가 시간을 절약하고, Keychain 사전 읽기는 ~65ms의 순차적 sync spawn을 절약한다. 이것이 Claude Code 시작 최적화의 핵심 기법이다: **ES module의 정적 분석 특성을 활용하여 모듈 그래프 평가 중에 I/O 집약적 작업을 미리 실행**.

**단계 1: Commander 라우팅 (동기)**

`entrypoints/cli.tsx`에서 Commander.js가 argv를 파싱하고, 하위 명령(`chat`, `api`, `mcp`, `resume` 등)이나 플래그에 따라 다른 실행 경로로 분배한다:

```typescript
// entrypoints/cli.tsx (단순화 구조)
async function main(): Promise<void> {
  // 빠른 경로: --version 제로 import
  // 일반 경로: await init() → setup() → 분기 실행
}
```

**단계 2: init() 초기화 (memoized, 한 번만 실행)**

`entrypoints/init.ts`의 `init` 함수가 `memoize`로 감싸져 여러 번 호출해도 한 번만 초기화된다:

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // 설정 시스템 활성화
  applySafeConfigEnvironmentVariables()  // 신뢰 대화 전의 안전한 env vars
  applyExtraCACertsFromConfig()     // TLS 연결 전에 CA 인증서 설정
  setupGracefulShutdown()           // 종료 정리 훅 등록
  // 지연 로딩: OpenTelemetry (~400KB) + gRPC (~700KB)
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // 비동기 캐시
  detectCurrentRepository()          // GitHub repo 감지
  preconnectAnthropicApi()           // TCP+TLS 사전 연결 (~100-200ms overlap)
  configureGlobalMTLS()
  configureGlobalAgents()            // proxy 설정
})
```

**단계 3: setup() 세션 초기화 (매 세션 호출)**

```typescript
// setup.ts — 핵심 단계 시퀀스
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. UDS 메시징 서버 (swarm/ant 모드)
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Terminal 백업 확인 (iTerm2/Terminal.app)
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — 모든 cwd에 의존하는 코드 전에 반드시 먼저
  setCwd(cwd)
  // 4. Hooks 설정 스냅샷 (setCwd() 이후에 반드시)
  captureHooksConfigSnapshot()
  // 5. Worktree 생성 (--worktree인 경우)
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. 백그라운드 작업 등록 (SessionMemory, context collapse)
  if (!isBareMode()) initSessionMemory()
  // 7. Plugin 사전 가져오기 (병렬, 비차단)
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. 분석 sink 활성화 + 첫 번째 원격 측정 이벤트
  initSinks()
  logEvent('tengu_started', {})
  // 9. 릴리스 노트 확인 (인터랙티브 모드)
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**단계 4: REPL 렌더링**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // 지연 로딩 UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

마지막으로 Ink가 터미널을 인수하고, React 컴포넌트 트리가 렌더링을 시작하며, REPL이 준비된다.

### 병렬 사전 가져오기 전략

Claude Code의 시작 최적화는 "**일찍 트리거할수록, 늦게 기다릴수록**" 원칙을 따른다:

| 작업 | 트리거 시점 | 기다리는 시점 |
|------|----------|----------|
| MDM 자식 프로세스 (`plutil/reg query`) | `main.tsx` 첫 번째 import 부작용 | `applySafeConfigEnvironmentVariables()` 호출 전 |
| Keychain 사전 읽기 (OAuth + API key) | `main.tsx` 세 번째 import 부작용 | `ensureKeychainPrefetchCompleted()` |
| Claude API TCP 사전 연결 | `init()` 내 `preconnectAnthropicApi()` | 첫 번째 API 요청 시 자동 연결 재사용 |
| Plugin hooks 로딩 | `setup()` 내 fire-and-forget | `processSessionStartHooks()` 렌더링 전 |
| MCP configs 읽기 | `getClaudeCodeMcpConfigs()` 시작 | 인터랙티브 모드의 `getMcpToolsCommandsAndResources()` |

### 지연 로딩 메커니즘

Claude Code는 시작 핵심 경로의 대형 모듈에 명시적 지연 로딩을 적용했다:

```typescript
// entrypoints/init.ts — OpenTelemetry 지연 로딩 주석
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

또한 `replLauncher.tsx`는 마지막 순간에야 App과 REPL 컴포넌트를 `import`하여, Commander 라우팅이 완료되기 전에 React 트리가 평가되는 것을 방지한다.

`bun:bundle`의 `feature()` 함수는 컴파일 시점 DCE를 구현한다:

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

외부 빌드에서는 이 코드가 완전히 제거되어 번들 크기를 줄인다.

### setup.ts 초기화 단계 상세

`setup.ts`의 477행은 다음과 같은 핵심 제약을 중심으로 전개된다:

1. **`setCwd()` 를 가장 먼저 호출해야 한다**: 이후의 모든 작업(hooks, settings, plugin 로딩)이 올바른 cwd에 의존
2. **Hooks 스냅샷은 반드시 `setCwd()` 이후에**: 올바른 디렉토리에서 `.claude/settings.json`을 읽도록 보장
3. **Worktree 생성은 `getCommands()` 이전에**: 그렇지 않으면 `/eject` 명령을 사용할 수 없음
4. **`initSinks()`는 모든 백그라운드 작업 등록 이후에**: 분석 이벤트 큐가 준비됐음을 보장

`--bare` 모드(스크립트/SDK 헤드리스 호출)는 많은 인터랙티브 기능을 건너뛴다: terminal 백업 확인, plugin hook 사전 가져오기, commit attribution, team memory watcher 등, 스크립트 호출의 시작 오버헤드를 최소화한다.

### bootstrap/state.ts 상태 구축

`state.ts`(1,758행)는 전체 세션의 전역 싱글톤 상태를 유지한다. 핵심 `State` 타입:

```typescript
// bootstrap/state.ts (State 타입 정의, 일부)
type State = {
  originalCwd: string
  projectRoot: string          // 안정적인 프로젝트 루트 디렉토리, worktree가 변경하지 않음
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // 원격 측정 카운터
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // 로그/추적 제공자
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... 총 약 60 개의 필드
}
```

**설계 제약**: 주석 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`가 아키텍처 가드 역할을 한다. ESLint 규칙 `custom-rules/bootstrap-isolation`은 state.ts가 순환 의존을 초래할 수 있는 모듈에서 임포트되는 것을 방지한다. 모든 상태는 setter/getter 함수를 통해 접근하며, 가변 객체를 직접 노출하지 않는다.

---

## 1.5 진입점 분석

### CLI 진입점 (interactive mode)

`entrypoints/cli.tsx`는 가장 복잡한 진입점으로, 모든 사용자 대면 기능 라우팅을 담당한다:

**시작 경로**:
1. Commander.js가 argv를 파싱 → 하위 명령이나 플래그 식별
2. `await init()` 초기화 (memoized)
3. MCP configs, 엔터프라이즈 정책, Chrome 통합 처리
4. `await setup(cwd, permissionMode, ...)` 세션 초기화
5. 모드에 따라 분기:
   - **인터랙티브 모드**: `showSetupScreens()` → `launchRepl()` → React TUI
   - **Print 모드(`-p`)**: `runHeadless()` → `QueryEngine` → stdout
   - **Resume 모드**: `loadConversationForResume()` → 이전 세션 복원
   - **Teleport 모드**: 원격 세션 인수

**핵심 CLI 옵션** (일부):

| 플래그 | 기능 |
|------|------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | 동적 MCP 서버 설정 |
| `--worktree` | git worktree 격리 생성 |
| `--tmux` | tmux session 내에서 실행 |
| `--model` | 주 루프 모델 오버라이드 |
| `--resume` | 이전 세션 복원 |

### SDK 진입점 (programmatic API)

`-p` 플래그나 SDK 프로그래밍 API를 통해 호출할 때 React TUI를 우회하고 직접 `QueryEngine.ts`로 진입한다:

- `isNonInteractiveSession = true`
- 모든 UI 렌더링(Ink)을 건너뜀
- `SDKMessage` 타입의 스트리밍 출력을 stdout으로
- `SDKStatus`, `SDKPermissionDenial`, `SDKCompactBoundaryMessage` 등의 구조화된 출력 지원

SDK 모드에는 전용 beta features도 있다: `entrypoints/sdk/coreSchemas.ts`가 구조화된 JSON 입력/출력 schema를 정의하고, `entrypoints/agentSdkTypes.ts`가 `HookEvent`, `ModelUsage` 등 SDK 전용 타입을 정의한다.

### MCP 진입점 (MCP server mode)

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools: Claude Code의 모든 도구를 MCP tools로 노출
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool: 해당 Tool 구현으로 프록시 실행
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

MCP 모드는 Claude Code의 전체 도구 집합(BashTool, FileReadTool, GrepTool 등)을 외부 MCP 클라이언트에 역으로 노출하여 "Claude Code as MCP server"를 실현한다.

### 세 가지 진입점의 공유 로직

어느 진입점이든 공유하는 것:
- `bootstrap/state.ts` 전역 상태
- `entrypoints/init.ts` 초기화 (memoized로 한 번만 실행 보장)
- `Tool.ts` 도구 레지스트리
- `services/` 아래의 모든 서비스 (API 클라이언트, 권한 시스템 등)
- Hooks 생명주기 시스템

차이점은 React TUI를 렌더링하는지 여부와 출력 형식(인터랙티브 텍스트 vs. 구조화된 JSON)이다.

---

## 1.6 설계 결정 분석

### 왜 Bun을 선택했는가, Node.js가 아니라

코드에서 Bun 사용의 특징을 관찰할 수 있다:

1. **`bun:bundle`의 `feature()` 함수**: Bun 고유의 컴파일 시점 feature flag 메커니즘으로, Dead Code Elimination을 지원한다. `main.tsx`에서 많이 사용된다(COORDINATOR_MODE, KAIROS, CHICAGO_MCP, UDS_INBOX 등), 외부 빌드에서 이 실험 코드가 완전히 제거된다.

2. **Bun의 WebView API** (조건부 참조): `typeof Bun !== 'undefined' && 'WebView' in Bun`, 일부 기능이 Bun 고유 API에 의존한다는 것을 보여준다.

3. **Bun의 single-file executable**: 주석에 `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`가 언급되어, 릴리스 산출물이 Bun으로 컴파일된 단일 파일 실행 파일임을 나타낸다.

4. **성능**: Bun의 시작 속도와 모듈 로딩 속도가 Node.js보다 현저히 빠르며, CLI 도구의 TTFR에 매우 중요하다.

동시에 Node.js 18+ 호환성을 유지하는 것은(`setup.ts`에 Node 버전 확인이 있음) Bun 이외의 환경(CI, 엔터프라이즈 관리 머신)을 지원하기 위해서다.

### 왜 React/Ink로 터미널 UI를 만들었는가

`components/` 디렉토리의 81,546행 코드는 UI 복잡도가 매우 높다는 것을 보여준다. 원시 ANSI 제어 코드로 수작업으로 작성하면 유지보수 비용이 감당하기 어려울 것이다. React/Ink 선택의 이점:

1. **선언적 UI**: 스트리밍 출력, 도구 실행 상태, 권한 확인 팝업 등을 모두 React state로 구동할 수 있으며, 명령형 커서 제어가 필요 없음
2. **컴포넌트 격리**: `screens/REPL.tsx`는 전체 레이아웃에만 집중하고, 각 하위 기능(입력 상자, 메시지 목록, 도구 진행 상황)이 각자 캡슐화됨
3. **Hot reload 친화적**: 개발 시 표준 React DevTools 사고방식으로 디버그 가능
4. **테스트 가능성**: React 컴포넌트는 `@testing-library/react`로 단위 테스트를 작성할 수 있으며, 실제 터미널에 의존하지 않음

### 병렬 사전 가져오기의 성능 최적화 사고방식

Claude Code의 시작 최적화는 명확한 우선순위 모델을 갖고 있다: **TTFR(Time To First Render) 최우선, "모든 초기화 완료"가 아니라**.

구체적으로:
- Keychain 읽기 (~65ms)가 첫 번째 import 부작용에서 트리거되며, API key가 필요할 때까지 기다리지 않음
- MCP 서버 연결이 백그라운드에서 병렬로 진행되며, REPL 렌더링이 기다리지 않음 (사용자가 화면을 본 후에야 MCP 연결이 완료됨)
- Release notes, GrowthBook 설정, plugin hooks가 모두 fire-and-forget

대가는 "사전 가져오기 완료 전에 소비됨"의 race condition을 세심하게 관리해야 한다는 것으로, `ensureKeychainPrefetchCompleted()` 등의 await 포인트로 정밀 제어한다.

### 지연 로딩 vs. 사전 로딩의 tradeoff

| 전략 | 대상 | 이유 |
|------|------|------|
| 사전 로딩 (import 부작용) | MDM 자식 프로세스, Keychain | I/O 집약적, 빠를수록 좋음 |
| 지연 로딩 (`await import()`) | OpenTelemetry (~400KB), gRPC (~700KB), React TUI 컴포넌트 | 모듈 평가 비용이 높으며, 핵심 경로에 없음 |
| 조건부 로딩 (`feature()` DCE) | COORDINATOR_MODE, KAIROS, CHICAGO_MCP | 실험 기능, 외부 사용자에게 불필요 |
| `setImmediate()` 지연 | commit attribution hook | setup()의 마이크로태스크 윈도우에서 event loop 차단 방지 |

이 계층화된 전략으로 Claude Code가 시작 시 "화면 표시에 필요한 최소한의 작업"만 수행하게 한다.

---

## 1.7 이식 가능한 패턴

### 시작 최적화의 범용 패턴

Claude Code의 시작 시퀀스는 재사용 가능한 "**병렬 예열 + 지연 로딩 + DCE**" 3층 최적화 프레임워크를 보여준다:

**Pattern 1: ES module 부작용으로 I/O 예열**
```typescript
// import 문 사이에 fire-and-forget I/O 삽입
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // 즉시 트리거, await 없음
import { SomethingElse } from './other.js'  // 병렬 로딩
```
적용: "반드시 읽어야 하지만 느린" 초기화 데이터(설정 파일, 자격증명, 네트워크 사전 연결)가 있는 모든 경우.

**Pattern 2: memoize 단일 초기화**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
적용: 여러 진입점이 공유하는 초기화 로직으로, 중복 실행을 방지.

**Pattern 3: `--bare` 모드 계층화**
스크립트/API 호출은 UI, terminal 확인, analytics 등이 필요 없으므로, `isBareMode()`로 빠르게 건너뛰어 헤드리스 호출의 낮은 오버헤드를 유지한다.

**Pattern 4: 상태 분리**
`bootstrap/state.ts`를 엄격한 리프 모듈(순환 의존 없음)로 사용하고, setter/getter로 접근하며, ESLint 규칙으로 강제한다. 이로써 상태 모듈을 어디서든 안전하게 import할 수 있다.

### Doramagic CLI가 참고할 수 있는 점

위의 분석을 바탕으로 Doramagic CLI는 아키텍처 설계에서 다음 패턴을 채택할 수 있다:

1. **시작 핵심 경로 분리**: "렌더링 전에 완료해야 하는 것"과 "렌더링 후에 완료할 수 있는 것"을 엄격히 분리하고, 주석으로 이유를 표시(Claude Code의 `// ~65ms on every macOS startup` 주석 스타일 참조)

2. **전역 상태 싱글톤 + 접근자 패턴**: `bootstrap/state.ts`를 참조하여, 엄격한 리프 모듈로 세션 상태를 유지하고, 상태가 여기저기 흩어지는 것을 방지

3. **`memoize` 초기화 함수**: 어느 진입점에서 호출하든 초기화가 한 번만 실행되도록 보장

4. **세 가지 모드 분리**: interactive(React TUI) / headless(-p 플래그) / server(MCP), 하위 도구와 서비스 계층을 공유

5. **feature flag + DCE**: 실험적 기능을 feature flag로 감싸고, 릴리스 시 자동으로 제거

---

## 1.8 소스 인덱스

| 파일 | 행수 | 핵심 내용 |
|------|------|----------|
| `main.tsx` | 4,683 | 주 진입점, Commander 라우팅, 상태 초기화, 인터랙티브/headless 분기 |
| `setup.ts` | 477 | 세션 초기화: cwd, hooks, worktree, plugin 사전 가져오기 |
| `bootstrap/state.ts` | 1,758 | 전역 상태 싱글톤, `State` 타입 정의, 모든 getter/setter |
| `entrypoints/init.ts` | ~400 | memoized 전역 초기화: config, mTLS, proxy, OTel 지연 로딩 |
| `entrypoints/cli.tsx` | ~2,000 | Commander.js 라우팅, 인터랙티브/print/resume/teleport 분기 |
| `entrypoints/mcp.ts` | ~200 | MCP server 모드, 도구 집합 노출 |
| `entrypoints/sdk/coreSchemas.ts` | - | SDK 모드 구조화된 입력/출력 schema |
| `entrypoints/agentSdkTypes.ts` | - | SDK 전용 타입 (HookEvent, ModelUsage 등) |
| `replLauncher.tsx` | ~30 | 지연 로딩 App + REPL, React TUI 시작 |
| `QueryEngine.ts` | ~1,500 | 세션 생명주기 관리, headless 경로 핵심 |
| `Tool.ts` | - | 도구 인터페이스 정의 (inputSchema, call, prompt 등) |
| `tools/` | 50,828 | 30 개의 도구 구현 (BashTool/FileEditTool/AgentTool 등) |
| `services/api/` | - | Claude API 호출, 재시도, usage 통계 |
| `services/mcp/client.ts` | - | MCP 클라이언트 연결 관리 |
| `utils/startupProfiler.ts` | - | `profileCheckpoint()` 성능 측정 포인트 |
| `utils/secureStorage/keychainPrefetch.ts` | - | macOS Keychain 병렬 사전 읽기 |
| `utils/settings/mdm/rawRead.ts` | - | MDM 설정 병렬 읽기 |

### 핵심 코드 위치

- **병렬 예열 시작점**: `main.tsx:12-20` (3 개의 import 부작용)
- **memoized 초기화**: `entrypoints/init.ts:57` (`export const init = memoize(...)`)
- **전역 상태 타입**: `bootstrap/state.ts:30-200` (`type State = {...}`)
- **MCP server 정의**: `entrypoints/mcp.ts:42` (`startMCPServer`)
- **REPL 렌더링 진입점**: `replLauncher.tsx:14` (`launchRepl`)
- **도구 인터페이스**: `Tool.ts:1-30` (`ToolInputJSONSchema`, `ToolUseContext`)
- **setup 핵심 순서**: `setup.ts:77-230` (setCwd → captureHooksConfigSnapshot → worktree → background jobs)

---

*장 분량: 약 9,800 문자 | 소스 스냅샷 날짜: 2026-03-31*
