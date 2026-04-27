# 제 7 장: 명령 시스템

## 7.1 개요 및 위치

Claude Code의 명령 시스템은 사용자가 REPL과 상호작용하는 핵심 진입점입니다. 사용자가 입력창에 `/`를 입력할 때마다 이 시스템이 작동합니다. 세 가지 역할을 맡습니다:

1. **UI 제어 레이어**: LLM을 거치지 않고 터미널 인터페이스 상태를 직접 조작합니다（예: `/clear`, `/theme`, `/vim`）
2. **세션 관리 레이어**: 대화 이력, 컨텍스트 압축 및 복원을 관리합니다（예: `/compact`, `/resume`, `/branch`）
3. **능력 확장 레이어**: 복잡한 작업을 모델에 위임하고 Prompt 전개 메커니즘을 통해 실행합니다（예: `/review`, `/skills`）

명령 시스템의 경계 설계는 명확한 관심사 분리를 구현합니다: 명령은 "트리거"를 담당하고, 도구（Tools）는 "실행"을 담당하며, LLM은 "결정"을 담당합니다. `/review` 명령은 git를 직접 호출하지 않고, 리뷰 prompt를 대화 흐름에 주입하여 모델이 후속 도구 호출 체인을 구동하도록 합니다.

---

## 7.2 이론적 기반

### 명령 패턴（Command Pattern）

시스템 설계는 고전적인 GoF 명령 패턴과 매우 잘 일치합니다:

- **Command 인터페이스**: `Command` 유니온 타입（`PromptCommand | LocalCommand | LocalJSXCommand`）, 요청을 통합 캡슐화
- **ConcreteCommand**: 각 `commands/<name>/index.ts` 파일이 구체적인 명령 구현
- **Invoker**: REPL의 `processSlashCommand`가 실행을 스케줄링
- **Receiver**: `ToolUseContext`（대화 상태）, `AppState`（애플리케이션 상태）가 조작되는 객체

그러나 Claude Code는 고전적인 패턴에 두 가지 핵심 확장을 추가합니다:

**지연 로딩**: 명령은 등록 시 즉시 인스턴스화되지 않고 `load(): Promise<Module>`을 통해 지연 로딩됩니다. 이는 시작 오버헤드를 첫 번째 호출 시점으로 분산시키며, 무거운 의존성이 있는 명령（예: `/insights`의 113KB HTML 렌더링 모듈）에 특히 중요합니다.

**타입화된 반환값**: 명령은 반환값이 없는 동작（void）이 아니라 구조화된 결과（`LocalCommandResult`）를 반환하며, 상위 REPL이 렌더링 방법을 결정하여 실행과 표시의 디커플링을 구현합니다.

### REPL 명령 처리의 설계 패턴

Claude Code가 채택한 REPL 명령 처리는 두 가지 핵심 원칙을 따릅니다:

**Immediate vs Queued**: 명령 객체의 `immediate?: boolean` 필드가 명령이 메시지 큐를 우회하여 즉시 실행할지 결정합니다. `/clear`, `/exit` 등의 인터페이스 작업은 즉각적인 응답이 필요하고, `/compact` 같이 API 호출이 포함된 작업은 큐에 들어가 순서대로 처리됩니다.

**Auth-gated 가용성**: 런타임 feature flag（`isEnabled()`）와 달리, `availability` 필드는 명령 목록 필터링 단계에서 효력이 발생하여 인가되지 않은 사용자는 특정 명령의 존재조차 볼 수 없도록 합니다（예: claude.ai 구독자만 사용 가능한 명령）.

---

## 7.3 명령 등록 메커니즘

### commands.ts의 등록 흐름

명령 등록의 핵심 로직은 `commands.ts`（754줄）에 집중되어 있으며, 전체적으로 네 가지 레이어로 나뉩니다:

**첫 번째 레이어: 정적 내장 명령**

```typescript
// commands.ts:240-310（핵심 코드）
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... 약 60개의 내장 명령
])
```

`COMMANDS` 함수가 모듈 수준 상수가 아닌 `memoize`로 래핑된 이유는 일부 명령이 등록 시 구성 파일을 읽어야 하는데, 모듈 초기화 시에는 구성이 아직 사용 불가능하기 때문입니다.

