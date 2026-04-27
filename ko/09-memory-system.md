# 제 9 장：기억 시스템

## 9.1 개요 및 위치

Claude Code의 기억 시스템은 전체 도구 체인에서 가장 정밀하게 설계되고 공학적 투자가 가장 깊은 하위 시스템 중 하나다. 이것은 LLM의 가장 근본적인 한계를 해결한다: 세션이 종료되면 컨텍스트 창이 초기화된다는 것. 사용자가 새 세션을 열 때마다 Claude는 백지 상태로 직면한다——사용자가 누구인지, 무엇을 선호하는지, 지난번에 어떤 실수를 했는지, 팀에 어떤 규범이 있는지를 모른다.

기억 시스템의 설계 목표는: **Claude가 세션을 넘어 연속성을 유지하여, 진정한 장기 협력자처럼 행동하게 하는 것이다.**

소스 코드 규모로 보면, 이것은 상당한 규모의 시스템이다:
- `memdir/` 디렉터리: 7개 파일, 1736줄
- `services/SessionMemory/`: 3개 파일, 1026줄
- `services/extractMemories/`: 2개 파일, 769줄
- `services/teamMemorySync/`: 5개 파일, 2167줄

합계 약 5700줄로, 전체 코드베이스의 약 1.1%를 차지하지만 그 복잡도와 설계 사고의 밀도는 이 비율을 훨씬 초과한다.

---

## 9.2 이론적 기반

### 인간 기억 모델과의 대응

시스템 아키텍처는 인지 과학의 세 가지 기억에 명확히 대응한다:

| 인간 기억 | Claude Code 대응 | 기술 구현 |
|---------|-----------------|---------|
| 작업 기억(Working Memory) | 현재 컨텍스트 창 | 세션 메시지 목록, 세션 종료 시 초기화 |
| 일화 기억(Episodic Memory) | Session Memory | `~/.claude/projects/<slug>/session-memory.md`, 세션 내 지속 업데이트 |
| 의미 기억(Semantic Memory) | Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`, 세션을 넘어 장기 보존 |

Session Memory는 "현재의 기억"에 해당한다——이번 세션에서 무엇을 하고 있는지, 어디까지 진행했는지를 기록한다; Persistent Memory는 "축적된 지식"에 해당한다——사용자 선호, 피드백 교훈, 프로젝트 배경.

### 지식 그래프 vs 문서 기억의 선택

시스템은 데이터베이스나 벡터 인덱스 대신 **파일 시스템 위의 Markdown 문서**를 선택했다. 이 선택은 `memoryTypes.ts`의 주석에 명확히 설명되어 있다:

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

이것은 제1원칙을 드러낸다: **실시간으로 쿼리할 수 있는 정보는 기억할 필요가 없다.** 기억은 "파생 불가능한" 컨텍스트만 저장한다——사용자 선호, 팀의 역사적 교훈, 프로젝트 이면의 동기. 이것은 지식 그래프의 설계와는 완전히 다르다. 지식 그래프는 구조화할 수 있는 모든 정보를 넣는 경향이 있다.

### 기억에서의 최종 일관성 적용

Team Memory의 동기화 설계는 명시적으로 최종 일관성 의미론을 채택한다:

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

삭제가 전파되지 않는 이 설계는 의도적이다——팀 기억은 "추가형" 자산이며, 실수로 삭제해도 영구적 손실이 되어서는 안 된다. 이것은 분산 시스템의 최종 일관성 원칙에 대한 보수적인 구현이다.

---

## 9.3 3층 기억 아키텍처

시스템은 3개의 층으로 구성되며, 생명주기가 가장 짧은 것에서 가장 긴 것 순으로:

### 제1층: Session Memory(세션 수준)

**파일 경로**: `~/.claude/projects/<sanitized-cwd>/session-memory.md`(`getSessionMemoryPath()`로 가져옴)

Session Memory는 **현재 세션 내에서 지속적으로 유지**되는 Markdown 파일로, 내용 구조가 고정되어 있다:

```markdown
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

(`services/SessionMemory/prompts.ts:14-36`, `DEFAULT_SESSION_MEMORY_TEMPLATE`)

