# 제 6 장: Skill 시스템

## 6.1 개요 및 위치

Skill 시스템은 Claude Code에서 가장 혁신적인 아키텍처 중 하나입니다. 재사용 가능한 워크플로우를 Markdown 파일로 인코딩하고, 슬래시 명령（`/skill-name`）이나 AI가 능동적으로 호출하는 방식으로 트리거합니다. 본질적으로 Skill은 "AI를 위한 SOP（Standard Operating Procedure）"입니다—인간 전문가가 복잡한 작업을 수행하는 단계, 판단 조건, 성공 기준을 구조화된 Markdown 형식으로 기록하여, AI가 재현 가능한 전문적 실행 능력을 갖출 수 있도록 합니다.

일반 Prompt와 다르게, Skill 시스템은 다음과 같은 핵심 특징을 가집니다:

1. **선언형 + 실행형 융합**: Frontmatter는 메타데이터를 선언하고（권한, 모델, 트리거 조건）, 본문은 실행 지시입니다
2. **다중 소스 로딩**: 내장（bundled）, 사용자 수준, 프로젝트 수준, Plugin 수준, MCP 소스로 우선순위에 따라 병합
3. **두 가지 실행 모드**: inline（현재 세션 컨텍스트에 주입）과 fork（독립적인 sub-agent에서 격리 실행）
4. **조건부 활성화**: `paths` frontmatter를 통해 파일 경로에 따라 자동 활성화되는 Skill 구현
5. **동적 발견**: 세션 과정에서 사용자가 파일을 조작함에 따라 자동으로 더 깊은 디렉터리의 Skill을 발견하고 로딩

Skill 시스템은 단순한 명령 별칭이 아니라 완전한 워크플로우 오케스트레이션 프레임워크입니다.

---

## 6.2 이론적 기반

### 재사용 가능한 워크플로우（Reusable Workflows）의 설계 패턴

Skill 시스템은 AI 도구 사용에서 핵심 문제를 해결합니다: **전문 지식을 어떻게 축적하고 재현 가능하게 만드는가?** 전통적인 코드 재사용은 함수와 클래스를 통해 이루어지지만, AI가 실행하는 "지식"은 자연어로 기술된 워크플로우로 코드 함수로 직접 캡슐화할 수 없습니다.

Skill의 설계는 SOP（Standard Operating Procedure）의 아이디어를 차용합니다—전문가의 실행 프로세스, 결정 포인트, 성공 기준을 구조화하여 기록함으로써 AI가 매번 동일한 고품질 경로를 따르도록 합니다.

### 선언형 vs 명령형 워크플로우 정의

Skill 시스템은 두 가지 스타일을 동시에 지원합니다:

- **선언형**: frontmatter를 통해 `allowed-tools`, `model`, `context` 등의 속성을 선언하고, 시스템이 권한 제어 및 실행 컨텍스트 구성을 자동으로 처리
- **명령형**: Skill 본문에 shell 명령（`!``command``）을 직접 삽입하여 "설명 속에 작업"을 구현

### Markdown-as-Code 이념

JSON/YAML 대신 Markdown을 Skill 형식으로 선택한 것은 심사숙고한 설계 결정입니다:

- **인간 가독성**: 개발자가 Skill을 직접 읽고 편집하면서 의도를 이해할 수 있음
- **AI에게 자연스러움**: AI 훈련 데이터에 Markdown이 대량 포함되어 있어 AI가 JSON보다 Markdown을 더 자연스럽게 이해함
- **점진적 구조화**: 순수 산문에서 시작하여 점차 제목, 단계, 규칙을 추가할 수 있으며 완전한 구조를 강제하지 않음
- **버전 관리 친화적**: Markdown diff는 인간이 읽기 쉬워 코드 리뷰 시 워크플로우 변경 사항을 한눈에 파악 가능

---

## 6.3 Skill 형식 및 데이터 구조

### Skill Markdown 파일의 형식 규범

Skill 파일은 고정된 디렉터리 구조를 따릅니다:

```
.claude/skills/<skill-name>/SKILL.md
```

파일 형식은 frontmatter + Markdown 본문입니다:

```markdown
---
name: my-skill
description: 이 Skill이 무엇을 하는지 한 문장으로 설명
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  사용자가 ...하고 싶을 때 사용합니다. 예: "cherry-pick to release", "hotfix".
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## 단계

### 1. 첫 번째 단계
구체적인 작업...