**두 번째 레이어: Feature-flag 조건 명령**

```typescript
// commands.ts:68-112（조건 import 코드）
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

이 명령들은 `bun:bundle`의 `feature()` 함수를 통해 dead code elimination을 수행하여, 빌드 시 활성화되지 않은 명령을 직접 제거하며 런타임 판단이 아닙니다.

**세 번째 레이어: 내부 전용 명령**

```typescript
// commands.ts:197-222（INTERNAL_ONLY_COMMANDS）
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, ultraplan, subscribePr, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
].filter(Boolean)
```

이 명령들은 `USER_TYPE === 'ant'`（Anthropic 내부 사용자）이고 데모 모드가 아닌 경우에만 등록됩니다. 내부 도구 및 디버그 명령의 격리 메커니즘입니다.

**네 번째 레이어: 동적 로딩 명령**

```typescript
// commands.ts:360-395（loadAllCommands）
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

Skills, 플러그인 명령, 워크플로우 명령 세 가지가 병렬로 비동기 로딩되고 우선순위에 따라 정렬됩니다: bundled skills가 최고 우선순위, 내장 명령이 최저. 이는 사용자 정의 명령이 동일한 이름의 내장 명령을 덮어쓸 수 있도록 보장합니다.

### 명령의 타입 정의

`types/command.ts`는 세 가지 상호 배타적인 명령 타입을 정의하며, `Command` 유니온 타입을 구성합니다:

```typescript
// types/command.ts（핵심 유니온 타입）
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| 타입 | 설명 | 대표 명령 |
|------|------|----------|
| `PromptCommand` | prompt로 전개되어 대화 흐름에 주입, 모델이 실행 | `/review`, `/skills`, 모든 Skill |
| `LocalCommand` | 순수 로컬 동기 실행, 텍스트 결과 반환 | `/compact`, `/context` |
| `LocalJSXCommand` | Ink React UI 컴포넌트 렌더링 | `/model`, `/resume`, `/config` |

`CommandBase`는 세 가지가 공유하는 기본 필드 집합으로 다음을 포함합니다:

```typescript
// types/command.ts（CommandBase 핵심 필드）
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // 런타임 feature flag 검사
  isHidden?: boolean                      // typeahead에서 숨기기
  argumentHint?: string                   // 파라미터 힌트 텍스트
  whenToUse?: string                      // 모델 호출 시나리오 설명
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // 큐를 우회하여 즉시 실행
  isSensitive?: boolean                   // 이력에서 파라미터 마스킹
}
```

### 명령 분류（내장 vs 플러그인 vs 사용자 정의）

```
명령 소스 계층（우선순위 높음에서 낮음으로）
├── bundledSkills        # Claude Code와 함께 패키징된 내장 Skills
├── builtinPluginSkills  # 활성화된 내장 플러그인이 제공하는 Skills
├── skillDirCommands     # 사용자 .claude/skills/ 디렉터리의 Skills
├── workflowCommands     # 워크플로우 스크립트 명령（feature: WORKFLOW_SCRIPTS）
├── pluginCommands       # 서드파티 플러그인이 등록한 명령
├── pluginSkills         # 서드파티 플러그인이 제공하는 Skills
└── COMMANDS()           # 하드코딩된 내장 명령（최저 우선순위）
```

---

## 7.4 명령 분류 완전 목록

다음은 `commands.ts`의 `ls` 출력 및 등록 목록을 기반으로 정리된 것입니다.

### 세션 관리 유형

| 명령 | 설명 |
|------|------|
| `/compact [instructions]` | 대화 이력 압축, 컨텍스트 윈도우 해제 |
| `/resume` | 이력 session 목록에서 선택하여 대화 복원 |
| `/branch [title]` | 현재 대화에서 새 session으로 분기 |
| `/rewind` | 대화의 특정 이력 노드로 되돌리기 |
| `/clear` | 현재 대화 기록 지우기 |
| `/session` | 현재 session 정보 표시 |
| `/rename` | 현재 session 이름 바꾸기 |
| `/summary` | 현재 대화 요약 생성（내부 명령） |
| `/export` | 대화 내용 내보내기 |
| `/copy` | 마지막 메시지를 클립보드에 복사 |

### 개발 도구 유형

| 명령 | 설명 |
|------|------|
| `/review [PR#]` | 로컬 코드 리뷰（`gh pr diff` 호출） |
| `/ultrareview [PR#]` | 클라우드 심층 코드 리뷰（10-20분, bughunter 구동） |
| `/commit` | 코드 변경 커밋（내부 명령） |
| `/commit-push-pr` | 커밋 + Push + PR 생성（내부 명령） |
| `/diff` | 현재 git diff 보기 |
| `/init` | 프로젝트 초기화（CLAUDE.md 생성） |
| `/add-dir` | 추가 작업 디렉터리 추가 |
| `/hooks` | 이벤트 후크 구성 관리 |
| `/files` | 세션에서 추적된 파일 목록 표시 |
| `/pr_comments` | PR 댓글 보기 |
| `/issue` | GitHub Issue 생성/보기（내부 명령） |
| `/autofix-pr` | PR의 문제 자동 수정（내부 명령） |

### 구성 유형

| 명령 | 설명 |
|------|------|
| `/model [name]` | 대화 모델 전환（대화형 선택기 포함） |
| `/config` | 구성 항목 보기/수정 |
| `/theme` | 터미널 테마 전환 |
| `/vim` | vim 입력 모드 전환 |
| `/keybindings` | 단축키 바인딩 관리 |
| `/permissions` | 도구 권한 보기/수정 |
| `/privacy-settings` | 개인 정보 설정 관리 |
| `/output-style` | 출력 형식 선호도 설정 |
| `/effort` | 응답 노력 수준 설정 |
| `/fast` | 빠른 모드 전환 |
| `/plan` | Plan 모드 전환（계획만, 실행 없음） |
| `/sandbox-toggle` | 샌드박스 모드 전환 |

### 디버깅 및 진단 유형

| 명령 | 설명 |
|------|------|
| `/doctor` | 구성 및 환경 문제 진단 |
| `/cost` | 현재 session의 토큰 소비 및 비용 표시 |
| `/context` | 컨텍스트 윈도우 사용 세부 정보 표시（카테고리별 테이블） |
| `/stats` | 사용 통계 데이터 표시 |
| `/usage` | API 사용량 정보 표시 |
| `/insights` | 이력 session 사용 분석 보고서 생성（113KB 모듈 지연 로딩） |
| `/heapdump` | 메모리 힙 스냅샷 생성（디버깅용） |
| `/debug-tool-call` | 도구 호출 디버깅（내부 명령） |
| `/perf-issue` | 성능 문제 기록（내부 명령） |
| `/ant-trace` | Anthropic 내부 추적（내부 명령） |

### 신원 및 서비스 유형

| 명령 | 설명 |
|------|------|
| `/login` | Claude.ai 계정 로그인 |
| `/logout` | 로그아웃 |
| `/upgrade` | 더 높은 요금제로 업그레이드 |
| `/install-github-app` | GitHub App 설치 |
| `/install-slack-app` | Slack App 설치 |
| `/ide` | IDE 통합 관리 |
| `/terminalSetup` | 터미널 통합 구성 |
| `/mobile` | 모바일 연결 QR 코드 표시 |
| `/chrome` | Chrome 확장 관리 |
| `/desktop` | 데스크톱 앱 관리 |

### 고급 기능 유형

| 명령 | 설명 |
|------|------|
| `/mcp` | MCP 서버 관리（목록/시작/재시작） |
| `/skills` | Skills 관리（목록/설치/업데이트） |
| `/tasks` | 백그라운드 작업 관리 |
| `/agents` | 자식 에이전트 관리 |
| `/memory` | 프로젝트 메모리 파일 관리（CLAUDE.md） |
| `/plan` | 계획 모드 진입 |
| `/thinkback` | 모델 사고 과정 역추적 |
| `/thinkback-play` | 사고 역추적 애니메이션 재생 |
| `/advisor` | AI 고문 모드 |
| `/plugin` | 플러그인 관리 |
| `/reload-plugins` | 플러그인 다시 로딩 |
| `/passes` | 멀티 라운드 리뷰 passes 관리 |
| `/feedback` | Anthropic에 피드백 보내기 |
| `/btw` | 주석 메시지 추가 |
| `/tag` | 대화에 태그 달기 |
| `/stickers` | 스티커 표시（이스터에그 기능） |

Feature-flag 조건 명령（기본적으로 숨겨짐）:

| 명령 | Feature Flag | 설명 |
|------|-------------|------|
| `/ultraplan` | `ULTRAPLAN` | 클라우드 슈퍼 계획（장기 비동기） |
| `/voice` | `VOICE_MODE` | 음성 입력 모드 |
| `/bridge` | `BRIDGE_MODE` | 원격 제어 브릿지 모드 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 스크립트 워크플로우 명령 |
| `/peers` | `UDS_INBOX` | 피어 session 통신 |
| `/fork` | `FORK_SUBAGENT` | 명시적으로 자식 에이전트 생성 |
| `/buddy` | `BUDDY` | Buddy 협업 모드 |

---

## 7.5 명령 실행 흐름

### 사용자 입력 "/"에서 명령 실행까지의 완전한 경로

```
사용자가 "/compact some instructions"를 입력
        │
        ▼
    REPL 입력 처리기
    "/" 접두사 감지
        │
        ▼
    getCommands(cwd)                    ← 모든 소스에서 명령 목록 집계
    findCommand("compact", commands)     ← name / aliases로 검색
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← auth 타입 게이트 확인
    isCommandEnabled(cmd)               ← feature flag / isEnabled() 확인
        │
        ├── cmd.immediate 확인          ← true: 큐를 우회하여 즉시 실행
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        LocalCommandResult 반환
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        React 컴포넌트를 터미널에 렌더링
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   ContentBlockParam[] 반환
                                   대화 흐름에 주입 → 모델 추론 트리거
```

### 명령 파라미터 파싱

명령 시스템에는 통합된 파라미터 파싱 프레임워크가 내장되어 있지 않습니다—이것은 의도적인 설계 선택입니다. 각 명령은 자체 `args: string` 파라미터를 처리하여 극도의 유연성을 유지합니다:

- `/compact`는 `args.trim()`을 직접 사용자 정의 압축 지시로 사용
- `/review`는 `/^\d+$/.test(prNumber)`로 PR 번호인지 판단
- `/model`은 args가 있을 때 `SetModelAndClose`로 직접 설정하고, args가 없을 때 대화형 `ModelPickerWrapper` 렌더링
- `/resume`은 session ID（UUID）, 사용자 정의 제목 또는 파라미터 없을 때 목록 선택기를 지원

이 설계는 통합 파싱 레이어의 복잡성을 피하며, 대가로 각 명령이 자체적으로 엣지 케이스를 처리해야 합니다.

### 명령 출력 렌더링

`LocalCommandResult`의 세 가지 타입은 서로 다른 렌더링 경로에 해당합니다:

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // 텍스트 메시지로 렌더링
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // 컨텍스트 교체 로직 트리거
  | { type: 'skip' }                      // 아무것도 렌더링하지 않음
```

`LocalJSXCommand`는 `onDone()` 콜백을 통해 결과를 REPL에 전달합니다:

```typescript
// types/command.ts（LocalJSXCommandOnDone）
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // 메시지 표시 방식
    shouldQuery?: boolean                   // 즉시 모델 쿼리 트리거 여부
    metaMessages?: string[]                 // 모델에게는 보이지만 사용자에게는 안 보이는 메시지 삽입
    nextInput?: string                      // 다음 입력에 자동 채우기
    submitNextInput?: boolean               // 자동 제출 여부
  },
) => void
```

`display: 'system'`은 시스템 메시지 스타일로 표시하고（회색 이탤릭）, `display: 'user'`는 일반 사용자 메시지로, `display: 'skip'`은 완전히 표시하지 않습니다.

---

## 7.6 대표 명령 심층 분석

### /compact 명령의 구현 세부사항

`/compact`는 명령 시스템에서 로직이 가장 복잡한 명령 중 하나로, 대화 이력 압축의 핵심 책임을 집니다.

**실행 결정 트리**（`commands/compact/compact.ts`）:

```
/compact [instructions]
    │
    ├── 사용자 정의 지시 있음?
    │   └── 없음 → trySessionMemoryCompaction()   ← 먼저 Session Memory 압축 시도
    │                  성공하면 즉시 반환, 가장 빠른 경로
    │
    ├── isReactiveOnlyMode() ?
    │   └── 예 → compactViaReactive()               ← 반응형 압축 경로（새 아키텍처）
    │               병렬 실행: executePreCompactHooks + getCacheSharingParams
    │               호출: reactiveCompactOnPromptTooLong()
    │
    └── 아니오 → 전통 압축 경로
              microcompactMessages()                  ← 먼저 미세 압축으로 토큰 감소
              compactConversation()                   ← 주 압축（요약 생성）
              setLastSummarizedMessageId(undefined)   ← 추적 포인터 재설정