세션이 종료되어도 삭제되지 않으며, Auto Compact 메커니즘이 컨텍스트를 압축할 때 읽어 새 컨텍스트 창에 "이전 이야기" 형식으로 주입된다.

**데이터 구조 제약**:
- 각 섹션 상한 2000 토큰(`MAX_SECTION_LENGTH = 2000`)
- 전체 상한 12000 토큰(`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`)
- 한도 초과 시 시스템이 prompt에 경고를 추가하고 Agent에게 압축을 요청함

**생명주기**: 현재 프로젝트 세션과 연결되며, Auto Compact 트리거 시 읽힘

### 제2층: Persistent Memory(세션을 넘는 영속 기억)

**파일 경로**: `~/.claude/projects/<sanitized-git-root>/memory/`

이것이 핵심 장기 기억 층이다. 각 기억은 YAML frontmatter가 있는 독립적인 `.md` 파일로 저장된다:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

(`memdir/memoryTypes.ts:230-240`, `MEMORY_FRONTMATTER_EXAMPLE`)

경로 해석 로직은 `getAutoMemPath()`가 담당한다(`memdir/paths.ts:173-190`). 해석 우선순위:

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 환경 변수(Cowork 다중 사용자 시나리오)
2. `settings.json`의 `autoMemoryDirectory`(policy/local/user 소스만 신뢰, **projectSettings 불신** — 악성 저장소의 쓰기 경로 탈취 방지)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`(기본값)

Git 작업 트리는 canonical git root(`findCanonicalGitRoot`)로 통일되어, 같은 저장소의 다른 worktree가 동일한 기억을 공유하도록 한다.

**생명주기**: 영구적, 사용자가 명시적으로 삭제하거나 Agent가 능동적으로 업데이트/삭제할 때까지

### 제3층: Team Memory(팀 공유 기억)

**파일 경로**: `~/.claude/projects/<sanitized-git-root>/memory/team/`(`getTeamMemPath()` 반환값)

Team Memory는 Persistent Memory의 하위 디렉터리로, REST API를 통해 동일 GitHub 저장소의 모든 인증된 멤버 간에 동기화된다. Auto Memory 위의 확장이며, `isTeamMemoryEnabled()`는 먼저 `isAutoMemoryEnabled()`를 확인하여 상위 시스템이 활성화되어 있는지 확인한다.

**생명주기**: Anthropic 서버에서 유지, 사용자를 넘어, 기기를 넘어 영속

---

## 9.4 MEMORY.md 인덱스 메커니즘

MEMORY.md는 Persistent Memory 층의 **인덱스 파일**이지 내용 파일이 아니다. 시스템은 여러 곳에서 이 둘을 명확히 구분한다:

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### 형식 규격

MEMORY.md의 각 줄은 특정 기억 파일로의 링크다:

```
- [사용자 선호: 간결한 응답](feedback_terse_responses.md) — 사용자가 응답 끝의 요약을 싫어함
- [프로젝트 배경: Auth 미들웨어 재작성](project_auth_rewrite.md) — 법무 컴플라이언스 요건, 기술 부채 아님
```

MEMORY.md는 매 세션 시작 시 시스템 프롬프트에 로드되므로 그 크기는 매 요청의 토큰 소비에 직접 영향을 미친다.

### 200줄 / 25KB 이중 제한

시스템은 `memdir/memdir.ts`에서 엄격한 이중 상한을 정의한다:

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

(`memdir/memdir.ts:30-33`)

잘라내기 로직은 `truncateEntrypointContent()`에 구현되어 있다(`memdir/memdir.ts:55-102`): 먼저 줄 수로 잘라내고, 그 다음 바이트 수로 잘라낸다(마지막 줄바꿈 문자에서 잘라내어 줄 중간 잘라내기 방지). 잘라낸 후에는 인덱스가 너무 길다는 경고 메시지를 추가한다.

**설계 의도**: 줄당 약 125자 × 200줄 ≈ 25KB. 이것은 실측(p97 분위)을 통한 합리적인 상한이다. 바이트 제한은 "200줄 이하지만 각 줄이 매우 긴" 경계 케이스에 대한 것이다(실측 p100: 197KB가 줄 수 제한을 초과하지 않음).

### 기억 파일과의 관계

기억 작성은 **두 단계 작업**이다:
1. 내용 파일 작성(`user_role.md`, `feedback_testing.md` 등)
2. MEMORY.md에 해당 항목 추가

읽을 때는 findRelevantMemories가 선택한 파일만 읽힌다(9.7 참조). MEMORY.md 자체는 시스템 프롬프트에 항상 상주한다.

---

## 9.5 네 가지 기억 유형

시스템은 모든 기억을 네 가지 유형으로 제한한다. 이것이 설계에서 가장 중요한 결정 중 하나다. 유형 정의는 `memdir/memoryTypes.ts`에 있다(`MEMORY_TYPES` 상수):

### user 유형

**적용 시나리오**: 사용자의 역할, 목표, 책임, 지식 배경

**트리거 시점**: 사용자의 역할, 선호, 직무, 지식 수준을 알게 된 언제든지

**용도**: 특정 사용자의 인지 수준과 필요에 맞게 응답 방식 조정

**범위**: 항상 private(개인 전용), Team Memory 모드에서도 마찬가지

**반면 사례(저장하지 말아야 할 내용)**:
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### feedback 유형

**적용 시나리오**: 작업 방식에 대한 사용자의 수정과 확인——"이렇게 하지 마라"뿐만 아니라 "계속 이렇게 해라"도 포함

**구조 요건**:
- 규칙 자체
- `**Why:**` 줄(이유 제공, 경계 케이스에서 적용 여부 판단 가능하도록)
- `**How to apply:**` 줄(언제 어디서 효력이 있는지)

**독특한 설계**: 실패 교훈과 성공 확인 **모두** 기록해야 한다고 명시:

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**트리거 시점**: 사용자가 "이렇게 하지 마라"(명시적 수정) 또는 "바로 이거야"/"완벽해"(암묵적 확인, 더 파악하기 어려움)라고 말할 때

**범위**: 기본적으로 private; 지침이 명확히 프로젝트 수준 규범일 때만(예: 테스트 전략, 빌드 제약) team으로 저장

### project 유형

**적용 시나리오**: 진행 중인 작업, 목표, 계획, 버그 또는 이벤트에 관한 정보로, **코드나 git 기록에서 파생할 수 없는** 것

**구조 요건**:
- 사실/결정 자체
- `**Why:**` 줄(동기——보통 제약 조건, 마감일 또는 이해관계자 요구)
- `**How to apply:**` 줄(제안에 어떤 영향을 미치는지)

**중요 규칙**: 저장 시 상대적 날짜를 절대적 날짜로 변환해야 함("다음 주 목요일" → "2026-04-08"). 시간이 지나도 기억을 해석할 수 있도록.

**범위**: 기본적으로 team(프로젝트 컨텍스트는 본질적으로 공유됨)

**감퇴 특성**: project 유형 기억은 감퇴가 가장 빠르며, Why 필드는 기억이 여전히 유효한지 판단하는 데 도움이 됨

### reference 유형

**적용 시나리오**: 외부 시스템의 정보 위치를 가리키는 포인터(Linear 프로젝트, Slack 채널, Grafana 대시보드 등)

**트리거 시점**: 외부 리소스의 위치와 용도를 알게 될 때

**범위**: 보통 team

**전형적 예시**:

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### 저장하지 말아야 할 내용(명시적 배제)

`WHAT_NOT_TO_SAVE_SECTION`은 저장하지 말아야 할 여섯 가지 내용을 명시한다(`memdir/memoryTypes.ts:196-207`):

1. 코드 패턴, 컨벤션, 아키텍처, 파일 경로——프로젝트 현재 상태에서 파생 가능
2. Git 기록, 최근 변경——`git log`/`git blame`이 권위 있는 출처
3. 디버그 솔루션 또는 수정 방법——수정은 코드에, 컨텍스트는 커밋 메시지에
4. 이미 CLAUDE.md에 기록된 내용
5. 임시 작업 세부 사항: 진행 중인 작업, 임시 상태, 현재 세션 컨텍스트
6. **사용자가 명시적으로 저장을 요청한 위의 내용들도**——사용자가 PR 목록을 저장하라고 요청하면 "예상치 못하거나 명확하지 않은 내용이 있나요? 그게 저장할 가치가 있는 것입니다"라고 물어봐야 함

---

## 9.6 자동 기억 추출

### Fork Agent 자동 추출 메커니즘

기억 추출은 "Fork Agent" 패턴을 사용한다——메인 세션과 완전히 동일한 Agent 컨텍스트를 생성하여, 백그라운드에서 비동기적으로 실행하면서 메인 대화를 차단하지 않는다.

이 메커니즘의 핵심은 `runForkedAgent()`로, 추출 Agent는 부모 세션의 prompt cache를 공유하여 최대한의 캐시 히트율을 달성한다:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // 메인 세션 기록에 쓰지 않음, 경쟁 조건 방지
  maxTurns: 5,            // 검증 무한 루프 방지 하드 상한
})
```