**성공 기준**: 이 단계가 완료되었음을 증명하는 체크포인트
```

### frontmatter 필드 상세

다음은 `parseSkillFrontmatterFields` 함수（`loadSkillsDir.ts:184`）가 파싱하는 모든 필드입니다:

| 필드 | 타입 | 설명 |
|------|------|------|
| `name` | string | 표시 이름（디렉터리 이름과 다를 수 있음） |
| `description` | string | 한 문장 설명, `/help`에 표시 |
| `allowed-tools` | string[] | 화이트리스트 도구 목록, `Bash(git:*)` 접두사 패턴 지원 |
| `argument-hint` | string | 사용자 트리거 시 파라미터 힌트, 예: `"<branch-name>"` |
| `arguments` | string[] | 파라미터 이름 목록, `$arg_name` 변수 치환에 사용 |
| `when_to_use` | string | AI에게 이 Skill을 능동적으로 호출할 시기 알려줌, 트리거 구문 포함 |
| `version` | string | Skill 버전 번호 |
| `model` | string | 모델 오버라이드, 예: `opus`, `sonnet`, `inherit`는 상속을 의미 |
| `disable-model-invocation` | boolean | AI 능동적 호출 금지, 사용자 수동 트리거만 가능 |
| `user-invocable` | boolean | `/help`에서 표시 여부（기본값 `true`） |
| `context` | `"fork"` | 설정 시 독립적인 sub-agent에서 실행 |
| `agent` | string | agent 타입 지정 |
| `effort` | EffortValue | 모델 사고 깊이에 영향 |
| `paths` | string[] | gitignore 문법의 경로 패턴, 조건부 활성화에 사용 |
| `hooks` | HooksSettings | Skill 실행 중 hook 구성 |
| `shell` | FrontmatterShell | 인라인 shell 명령 실행 구성 |

### SkillDefinition 타입

`bundledSkills.ts`에 `BundledSkillDefinition`이 정의되어 있고（12-41행）, 파일 시스템 Skill은 `Command` 타입（`src/types/command.js`）에 해당합니다. 두 가지는 `createSkillCommand`（`loadSkillsDir.ts:269`）에서 통합된 `Command` 객체로 수렴됩니다:

```typescript
// loadSkillsDir.ts:316-400
return {
  type: 'prompt',
  name: skillName,
  description,
  allowedTools,
  argumentHint,
  argNames: argumentNames.length > 0 ? argumentNames : undefined,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context: executionContext,
  agent,
  effort,
  paths,
  contentLength: markdownContent.length,
  isHidden: !userInvocable,
  progressMessage: 'running',
  loadedFrom,
  hooks,
  skillRoot: baseDir,
  async getPromptForCommand(args, toolUseContext) { ... }
} satisfies Command
```

---

## 6.4 Skill 로딩 메커니즘

### loadSkillsDir의 완전한 로딩 흐름

`getSkillDirCommands`（`loadSkillsDir.ts:638`）는 전체 로딩 흐름의 진입점으로, `lodash-es/memoize`를 사용하여 결과를 캐시하고 중복 I/O를 방지합니다:

```
시작 시
  ├── policySettings: ~/.claude-managed/.claude/skills/（기업 관리）
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/（cwd에서 home까지 위로 순회）
  ├── additionalDirs: --add-dir로 지정된 추가 디렉터리
  └── legacyCommands: .claude/commands/（하위 호환성）

세션 중（동적 발견）
  └── 사용자가 파일을 읽고 쓸 때 → discoverSkillDirsForPaths() → addSkillDirectories()
```

로딩 결과는 `realpath`로 중복을 제거하여（`loadSkillsDir.ts:728-763`）, symlink로 인한 중복 로딩을 방지합니다.

### 다중 소스 로딩 우선순위

코드 주석에 로딩 우선순위가 명확히 설명되어 있습니다（`loadSkillsDir.ts:677-714`）:

```
managed（기업 정책） < user（사용자 수준） < project（프로젝트 수준） < additional（--add-dir）
```

이것은 "더 구체적일수록 우선순위가 높다"는 원칙입니다: 프로젝트 수준이 사용자 수준을 덮어씁니다. 프로젝트에는 특정 요구사항이 있기 때문입니다.

**특수 케이스:**
- `--bare` 모드: 자동 발견을 건너뛰고 `--add-dir`로 명시적으로 지정된 디렉터리만 로딩
- `skillsLocked`（plugin-only policy）: 사용자/프로젝트 수준 Skill 로딩 금지, Plugin 소스만 허용
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS` 환경 변수: managed 수준 Skill 건너뜀

### Skill 발견 및 매칭 로직