```

핵심 설계 포인트: 압축 전 반드시 `getMessagesAfterCompactBoundary(messages)`를 호출하여 REPL이 UI scrollback을 위해 보존한 이미 잘린 메시지를 필터링해야 합니다—이 메시지들은 요약에 나타나서는 안 됩니다.

압축 성공 후의 정리 시퀀스는 고정됩니다:
1. `setLastSummarizedMessageId(undefined)` — 메시지 포인터 재설정
2. `suppressCompactWarning()` — "컨텍스트가 곧 소진됩니다" 경고 억제
3. `getUserContext.cache.clear?.()` — 사용자 컨텍스트 캐시 지우기
4. `runPostCompactCleanup()` — 압축 후 후크 트리거

**Reactive Compact 경로**는 병렬 최적화를 활용합니다:

```typescript
// compact.ts:compactViaReactive（핵심 병렬 섹션）
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // 압축 전 후크 실행（자식 프로세스 시작 가능）
  getCacheSharingParams(context, messages),  // 시스템 prompt 빌드（모든 도구 순회）
])
```

두 가지가 서로 독립적이어서 병렬 실행은 대기 시간을 크게 줄입니다.

### /model 명령의 모델 전환 로직

`/model`은 `local-jsx` 타입으로, React 컴포넌트로 대화형 선택기를 렌더링합니다.

**두 가지 실행 경로**:

- **파라미터 있음**（`/model claude-sonnet-4-6`）: `SetModelAndClose` 컴포넌트 렌더링, `useEffect`에서 모델 검증을 비동기 실행하고 `onDone()`으로 즉시 완료
- **파라미터 없음**（`/model`）: `ModelPickerWrapper` 컴포넌트 렌더링, 완전한 `ModelPicker` 대화형 인터페이스 표시

**모델 전환의 상태 업데이트**:

```typescript
// model.tsx:handleSelect（핵심 상태 업데이트）
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // session 수준 임시 오버라이드 지우기
}))
```

**모델 검증 계층**（빠른 것에서 느린 것 순서）:
1. `isModelAllowed(model)` 확인 — 조직 제한 화이트리스트
2. `isOpus1mUnavailable(model)` 확인 — 1M 컨텍스트 권한 검사
3. `isKnownAlias(model)` 확인 — 알려진 별칭은 바로 통과（API 검증 건너뜀）
4. `validateModel(model)` — API를 호출하여 사용자 정의 모델명 검증

Fast Mode와 모델 전환은 연동됩니다: 새 모델이 Fast Mode를 지원하지 않으면 자동으로 끄고; 지원하고 이미 활성화되어 있으면 확인 메시지에 "Fast mode ON"을 표시합니다.

### /review 명령의 코드 리뷰 흐름

`/review`는 `PromptCommand` 타입의 전형적인 용도를 보여줍니다—간결한 prompt 템플릿으로 완전한 리뷰 흐름을 구동합니다:

```typescript
// review.ts:LOCAL_REVIEW_PROMPT（완전한 prompt 템플릿）
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