(`services/extractMemories/extractMemories.ts:258-267`)

`maxTurns: 5`의 설계 주석이 의도를 설명한다:

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

추출 Agent의 효율적인 전략은 명확히 "2턴 완성"으로 설계되었다:
- **1턴**: 업데이트가 필요한 모든 파일의 FileRead 요청을 병렬로 발송
- **2턴**: 모든 FileWrite/FileEdit 요청을 병렬로 발송

### 트리거 시점(Stop Hooks)

추출은 **완전한 쿼리 루프가 끝날 때마다** 트리거된다——즉 모델이 tool_use 없는 최종 응답을 생성할 때, `handleStopHooks`를 통해 `executeExtractMemories()`를 호출한다.

상태는 클로저로 관리되며, 핵심 변수는 다음과 같다:

```typescript
let lastMemoryMessageUuid: string | undefined    // 커서: 지난번 추출 위치
let inProgress = false                           // 동시 실행 방지
let pendingContext: {...} | undefined            // 실행 중에 들어온 호출을 여기 저장
let turnsSinceLastExtraction = 0                // 스로틀링 제어
```

(`services/extractMemories/extractMemories.ts:225-240`)

**동시성 제어 전략**: 추출이 진행 중일 때 새 호출이 들어오면, 새 호출은 폐기되는 것이 아니라 `pendingContext`에 "저장"된다. 현재 추출이 완료되면 즉시 최신 컨텍스트로 "후행 추출"을 한 번 더 실행하여 마지막 메시지 배치가 누락되지 않도록 한다.