**정적 발견**（시작 시）: `getSkillDirCommands`가 각 수준의 `~/.claude/skills/` 디렉터리를 스캔하며, 디렉터리 형식（`skill-name/SKILL.md`）만 지원하고 단일 `.md` 파일은 지원하지 않습니다.

**동적 발견**（세션 중）: 사용자가 파일을 읽고 쓸 때, `discoverSkillDirsForPaths`（`loadSkillsDir.ts:861`）가 파일 경로를 따라 위로 탐색하면서 각 디렉터리에 `.claude/skills/`가 있는지 확인하고, 발견 시 `addSkillDirectories`로 로딩합니다. `.gitignore`로 표시된 디렉터리는 건너뜁니다（`node_modules`의 Skill 오염 방지）.

**조건부 활성화**（paths frontmatter）: `paths` 필드가 있는 Skill은 초기에 모델에게 보이지 않고 `conditionalSkills` Map에 저장됩니다. 사용자가 경로와 일치하는 파일을 조작할 때, `activateConditionalSkillsForPaths`（`loadSkillsDir.ts:997`）가 `ignore` 라이브러리（gitignore 문법）로 매칭하고, 일치하면 `dynamicSkills`로 이동하여 활성화합니다.

---

## 6.5 SkillTool 실행 흐름

### /skill-name에서 실행까지의 완전한 경로

`SkillTool`（`tools/SkillTool/SkillTool.ts:330`）은 표준 `Tool` 구현으로, AI가 이 도구를 호출하여 Skill을 실행합니다. 완전한 실행 경로:

```
사용자가 /skill-name을 입력하거나 AI가 SkillTool을 호출하기로 결정
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── 선행 슬래시 제거（호환성 처리）
  │     ├── _canonical_ 접두사 확인（원격 Skill, 실험적）
  │     ├── findCommand()로 등록된 Command 검색
  │     ├── disableModelInvocation 플래그 확인
  │     └── type === 'prompt' 확인
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── deny 규칙 확인
  │     ├── 원격 canonical Skill 확인（자동 허용）
  │     ├── allow 규칙 확인
  │     ├── skillHasOnlySafeProperties() → 안전 Skill 자동 허용
  │     └── 기본: 팝업으로 사용자에게 묻기（behavior: 'ask'）
  │
  └── call (SkillTool.ts:580)
        ├── context === 'fork' 확인 → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent()（독립적인 sub-agent）
        └── 그 외（inline）→ processPromptSlashCommand()
              └── newMessages + contextModifier를 현재 세션에 주입
```

### Skill 컨텍스트 주입

Inline 실행 시 `call`이 `newMessages`와 `contextModifier`를 반환합니다（`SkillTool.ts:767-840`）:

- **newMessages**: Skill 전개 후의 메시지 목록, 현재 대화 컨텍스트에 주입
- **contextModifier**: `ToolUseContext`를 수정하는 함수로 다음에 사용:
  - `allowedTools` 누적（Skill이 선언한 도구 권한）
  - `mainLoopModel` 오버라이드（Skill이 model을 지정한 경우）
  - `effortValue` 오버라이드（Skill이 effort를 지정한 경우）

주목할 점은 `contextModifier`가 체인 호출 패턴을 채택한다는 것입니다（`SkillTool.ts:777`）. 여러 contextModifier가 누적되는 경우를 올바르게 처리하며 단순히 덮어쓰지 않습니다.

### Skill 변수 치환

`createSkillCommand`의 `getPromptForCommand`（`loadSkillsDir.ts:343-398`）는 Skill 내용을 반환하기 전 다음 치환을 수행합니다:

1. **파라미터 치환**: `$arg_name` → `substituteArguments()`로 사용자가 전달한 파라미터를 주입
2. **디렉터리 변수**: `${CLAUDE_SKILL_DIR}` → Skill 파일이 위치한 디렉터리의 절대 경로
3. **Session ID**: `${CLAUDE_SESSION_ID}` → 현재 세션 ID
4. **Shell 명령 실행**: `!``command`` ` → 실행 결과를 인라인으로（MCP Skill이 아닌 경우만）

MCP Skill은 Shell 명령 실행을 비활성화합니다（`loadSkillsDir.ts:372`）. 원격의 신뢰할 수 없는 Skill이 임의의 shell 명령을 주입하는 것을 방지합니다.

### Skill과 도구의 상호작용

Forked 실행 모드에서（`executeForkedSkill`, `SkillTool.ts:121`）, Skill은 완전히 격리된 sub-agent에서 실행됩니다:

- `runAgent()`를 통해 독립적인 agent를 시작하고, 독립적인 토큰 예산을 가짐
- 실행 과정의 tool use 메시지는 `onProgress` 콜백으로 보고되어 UI가 진행 상황을 표시할 수 있음
- 실행 결과는 `extractResultText`로 최종 텍스트를 추출하여 부모 agent에 반환
- `clearInvokedSkillsForAgent`로 메모리 해제（`SkillTool.ts:286`）

---

## 6.6 Bundled Skills 완전 목록 및 분석

내장 Skill은 `registerBundledSkill()`（`bundledSkills.ts:55`）을 통해 등록되며 CLI 시작 시 초기화됩니다. 다음은 전체 17개 내장 Skill에 대한 분석입니다:

### 1. `update-config`（`updateConfig.ts`, 475줄）

**기능**: Permissions, Hooks, Model, MCP 등 모든 구성 항목을 포함한 Claude Code의 `settings.json`을 구성합니다.

**특징**: Skill 본문이 동적으로 생성됩니다—`toJSONSchema(SettingsSchema())`를 사용하여 Zod schema에서 JSON Schema 문서를 자동 생성하여 문서가 실제 타입과 항상 동기화됩니다. 완전한 Hooks 문서（모든 Hook 이벤트, Hook 타입, JSON 출력 형식）를 포함합니다.

**트리거 시나리오**: 사용자가 동작 자동화, 권한 규칙, 환경 변수, 모델 설정을 구성하고 싶을 때.

### 2. `schedule`（`scheduleRemoteAgents.ts`, 447줄）

**기능**: 원격 스케줄링된 Agent（cron 트리거）를 관리하고, 예약 작업을 생성, 업데이트, 목록 조회, 실행합니다.

**특징**: 호출 전 여러 전제 조건을 확인합니다（OAuth tokens, 저장소 정보, MCP connectors, cloud environments）. 동적 정보를 Skill 프롬프트에 주입합니다. `AskUserQuestion` 도구로 사용자와 상호작용합니다.

**트리거 시나리오**: 사용자가 정기적으로 실행되는 Claude Code agent를 만들고 싶을 때（예: 일일 코드 리뷰, 자동 보고서）.

### 3. `keybindings-help`（`keybindings.ts`, 339줄）

**기능**: 사용자가 키보드 단축키를 사용자 정의하고, `~/.claude/keybindings.json`을 수정하도록 돕습니다.

**특징**: `generateContextsTable()`, `generateActionsTable()`을 통해 코드 상수에서 동적으로 문서를 생성하고, `generateReservedShortcuts()`로 다시 바인딩할 수 없는 단축키를 나열하여 사용자 오조작을 방지합니다.

**트리거 시나리오**: 사용자가 단축키를 다시 바인딩하거나, 조합 키를 추가하거나, 제출 키를 수정하고 싶을 때.

### 4. `lorem-ipsum`（`loremIpsum.ts`, 282줄）

**기능**: 토큰 계산 및 성능 테스트를 위해 고정 수량의 단일 토큰 단어 자리 표시자 텍스트를 생성합니다.

**특징**: API로 검증된 단일 토큰 단어 목록을 사용하여 `lorem` 파라미터가 토큰 수를 정확하게 제어할 수 있습니다. 벤치마킹 및 토큰 과금 분석에 자주 사용됩니다.

**트리거 시나리오**: 정확한 토큰 수의 테스트 텍스트가 필요할 때.

### 5. `skillify`（`skillify.ts`, 197줄）

**기능**: 현재 세션의 작업 과정을 재사용 가능한 SKILL.md 파일로 자동 변환합니다.

**특징**: 이것은 Skill 시스템의 "자기 번식" 메커니즘입니다. session memory와 사용자 메시지 이력을 읽어 사용자와 4번의 `AskUserQuestion` 대화로 워크플로우 이름, 단계, 파라미터, 트리거 조건을 확인하고, 최종적으로 표준 형식의 SKILL.md를 생성하여 디스크에 씁니다.

**제한**: `USER_TYPE === 'ant'`（Anthropic 내부 직원）에게만 사용 가능합니다.

**트리거 시나리오**: 세션 종료 시, 사용자가 방금 완료한 작업 흐름을 재사용 가능한 Skill로 고정화하고 싶을 때.

### 6. `claude-api`（`claudeApi.ts`, 196줄 + `claudeApiContent.ts`, 220줄）

**기능**: 개발자가 Claude API 또는 Anthropic SDK를 사용하여 애플리케이션을 구축하도록 돕습니다.