명령 자체에는 4줄의 핵심 코드만 있고 나머지는 모두 모델이 처리합니다—이것이 `PromptCommand`의 설계 철학입니다: **명령이 WHAT을 정의하고, 모델이 HOW를 결정합니다**.

이와 대조적으로 `/ultrareview`（`local-jsx` 타입）는 완전히 다른 경로를 실행합니다:

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← 무료 한도 / Extra Usage 잔액 확인
    │   ├── Team/Enterprise → 바로 통과
    │   ├── 무료 횟수 있음 → 통과, 힌트 첨부
    │   └── 한도 소진 → 초과 확인 대화상자 표시
    │
    └── launchRemoteReview()
        ├── PR 모드 → teleportToRemote(branchName: "refs/pull/N/head")
        └── 브랜치 모드 → git merge-base → git diff 확인 → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → 작업 URL 반환, 모델이 사용자에게 알림
```

`/ultrareview`는 코드 리뷰 작업을 클라우드로 "전송"하고, 로컬에서 `RemoteAgentTask`를 등록한 후 즉시 반환하며 폴링 메커니즘으로 결과를 수신합니다—이것은 비동기 작업 위임 패턴으로, 로컬 명령의 동기 실행 모델과 완전히 다릅니다.

---

## 7.7 명령과 Skill의 경계

### 두 가지의 차이점과 공통점

| 차원 | 명령（Command） | Skill |
|------|---------------|-------|
| 정의 방법 | TypeScript 코드, 하드코딩된 로직 | Markdown 파일, frontmatter + prompt 내용 |
| 로딩 시기 | 시작 시 정적 등록（내장）또는 비동기 로딩（플러그인） | 런타임에 파일 시스템 스캔 |
| 실행 타입 | `local` / `local-jsx` / `prompt` | `prompt`만（prompt로 전개） |
| 모델 호출 가능 | 대부분의 내장 명령은 모델 호출 금지（`source: 'builtin'`） | 설계상 모델이 SkillTool을 통해 호출 지원 |
| 사용자 가시성 | 모든 명령이 `/` typeahead에 표시 | `userInvocable`과 `hasUserSpecifiedDescription`에 따라 결정 |
| 컨텍스트 인식 | `ToolUseContext`를 통해 완전한 애플리케이션 상태 접근 | prompt 내용만 사용 가능, 직접적인 상태 접근 없음 |
| 소스 식별 | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### 설계 선택 뒤의 고려사항

**왜 내장 명령은 Markdown Skill을 사용하지 않는가?**

내장 명령은 애플리케이션 상태（`AppState`）에 접근하고, Node.js API（파일 시스템, 암호화）를 호출하고, React 컴포넌트를 렌더링해야 합니다—이 능력들은 prompt 템플릿이 표현할 수 있는 범위를 훨씬 넘어섭니다. `/compact`는 4가지 다른 압축 전략을 호출해야 하고, `/model`은 대화형 UI를 렌더링해야 하며, `/resume`은 session 파일을 읽고 써야 합니다. 이 모든 것은 코드여야 합니다.

**SkillTool의 필터링 로직**은 경계의 정확한 획정을 보여줍니다:

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← 내장 명령은 제외
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`**이 핵심 규칙입니다: 내장 명령은 모델이 호출 가능한 목록에서 명시적으로 제외됩니다. 이는 모델이 SkillTool을 통해 권한 검사를 우회하여 세션 상태를 직접 조작하는 것을 방지합니다.