**상호 배제 규칙**: 메인 Agent가 직접 기억 파일을 썼다면(`hasMemoryWritesSince` 감지), Fork Agent는 이번 추출을 건너뛰고 커서만 진행한다:

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // 메인 Agent가 썼음, fork agent 건너뜀, 커서 진행
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

(`services/extractMemories/extractMemories.ts:198-209`)

### 추출 프롬프트 분석

추출 프롬프트의 핵심 설계 철학은 **정보 효율**이다:

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // 기존 기억 목록 사전 주입, Agent가 ls에 한 턴 낭비하는 것 방지
  ].join('\n')
}
```

(`services/extractMemories/prompts.ts:20-47`)

기존 기억 목록(`existingMemories`) 사전 주입이 핵심 최적화다——Agent가 디렉터리 나열에 한 턴을 낭비하지 않도록, prompt에서 직접 구조화된 파일 목록(파일명, 유형, 타임스탬프, description)을 제공한다.

### Session Memory의 트리거 메커니즘

Session Memory는 다른 트리거 메커니즘을 사용한다——Stop Hooks 대신 `postSamplingHooks`를 통해, 매 모델 샘플링 후 업데이트가 필요한지 평가한다:

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

(`services/SessionMemory/sessionMemory.ts:130-150`)

기본 트리거 임계값(`DEFAULT_SESSION_MEMORY_CONFIG`, `services/SessionMemory/sessionMemoryUtils.ts:29-33`):

| 파라미터 | 기본값 | 설명 |
|-----|-------|------|
| `minimumMessageTokensToInit` | 10,000 | 세션 기억 초기화에 필요한 최소 토큰 수 |
| `minimumTokensBetweenUpdate` | 5,000 | 두 업데이트 사이에 최소 증가해야 할 토큰 수 |
| `toolCallsBetweenUpdates` | 3 | 두 업데이트 사이에 필요한 최소 도구 호출 수 |

이 값들은 GrowthBook 원격 설정(`tengu_sm_config`)으로 동적 조정 가능하다.

---

## 9.7 지능적 기억 소환

### Sonnet이 최대 5개의 관련 기억 선택

기억 소환은 전체 읽기가 아니라 **먼저 frontmatter를 스캔한 후 Sonnet을 사용하여 가장 관련성 높은 최대 5개를 선택**하는 방식이다.

핵심 흐름은 `findRelevantMemories()`에 있다(`memdir/findRelevantMemories.ts:32-66`):

1. `scanMemoryFiles()`가 기억 디렉터리를 스캔하여 각 파일의 처음 30줄(frontmatter)을 읽고 `MemoryHeader[]`를 반환
2. 이전 몇 턴에서 이미 표시된 기억을 필터링(`alreadySurfaced`), 새 내용을 위한 5개 슬롯 확보
3. Sonnet을 사용하여 `selectRelevantMemories()`를 호출, query와 파일 description 기반으로 가장 관련성 높은 파일명 선택
4. 선택된 기억의 경로와 mtime 반환

### 관련성 판단 로직

Sonnet의 시스템 프롬프트는 신중하게 설계되었다:

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

(`memdir/findRelevantMemories.ts:13-23`)

**핵심 설계**: 최근 사용한 도구의 참조 문서는 선택하지 않아야 하며(사용 중에는 참조 문서가 필요 없음), 하지만 동일 도구의 **함정/알려진 문제** 기억은 여전히 선택해야 한다(사용 중에 함정 경고가 가장 필요함).

API 호출은 구조화된 출력(JSON Schema)을 사용하여 반환 형식이 파싱 가능하도록 한다:

```typescript
output_format: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      selected_memories: { type: 'array', items: { type: 'string' } },
    },
    required: ['selected_memories'],
    additionalProperties: false,
  },
},
```

(`memdir/findRelevantMemories.ts:97-108`)

### 컨텍스트에 기억 주입 방식

선택된 기억은 `<system-reminder>` 태그로 감싸여 사용자 메시지 앞에 주입된다(`wrapMessagesInSystemReminder`). 1일 이상 된 기억에는 신선도 경고가 추가된다:

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

(`memdir/memoryAge.ts:38-47`)

이 설계는 실제 문제를 해결한다: 사용자들이 "만료된 기억을 바탕으로 자신 있게 단언"하는 문제를 보고했다——참조된 파일 경로나 함수명이 이미 수정되었지만, 기억의 citation이 단언을 더 의심스럽게 만들기보다 더 신뢰할 수 있는 것처럼 보이게 했다.

**드리프트 방지 메커니즘**: `MEMORY_DRIFT_CAVEAT`가 시스템 프롬프트에 주입되어, Agent가 기억에 기반하여 답변하기 전에 현재 상태를 검증하도록 요구한다:

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Team Memory 동기화

### REST API 동기화 메커니즘

Team Memory는 `services/teamMemorySync/`를 통해 서버 측 동기화를 구현한다. API 설계는 `index.ts` 상단에 완전히 설명되어 있다:

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → 메타데이터+해시만
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert 항목
404  = 아직 데이터 없음
```