**특징**:
- 현재 프로젝트 언어를 자동 감지（파일 확장자 스캔으로 Python/TypeScript/Java/Go/Ruby/C#/PHP/curl 지원）
- 지연 로딩（247KB의 `.md` 내용은 호출 시에만 로딩）하여 시작 시간에 영향을 미치지 않음
- 언어별 API 문서, Agent SDK 패턴, 스트리밍 등 포함
- `files` 메커니즘을 통해 문서를 임시 디렉터리에 쓰고, 모델이 Read/Grep 도구로 필요에 따라 읽을 수 있음

**트리거 시나리오**: 코드에서 `anthropic`을 import하거나 사용자가 Claude API 사용 방법을 물을 때.

### 7. `batch`（`batch.ts`, 124줄）

**기능**: 대규모 코드 변경（마이그레이션, 리팩토링, 대량 이름 바꾸기 등）을 5-30개의 병렬 worktree agent로 분해하여 실행합니다.

**특징**: 3단계 실행 모델—Plan（Plan Mode 진입으로 깊이 연구 분해）→ Spawn Workers（`isolation: "worktree"`가 있는 background agent를 병렬 시작）→ Track Progress（실시간 상태 테이블 렌더링）. 각 worker는 독립적인 git worktree에서 작업하여 서로 영향을 주지 않고, 완료 후 PR을 엽니다.

**트리거 시나리오**: 대규모 코드 마이그레이션, 전체 저장소 리팩토링, 대량 수정.

### 8. `loop`（`loop.ts`, 92줄）

**기능**: 고정 간격으로 prompt 또는 슬래시 명령을 반복 실행합니다.

**특징**: 시간 간격을 지능적으로 파싱하여（`5m`, `2h` 접두사 형식과 `every 20m` 접미사 형식 지원）cron 표현식으로 변환하고, `ScheduleCronTool`을 호출하여 스케줄링된 작업을 등록합니다. 설정 후 첫 번째 스케줄링 트리거를 기다리지 않고 즉시 한 번 실행합니다.

**트리거 시나리오**: 사용자가 정기적으로 배포 상태를 확인하거나 특정 Skill을 주기적으로 실행하고 싶을 때.

### 9. `remember`（`remember.ts`, 82줄）

**기능**: auto-memory 항목을 검토하고, `CLAUDE.md`, `CLAUDE.local.md` 또는 팀 memory로 승격하도록 제안합니다.

**특징**: "먼저 제안 후 확인" 원칙을 채택하여 파일을 직접 수정하지 않고 분류 보고서（승격 대기/정리 대기/의심스러운/조치 불필요）를 표시하고, 사용자 승인 후에 실행합니다. 프로젝트 수준 규범（CLAUDE.md）, 개인 선호도（CLAUDE.local.md）, 조직 수준 지식（팀 memory）을 구분합니다.

**제한**: `USER_TYPE === 'ant'`이고 auto-memory 기능이 활성화된 경우에만 사용 가능합니다.

**트리거 시나리오**: 사용자가 memory를 정리하여 auto-memory가 무한정 축적되는 것을 방지하고 싶을 때.

### 10. `simplify`（`simplify.ts`, 69줄）

**기능**: 현재 git diff에 대해 3차원 코드 리뷰（코드 재사용, 코드 품질, 효율성）를 수행하고 발견된 문제를 직접 수정합니다.

**특징**: 세 개의 병렬 sub-agent를 동시에 시작하여 각각 담당합니다:
- **코드 재사용 Agent**: 바퀴의 재발명을 발견하고 기존 유틸리티 함수를 가리킴
- **코드 품질 Agent**: 중복 상태, 파라미터 팽창, 복사-붙여넣기, 누설된 추상화 등을 발견
- **효율성 Agent**: 불필요한 계산, 누락된 동시성, N+1 패턴, 메모리 누출 등을 발견

세 개의 agent가 완료된 후 발견 사항을 병합하고 직접 수정합니다. 보고만 하지 않습니다.

**트리거 시나리오**: 코드 일부를 완료한 후의 품질 검토. `batch` Skill의 worker 흐름에 의해 자동으로 호출되기도 합니다.

### 11. `debug`（`debug.ts`）

**기능**: 현재 Claude Code 세션의 디버그 로그를 진단하여 문제 해결을 돕습니다.

**특징**: tail 읽기（최대 64KB）를 통해 디버그 로그의 마지막 몇 줄을 읽어 긴 세션에서 로그 파일이 너무 커서 메모리 피크가 발생하는 것을 방지합니다. Anthropic 직원이 아닌 경우 먼저 debug logging을 활성화한 후 읽습니다. `disableModelInvocation: true`로 표시되어 AI가 자동으로 호출하지 못합니다（사용자 수동 트리거만 가능）.