**원격 안전 명령 집합（REMOTE_SAFE_COMMANDS）**은 이 경계를 더욱 세분화합니다:

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

20개의 명령만 `--remote` 모드에서 사용 가능합니다—이 명령들은 로컬 파일 시스템, git, IDE에 의존하지 않고 순수 TUI 상태 작업으로, 원격 브릿지 세션에서 안전하게 실행될 수 있습니다.

---

## 7.8 설계 결정 분석

**결정 1: 통합 인터페이스 대신 세 가지 명령 타입**

`local` / `local-jsx` / `prompt`의 세 가지 분류가 복잡도를 높이는 것처럼 보이지만, 각 타입이 서로 다른 핵심 문제를 해결합니다:
- `local`은 부수 효과가 있지만 UI가 없는 작업을 처리（구조화된 데이터 반환 필요）
- `local-jsx`는 대화형 인터페이스가 필요한 작업을 처리（Ink 렌더링 트리에 의존）
- `prompt`는 모델에 위임할 수 있는 작업을 처리（가장 낮은 결합도）

강제로 단일 인터페이스로 통합하면, 모든 명령이 React 렌더링을 처리해야 하거나（불필요한 의존성）, 타입 안전성을 잃게 됩니다.

**결정 2: 전역 싱글톤 대신 cwd로 memoize**