(`services/teamMemorySync/index.ts:10-13`)

동기화는 **OAuth 인증**에 의존한다(`CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE` 필요), GitHub 저장소(`owner/repo`)를 범위로 사용한다.

**Watcher 메커니즘**: `watcher.ts`는 `fs.watch({recursive: true})`를 사용하여 team 디렉터리 변경을 감시하고, 2초 디바운스 후 push를 트리거한다(`DEBOUNCE_MS = 2000`). chokidar 대신 네이티브 `fs.watch`를 의도적으로 선택:

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS는 FSEvents(O(1) 파일 디스크립터), Linux는 inotify(O(하위 디렉터리 수))를 사용하며, 모두 chokidar의 kqueue 방식보다 우수하다.

### 낙관적 잠금(If-Match)

업로드는 `If-Match` HTTP 헤더에 ETag(checksum)를 전달하여 낙관적 동시성 제어를 사용한다:

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

(`services/teamMemorySync/index.ts:uploadTeamMemory`)

서버가 412 Precondition Failed를 반환하면 충돌이 발생한 것이다(다른 사용자가 그 사이에 공유 기억을 수정함). 시스템은 `GET ?view=hashes` 엔드포인트(경량, 각 key의 SHA-256 해시만 반환, 내용 본문 없음)를 사용하여 로컬의 `serverChecksums`를 새로 고치고 delta를 재계산하여 재시도한다:

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### 충돌 해결 전략