### 12. `stuck`（`stuck.ts`）

**기능**: 컴퓨터의 다른 동결 또는 중단된 Claude Code 프로세스를 진단하고 보고서를 Slack 채널에 전송합니다.

**특징**: Anthropic 내부 진단 도구. 높은 CPU（≥90% 지속）, D 상태（I/O 중단）, T 상태（Ctrl+Z 중지）, Z 상태（좀비 프로세스）, 높은 메모리（≥4GB） 등의 이상을 감지합니다. 두 메시지 구조를 사용하여 Slack 보고서를 전송합니다（최상위 요약 + 스레드 세부사항）.

### 13. `verify`（`verify.ts`）

**기능**: 애플리케이션을 실행하여 코드 변경이 예상대로 작동하는지 검증합니다.

**특징**: `verifyContent.ts`에서 Skill 본문을 읽고（SKILL.md 파싱）, `files` 메커니즘으로 보조 파일을 임시 디렉터리에 씁니다. `USER_TYPE === 'ant'`에게만 사용 가능합니다.

### 14. `claudeInChrome`（`claudeInChrome.ts`）

**기능**: Side Panel 확장이 있는 실제 Chrome 브라우저에 연결된 headless 세션을 시작하여 Claude가 브라우저를 실시간으로 제어할 수 있습니다.

### 15. `claudeCodeGuide`（`AgentTool` 시스템에 내장）

Claude Code 내부 온보딩 흐름에 사용됩니다.

---

## 6.7 Skill과 Command의 관계

### 둘의 경계

Claude Code 설계에서 Skill과 Command는 한때 다른 개념이었지만 이제 통합되었습니다:

- **역사적으로**: `/commands/` 디렉터리에는 간단한 prompt 명령（`.md` 파일）이, `/skills/` 디렉터리에는 더 복잡하고 디렉터리 구조를 가진 워크플로우（`skill-name/SKILL.md`）가 저장됨
- **현재**: 두 가지 모두 `loadSkillsDir.ts`에 의해 로딩되어 통합 `Command` 타입으로 변환되며, `/commands/`는 `loadedFrom: 'commands_DEPRECATED'`로 표시됨（`loadSkillsDir.ts:608`）

현재 실제 차이는 로딩 경로에만 있습니다:
- `/skills/skill-name/SKILL.md`: 새 형식, 권장 사용, `baseDir` 지원（Skill이 보조 파일을 가질 수 있음）
- `/commands/skill-name.md` 또는 `/commands/skill-name/SKILL.md`: 구 형식, 하위 호환성

### 언제 Skill을 쓰고 언제 Command를 쓰는가

| 시나리오 | 권장 방법 |
|------|---------|
| 다중 파일 워크플로우（Skill에 보조 리소스 파일 첨부） | `/skills/` 디렉터리 형식 |
| 간단한 prompt 재사용（단일 md 파일로 충분） | 여전히 `/commands/` 사용 가능（호환） |
| `${CLAUDE_SKILL_DIR}` 변수가 필요한 경우 | 반드시 `/skills/` 디렉터리 형식 사용 |
| `files:` 내장 리소스가 필요한 경우（bundled skill） | `BundledSkillDefinition.files` |
| CLI 바이너리에 내장 | `registerBundledSkill()` |

---

## 6.8 설계 결정 분석

### 왜 JSON/YAML 대신 Markdown을 선택했는가

Skill의 실행 지시（본문）는 자연어로 작성되어야 AI가 이해하고 따를 수 있습니다. JSON/YAML은 구조화된 데이터만 인코딩할 수 있어 "먼저 관련 파일을 검색하고, 의존성 관계를 분석하되, 테스트 파일은 수정하지 않는다"와 같은 복잡한 지시를 직접 작성할 수 없습니다.

Markdown은 두 가지 모두를 충족합니다: frontmatter（YAML）는 구조화된 메타데이터를 담당하고, 본문（Markdown）은 인간이 읽을 수 있는 실행 지시를 담당합니다. 이것은 실용주의적인 형식 선택입니다.

### Skill의 권한 제어