`loadAllCommands = memoize(async (cwd: string) => ...)`는 작업 디렉터리를 캐시 키로 사용하여 다른 디렉터리의 Claude Code 인스턴스가 독립적인 명령 캐시를 가집니다. 이는 monorepo 및 멀티 프로젝트 시나리오에서 각 디렉터리가 독립적인 Skills 집합을 가질 수 있도록 지원합니다.

**결정 3: 통합 파라미터 파싱 없음**

이것은 의도적인 "느슨한 설계"입니다. 통합 파싱 프레임워크（commander.js 등）는 각 명령이 완전한 파라미터 schema를 선언하도록 강제하는데, `/compact` 같이 "자유 텍스트 지시" 명령에는 전혀 의미가 없습니다. 원시 문자열을 보존하여 각 명령이 자체적으로 파싱 방법을 결정하게 하면, 유연성을 얻는 대신 일관성을 희생합니다.

**결정 4: Availability vs isEnabled의 두 레이어 게이팅**

두 레이어 게이팅은 다른 생명주기의 가시성 문제를 해결합니다:
- `availability`는 명령 목록 빌드 시 필터링하고 결과를 캐시하여 정적인 auth 타입 검사에 적합
- `isEnabled()`는 매번 `getCommands()` 호출 시 재평가되어（캐시 없음）동적 feature flag 검사에 적합

주석에서 `isEnabled()`가 memoize되지 않는 이유를 특별히 설명합니다: `/login` 실행 후 auth 상태가 변경되므로 명령 목록에 즉시 반영되어야 합니다.

**결정 5: 내부 명령은 별도 패키지 관리 없음**

`INTERNAL_ONLY_COMMANDS`는 환경 변수 `USER_TYPE === 'ant'`로 직접 가시성을 제어하며, 별도 npm 패키지를 사용하지 않습니다. 이는 빌드 복잡도를 단순화하지만, 외부 빌드 시 dead code elimination으로 이 코드를 제거해야 합니다（`filter(Boolean)`은 `null` 조건 명령에도 효과적）.

---

## 7.9 이전 가능한 패턴

### 패턴 1: 명령 타입 세 가지 분류

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**적용 시나리오**: "순수 로직", "UI 상호작용", "LLM 위임"을 동시에 지원해야 하는 모든 명령 시스템. 세 가지의 경계가 매우 명확하여 다른 REPL/CLI 프레임워크로 직접 이식할 수 있습니다.

**핵심 가치**: 타입 시스템이 관심사 분리를 강제하여 런타임 isinstance 검사가 불필요합니다.

### 패턴 2: 지연 로딩 + cwd로 memoize

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

**적용 시나리오**: 명령 수가 많거나（>50）, 일부 명령이 무거운 모듈에 의존하는 CLI 도구.

**구현 핵심**: memoize 키는 명령 집합에 영향을 미치는 모든 요소를 포함해야 하며（여기서는 cwd）, 캐시 무효화 시기는 실제 상태 변화에 해당해야 합니다（여기서는 `clearCommandsCache()`）.

### 패턴 3: 다중 소스 명령 집계 + 우선순위 정렬

```typescript
return [
  ...bundledSkills,       // 최고 우선순위（동일 이름의 내장 명령 덮어쓸 수 있음）
  ...pluginCommands,
  ...COMMANDS(),          // 최저 우선순위（덮어쓸 수 있음）
]
```

**적용 시나리오**: 플러그인 생태계를 지원하는 CLI 도구로, 서드파티 확장이 내장 동작을 덮어쓸 수 있어야 합니다.

**주의사항**: `findCommand`가 목록의 첫 번째 일치 항목을 반환하므로, 배열 순서가 우선순위 순서입니다. 설계 시 명확하게 문서화해야 합니다.