충돌 해결 전략은 **서버 승리(server wins per-key)**다——Pull 시 서버 내용이 로컬을 덮어씀. Delta push는 로컬과 서버 해시가 다른 key만 업로드하며, 서버는 upsert 의미론을 사용한다(PUT에 없는 key는 보존).

배치 업로드 제한(`MAX_PUT_BODY_BYTES = 200_000`)은 요청 본문이 너무 커서 API Gateway에 거부되는 것을 방지한다(약 256-512KB에서 HTML 형식의 413을 반환하는 게이트웨이를 관측했으며, 이는 애플리케이션 층의 구조화된 413과 다름). 한도 초과 시 자동으로 여러 개의 순차적 PUT으로 분할하며, upsert 의미론이 안전성을 보장한다:

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // 탐욕적 빈 패킹: 바이트 수로 배치, 각 배치는 MAX_PUT_BODY_BYTES 초과하지 않음
  ...
}
```

(`services/teamMemorySync/index.ts:batchDeltaByBytes`)

**영구 실패 억제**: 일부 오류(no_oauth, no_repo, 409/429 제외 4xx)는 재시도로 자가 복구가 불가능하다. 시스템이 이런 오류를 감지하면 `pushSuppressedReason`을 설정하여 watcher가 트리거하는 push가 무한 재시도 루프에 빠지지 않도록 한다(OAuth 없는 기기가 2.5일 동안 167K 이상의 push 이벤트를 보내는 것이 관측된 적 있음).

---

## 9.9 설계 결정 분석

### 데이터베이스 대신 파일 시스템을 사용하는 이유

파일 시스템 + Markdown 설계에는 몇 가지 핵심 장점이 있다:

1. **Agent가 직접 조작 가능**: FileRead/FileWrite/FileEdit 도구가 Claude의 네이티브 도구다. Agent가 기억을 쓰는 것과 코드를 쓰는 것에 동일한 도구 세트를 사용하므로 인지 부담이 최소화된다.

2. **사용자가 검사 가능**: `~/.claude/projects/.../memory/`는 일반 폴더로, 사용자가 직접 `ls`, `cat`, `vim`을 사용할 수 있어 완전히 투명하다.

3. **Git 친화적**: Markdown 파일은 자연스럽게 diff, grep, git history를 지원하여 Team Memory의 delta 계산을 용이하게 한다.

4. **불필요한 추상화 방지**: 데이터베이스는 스키마 마이그레이션, 백업 전략, 쿼리 층이 필요하다——"수백 KB의 Markdown 파일"에는 과도한 엔지니어링이다.

### MEMORY.md 크기를 제한하는 이유

200줄 / 25KB 제한의 배후에는 실측 데이터(p97/p100 관측값)가 있다. 핵심 이유:

- MEMORY.md는 **매 요청**마다 시스템 프롬프트에 로드되며, 크기가 토큰 소비에 직접 영향을 미침
- 너무 큰 인덱스는 실제로 유용한 컨텍스트 공간을 침범함
- 강제 제한은 사용자와 Agent가 인덱스를 정제되게 유지하도록 유도하며, 각 줄에 내용이 아닌 "훅"만 작성하게 함

이것은 "제약을 통해 품질을 촉진"하는 전형적인 설계다——기술적으로 더 많이 수용할 수 없어서가 아니라, 제약을 통해 올바른 사용 방식을 유도하는 것이다.

### 기억 보안의 설계 고려사항

시스템에는 다층 보안 설계가 있다:

**경로 순회 방지**: `teamMemPaths.ts`는 세 겹의 검사를 구현한다——먼저 문자열 수준에서 `..`, URL 인코딩 순회, Unicode 정규화 공격을 확인하고, 그 다음 `realpath`를 통해 심볼릭 링크를 해석하여 실제 파일 시스템 경로를 검증한다:

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

(`memdir/teamMemPaths.ts:130-133`)

**시크릿 스캔**: Team Memory에 쓸 때, `scanForSecrets()`가 30가지 고신뢰도 자격증명 패턴을 스캔한다(gitleaks 규칙 라이브러리에서, AWS, GCP, GitHub, Anthropic, OpenAI 등 주요 플랫폼의 토큰 형식 포함). 스캔은 **업로드 전**과 **쓰기 전** 두 번 실행된다:

- `teamMemSecretGuard.ts`의 `checkTeamMemSecrets()`는 FileWriteTool/FileEditTool의 `validateInput` 단계에서 쓰기를 차단
- `readLocalTeamMemory()`는 push 전에 다시 스캔하여 민감한 정보가 포함된 파일을 건너뜀

**최소 권한 도구 제어**: 추출 Agent의 `canUseTool` 함수는 다음만 허용한다:
- FileRead/Grep/Glob(읽기 전용)
- 읽기 전용 Bash 명령(ls/find/cat/stat/wc/head/tail)
- FileEdit/FileWrite이며 경로가 반드시 memory 디렉터리 내에 있어야 함

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

(`services/extractMemories/extractMemories.ts:171-176`)

**ProjectSettings 보안 예외**: `autoMemoryDirectory` 설정은 policy/local/user 소스만 신뢰하며, projectSettings(`.claude/settings.json`)를 명시적으로 배제한다:

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 이식 가능한 패턴

Doramagic 기억 시스템 설계에서 직접 참고할 수 있는 패턴:

### 패턴 1: 파생 불가능 원칙

**무엇을 기억해야 하는가**: 현재 상태(코드, 파일, git)를 쿼리하여 얻을 수 있는 정보는 기억할 가치가 없다. 기억은 "역사적 컨텍스트"——왜 이 결정을 내렸는지, 어떤 함정을 밟았는지, 사용자의 암묵적 선호——만 저장해야 한다.

**Doramagic 적용**: Soul Extractor가 추출하는 "UNSAID"와 "WHY" 층은 자연스럽게 이 원칙에 부합한다. OpenClaw 규칙 문서는 쿼리 가능하므로 기억할 필요가 없지만; "이 OpenClaw 규칙이 발행 실패를 초래한 적 있다"는 교훈은 기억할 가치가 있다.

### 패턴 2: 두 단계 쓰기 + 경량 인덱스

파일 + 인덱스의 두 단계 쓰기 패턴은 인덱스가 항상 정제되도록 보장하며(각 줄 150자 이내 강제 제약), 내용 파일은 자세하게 펼칠 수 있다. 인덱스의 토큰 소비는 고정되며, 내용 읽기는 수요에 따라 이루어진다.

**Doramagic 적용**: 기억 시스템의 `MEMORY.md`는 Doramagic의 "적목 목차"와 유사하다——경량으로 로드 가능한 인덱스가 필요에 따라 펼칠 수 있는 상세 파일을 가리킨다.

### 패턴 3: Fork Agent 백그라운드 추출

메인 대화를 차단하지 않고, prompt cache를 공유하며, 캐시 히트율을 최대화하는 것이 백그라운드 후처리 작업의 표준 패턴이다. 핵심 구현 세부사항:
- `skipTranscript: true`로 메인 세션 기록 쓰기 방지
- `maxTurns: N`으로 Agent가 검증 루프에 빠지는 것 방지
- 커서 메커니즘(`lastMemoryMessageUuid`)으로 매번 증분만 처리
- Stash + 후행 실행으로 Agent가 바쁠 때 최신 메시지가 누락되지 않도록 보장

### 패턴 4: 신선도 인식

기억은 영구적으로 유효한 사실이 아니라 시효가 있는 관찰이다. 시스템은 다음을 통해:
1. 소환 시 "N일 전"의 나이 힌트 첨부
2. 시스템 프롬프트에 드리프트 방지 지시사항 삽입(먼저 검증한 후 참조)
3. Agent가 만료된 기억을 발견할 때 보존하는 대신 능동적으로 업데이트하도록 요구

Doramagic의 "지식 추출" 시나리오에 특히 관련이 있다——추출된 WHY/UNSAID는 프로젝트 진화에 따라 만료될 것이며, 유사한 메커니즘으로 신선도를 유지해야 한다.

### 패턴 5: 시크릿 스캔 전치

"경계를 넘는" 쓰기(공유 공간에 쓰기, 네트워크 업로드) 전에 시크릿을 스캔해야 한다. gitleaks 규칙 라이브러리는 고신뢰도 패턴 집합을 제공하며 직접 재사용할 수 있다. 핵심 설계: 스캔은 쓰기 도구의 `validateInput` 단계에서 실행되며(사후가 아님), 시크릿이 어떤 영속화 경로에도 닿지 않도록 보장한다.

---

## 9.11 소스 코드 인덱스

| 파일 | 줄수 | 핵심 역할 |
|-----|------|---------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Session Memory 주 로직: 트리거 조건 판단, Fork Agent 호출, 수동 트리거 API |
| `services/SessionMemory/prompts.ts` | 324 | Session Memory 템플릿, 업데이트 prompt 구축, 섹션 크기 분석 |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Session Memory 상태 관리: 설정, 임계값 판단, 대기/동기화 유틸리티 |
| `services/extractMemories/extractMemories.ts` | 615 | Persistent Memory 추출: Fork Agent 호출, 클로저 상태, 동시성 제어 |
| `services/extractMemories/prompts.ts` | 154 | 추출 prompt 구축: auto-only와 combined(Team Memory 포함) 두 변형 |
| `memdir/memdir.ts` | 507 | MEMORY.md 잘라내기 로직, 기억 prompt 구축, Directory 생성 보장 |
| `memdir/paths.ts` | 278 | Auto Memory 경로 해석, 활성화/비활성화 판단, 경로 보안 검증 |
| `memdir/memoryTypes.ts` | 271 | 네 가지 기억 유형 정의, frontmatter 형식, 소환/드리프트 방지/파생 불가 원칙 |
| `memdir/findRelevantMemories.ts` | 141 | Sonnet 소환 선택: frontmatter 스캔 → 5개 관련 기억 |
| `memdir/memoryScan.ts` | 94 | 디렉터리 스캔 프리미티브: frontmatter 읽기, 목록 형식화 |
| `memdir/memoryAge.ts` | 53 | 신선도 계산: 일수, human-readable 텍스트, 만료 경고 |
| `memdir/teamMemPaths.ts` | 292 | Team Memory 경로, 경로 순회 방지(3겹 검증), 심볼릭 링크 해석 |
| `memdir/teamMemPrompts.ts` | 100 | Team Memory + Auto Memory 병합 prompt 구축 |
| `services/teamMemorySync/index.ts` | 1256 | 동기화 핵심: fetch/push 로직, 낙관적 잠금, 배치 분할, 충돌 재시도 |
| `services/teamMemorySync/watcher.ts` | 387 | 파일 감시: 디바운스 push, 영구 실패 억제, 시작/중지 생명주기 |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30가지 시크릿 스캔 규칙(gitleaks 하위 집합), redact 유틸리티 |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema: TeamMemoryData, 동기화 결과 유형, SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | 쓰기 전 시크릿 차단: FileWriteTool/FileEditTool validateInput 통합 |

**핵심 상수 빠른 참조**:

| 상수 | 값 | 위치 |
|-----|---|------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH`(Session Memory 섹션별) | 2,000 토큰 | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 토큰 | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10,000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5,000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| 소환 상한 | 5개 | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| 기억 파일 수 상한 | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Frontmatter 읽기 줄 수 | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Team Memory 타임아웃 | 30,000ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Push 디바운스 지연 | 2,000ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| 단일 파일 크기 상한 | 250,000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| PUT 요청 본문 상한 | 200,000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
| 시크릿 스캔 규칙 수 | 30 | `secretScanner.ts:SECRET_RULES` |