권한 제어는 "화이트리스트 + 질문" 메커니즘을 채택합니다（`SkillTool.ts:871-900`）:

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // CommandBase properties...
  'name', 'description', 'hasUserSpecifiedDescription', ...
  // NOT included: 'allowedTools', 'hooks', 'paths', etc.
])
```

`skillHasOnlySafeProperties()`는 Skill이 "안전 속성"만 사용하는지 확인합니다—Skill이 `allowedTools`, `hooks`, `paths` 등의 민감한 속성을 선언하지 않으면 사용자 확인 없이 자동으로 실행이 허용됩니다. 이것은 좋은 보안 설계입니다: 새로 추가된 속성은 기본적으로 안전하지 않으며, 명시적인 검토 후에만 화이트리스트에 추가될 수 있습니다.

### 안전한 파일 쓰기 메커니즘

내장 Skill은 `files` 필드를 통해 보조 파일을 임베드하고, 디스크에 쓸 때 엄격한 보안 조치를 사용합니다（`bundledSkills.ts:171-194`）:

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

`O_NOFOLLOW | O_EXCL`를 사용하여 symlink 공격을 방지하고, 파일 권한은 `0o600`（소유자만 읽기/쓰기）입니다. 쓰기 디렉터리에는 매 프로세스 시작 시 무작위 nonce가 포함되어 경로 예측 공격을 방지합니다.

### MCP Skill의 통합 전략

MCP Skill은 `mcpSkillBuilders.ts`를 통해 우아한 의존성 역전을 구현합니다（`mcpSkillBuilders.ts:1-43`）:

MCP 발견 로직（`mcpSkills.ts`）은 `createSkillCommand`와 `parseSkillFrontmatterFields`를 사용해야 하지만, 직접 import하면 순환 의존성이 발생합니다. 해결책:

1. `loadSkillsDir.ts`가 모듈 초기화 시 `registerMCPSkillBuilders()`를 호출하여 두 함수를 등록
2. `mcpSkills.ts`가 필요할 때 `getMCPSkillBuilders()`를 통해 가져옴

이 설계는 Bun 번들링의 기술적 제한도 해결합니다: Bun bundle에서 변수를 사용하는 동적 import（리터럴이 아닌）는 해결할 수 없어 `await import(variable)` 방식을 사용할 수 없고, 이 레지스트리 패턴만 사용할 수 있습니다.

---

## 6.9 이전 가능한 패턴

### Doramagic Skill 시스템 비교

| 차원 | Claude Code Skill | Doramagic Skill |
|------|------------------|-----------------|
| 파일 형식 | `SKILL.md`（Markdown + YAML frontmatter） | `SKILL.md`（동일한 형식）|
| 디렉터리 구조 | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| 실행 엔진 | SkillTool（AI 도구 호출） | OpenClaw 도구 호출 |
| 소스 우선순위 | policy < user < project < plugin | OpenClaw 플랫폼 규칙 |
| 내장 Skill | 15+ 개, 바이너리에 컴파일 | 구축 중 |
| 파라미터 치환 | `$arg_name`, frontmatter `arguments` | 동일한 메커니즘 |
| 실행 컨텍스트 | inline / fork（자식 agent） | inline（현재 단계） |
| 조건부 활성화 | `paths` frontmatter | 미구현 |
| 동적 발견 | 파일 작업이 자동 발견 트리거 | 미구현 |

### 참고할 수 있는 핵심 패턴

**1. `skillify` 패턴: 워크플로우의 자기 번식**

Claude Code의 `skillify` Skill은 매우 우아한 설계입니다—AI가 방금 수행한 작업을 분석하고, 대화로 사용자를 안내하여 재사용 가능한 Skill로 고정화합니다. Doramagic도 `/dora-skillify`를 구현하여 성공적인 지식 추출 과정을 프로젝트 특정 Skill로 고정화할 수 있습니다.

**2. `when_to_use`의 AI 능동적 호출 메커니즘**

`when_to_use` frontmatter 필드를 통해 AI는 사용자가 명시적으로 슬래시 명령을 입력하지 않아도 언제 Skill을 능동적으로 호출해야 하는지 알 수 있습니다. Doramagic의 Skill도 이 필드를 중요시하여 지식 추출이 적절한 시기에 자동으로 트리거될 수 있도록 해야 합니다.

**3. 동적 Skill 발견과 조건부 활성화**

파일 경로에 따라 Skill을 활성화하는 메커니즘은 Doramagic의 프로젝트 특정 지식 시나리오에 매우 적합합니다: 사용자가 특정 도메인의 파일을 조작할 때 해당 도메인의 추출 Skill을 자동으로 활성화합니다（예: TypeScript 파일 조작 시 프런트엔드 아키텍처 분석 Skill 활성화）.

**4. `files` 메커니즘을 통한 보조 리소스 관리**

내장 Skill은 `files` 필드를 통해 참조 문서와 예제 코드를 Skill 패키지에 내장하고, 모델이 필요에 따라 읽게 하여 한 번에 컨텍스트에 모두 주입하지 않습니다. Doramagic의 대형 Skill（Soul Extractor 등）은 이 패턴으로 추출 템플릿과 참조 자료를 관리할 수 있습니다.

**5. 보안 모델: allowedTools 화이트리스트 + 안전 Skill 자동 허용**

Skill은 frontmatter에 선언된 도구만 사용할 수 있습니다. Claude Code는 추가로 "안전 Skill"（특별한 권한 없음）과 "확인이 필요한 Skill"（allowedTools/hooks 있음）을 구분하여, 전자를 자동으로 허용함으로써 마찰을 줄입니다. 이 권한 모델은 OpenClaw 플랫폼이 참고할 만합니다.

---

## 6.10 소스 코드 색인

| 파일 | 줄수 | 역할 |
|------|------|------|
| `skills/loadSkillsDir.ts` | 1,087 | Skill 로딩 핵심: 발견, 파싱, 중복 제거, 조건부 활성화, 동적 발견 |
| `skills/bundledSkills.ts` | 220 | 내장 Skill 레지스트리, 파일 추출, 안전한 쓰기 |
| `tools/SkillTool/SkillTool.ts` | 1,108 | Skill 실행 도구: 검증, 권한, inline/fork 실행 |
| `skills/mcpSkillBuilders.ts` | 44 | MCP Skill 빌더 레지스트리（순환 의존성 해결） |
| `skills/bundled/updateConfig.ts` | 475 | update-config: settings.json 구성 도우미 |
| `skills/bundled/scheduleRemoteAgents.ts` | 447 | schedule: 스케줄링된 원격 agent 관리 |
| `skills/bundled/keybindings.ts` | 339 | keybindings-help: 키보드 단축키 구성 |
| `skills/bundled/loremIpsum.ts` | 282 | lorem-ipsum: 정확한 토큰 계산 자리 표시자 텍스트 |
| `skills/bundled/skillify.ts` | 197 | skillify: 세션 워크플로우를 Skill로 자동 고정화 |
| `skills/bundled/claudeApi.ts` | 196 | claude-api: Claude API 개발 도우미（다국어） |
| `skills/bundled/claudeApiContent.ts` | 220 | claude-api의 247KB 문서 내용（빌드 시 인라인） |
| `skills/bundled/batch.ts` | 124 | batch: 대규모 병렬 worktree 변경 |
| `skills/bundled/loop.ts` | 92 | loop: 간격을 두고 prompt 반복 실행 |
| `skills/bundled/remember.ts` | 82 | remember: memory 검토 및 승격 |
| `skills/bundled/simplify.ts` | 69 | simplify: 3차원 코드 리뷰 및 수정 |
| `skills/bundled/debug.ts` | 약 60 | debug: 세션 디버그 로그 진단 |
| `skills/bundled/stuck.ts` | 약 60 | stuck: 프로세스 동결 진단 + Slack 보고 |
| `skills/bundled/verify.ts` | 약 30 | verify: 애플리케이션 실행으로 코드 변경 검증 |
| `skills/bundled/claudeInChrome.ts` | 약 40 | claude-in-chrome: Chrome 브라우저 제어 |
| `skills/bundled/index.ts` | - | 모든 내장 Skill의 등록 진입점 |

**핵심 함수 색인:**

| 함수 | 파일:줄번호 | 설명 |
|------|----------|------|
| `getSkillDirCommands` | `loadSkillsDir.ts:638` | 주 로딩 진입점（memoized） |
| `parseSkillFrontmatterFields` | `loadSkillsDir.ts:184` | frontmatter 필드 파싱 |
| `createSkillCommand` | `loadSkillsDir.ts:269` | Command 객체 빌드 |
| `loadSkillsFromSkillsDir` | `loadSkillsDir.ts:407` | `/skills/` 디렉터리에서 로딩 |
| `discoverSkillDirsForPaths` | `loadSkillsDir.ts:861` | 동적 Skill 디렉터리 발견 |
| `activateConditionalSkillsForPaths` | `loadSkillsDir.ts:997` | 조건부 Skill 활성화 |
| `registerBundledSkill` | `bundledSkills.ts:55` | 내장 Skill 등록 |
| `executeForkedSkill` | `SkillTool.ts:121` | Fork 모드 실행 |
| `skillHasOnlySafeProperties` | `SkillTool.ts:871+` | 안전 Skill 판단 |
| `registerMCPSkillBuilders` | `mcpSkillBuilders.ts:31` | MCP 빌더 등록 |