### 패턴 4: Auth-gated 명령 가시성

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai': if (isClaudeAISubscriber()) return true; break
      case 'console':   if (!isUsing3PServices() && ...) return true; break
    }
  }
  return false
}
```

**적용 시나리오**: 다른 구독 수준의 사용자에게 다른 기능 집합을 표시해야 하는 SaaS 제품.

**핵심 설계**: 실행 단계에서 오류를 보고하는 대신 명령 목록 필터링 단계에서 차단합니다—사용자는 사용할 수 없는 명령을 볼 수 없어 인지 부담이 줄어듭니다.

### 패턴 5: Bridge Safe / Remote Safe 화이트리스트

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([session, exit, clear, ...])
```

**적용 시나리오**: 제한된 환경（원격 세션, 샌드박스, 모바일 브릿지）에서 실행해야 하는 명령 시스템.

**구현 아이디어**: 블랙리스트보다 안전합니다—새로 추가된 명령은 기본적으로 제한된 환경에서 사용 불가능하며, 명시적으로 화이트리스트에 추가해야 합니다. 이는 실수로 민감한 명령이 잘못된 환경에 노출되는 것을 방지합니다.

---

## 7.10 소스 코드 색인

| 파일 경로 | 줄수 | 내용 |
|---------|------|------|
| `src/commands.ts` | 754 | 명령 등록, 집계, 필터링, 검색의 모든 진입점 로직 |
| `src/types/command.ts` | ~250 | Command 유니온 타입 정의, CommandBase, 각 서브타입 상세 선언 |
| `src/commands/compact/compact.ts` | 287 | /compact 세 가지 경로 구현（session memory / reactive / traditional） |
| `src/commands/model/model.tsx` | 296 | /model 대화형 선택기 + 직접 설정 두 가지 경로, React Compiler 컴파일 출력 |
| `src/commands/review.ts` | ~50 | /review（prompt 타입）와 /ultrareview（local-jsx 타입）진입점 |
| `src/commands/review/reviewRemote.ts` | 316 | /ultrareview 원격 시작 로직: teleport, overage gate, 작업 등록 |
| `src/commands/resume/resume.tsx` | 274 | /resume session 목록 선택기 UI |
| `src/commands/branch/branch.ts` | 296 | /branch 대화 분기: JSONL 복사, sessionId 재작성, 충돌 처리 |
| `src/commands/context/context-noninteractive.ts` | 325 | /context 비대화형 경로: 분류별 토큰 통계, Markdown 테이블 렌더링 |
| `src/skills/loadSkillsDir.ts` | — | Skills 디렉터리 스캔 및 동적 로딩 로직 |
| `src/skills/bundledSkills.ts` | — | 제품과 함께 패키징된 내장 Skills 등록 |
| `src/plugins/builtinPlugins.ts` | — | 내장 플러그인의 Skill 명령 추출 |
| `src/utils/plugins/loadPluginCommands.ts` | — | 서드파티 플러그인 명령 로딩 및 캐시 |

**핵심 함수 색인**:

| 함수 | 파일 | 용도 |
|------|------|------|
| `getCommands(cwd)` | commands.ts | 현재 사용자가 사용 가능한 모든 명령 반환（주 진입점） |
| `findCommand(name, commands)` | commands.ts | 이름/별칭으로 명령 검색 |
| `meetsAvailabilityRequirement(cmd)` | commands.ts | auth 타입 게이트 검사 |
| `getSkillToolCommands(cwd)` | commands.ts | 모델이 호출 가능한 Skill 명령 집합 반환 |
| `getSlashCommandToolSkills(cwd)` | commands.ts | 사용자가 /를 통해 트리거할 수 있는 Skill 집합 반환 |
| `isBridgeSafeCommand(cmd)` | commands.ts | 명령이 bridge 모드에서 실행될 수 있는지 판단 |
| `formatDescriptionWithSource(cmd)` | commands.ts | 사용자 인터페이스에서 소스 주석이 추가된 설명 형식화 |
| `clearCommandsCache()` | commands.ts | 모든 명령 캐시 지우기（Skills 및 플러그인 포함） |
