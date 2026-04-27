# 第 6 章：Skill 系统

## 6.1 概述与定位

Skill 系统是 Claude Code 中最具创新性的架构之一。它将可复用的工作流（workflow）编码为 Markdown 文件，并通过斜杠命令（`/skill-name`）或 AI 主动调用的方式触发。本质上，Skill 是"给 AI 的 SOP"——把人类专家执行复杂任务的步骤、判断条件、成功标准，以结构化的 Markdown 格式沉淀下来，赋予 AI 可复现的专业执行能力。

与普通的 Prompt 不同，Skill 系统具备以下核心特征：

1. **声明式 + 执行式融合**：Frontmatter 声明元数据（权限、模型、触发条件），正文是执行指令
2. **多来源加载**：内置（bundled）、用户级、项目级、Plugin 级、MCP 来源，按优先级合并
3. **两种执行模式**：inline（注入当前会话上下文）和 fork（在独立 sub-agent 中隔离执行）
4. **条件激活**：通过 `paths` frontmatter 实现按文件路径自动激活的 Skill
5. **动态发现**：在会话过程中，随着用户操作文件，自动发现并加载更深层目录的 Skill

Skill 系统不是简单的命令别名，而是一个完整的工作流编排框架。

---

## 6.2 理论基础

### 可复用工作流（Reusable Workflows）的设计模式

Skill 系统解决了 AI 工具使用中一个核心问题：**专业知识如何沉淀并可复现？** 传统的代码复用通过函数和类，但 AI 执行的"知识"是自然语言描述的工作流，无法直接用代码函数封装。

Skill 的设计借鉴了 SOP（Standard Operating Procedure）的思想——把专家的执行流程、决策点、成功标准结构化记录，使 AI 每次执行时都遵循相同的高质量路径。

### 声明式 vs 命令式工作流定义

Skill 系统同时支持两种风格：

- **声明式**：通过 frontmatter 声明 `allowed-tools`、`model`、`context` 等属性，让系统自动处理权限控制和执行上下文配置
- **命令式**：Skill 正文中可以嵌入 shell 命令（`!``command``）直接执行，实现"说明中夹杂操作"

### Markdown-as-Code 的理念

选择 Markdown 而非 JSON/YAML 作为 Skill 格式，是一个深思熟虑的设计决策：

- **人类可读性**：开发者可以直接阅读和编辑 Skill，理解其意图
- **AI 天然友好**：AI 训练数据大量包含 Markdown，AI 对 Markdown 的理解比 JSON 更自然
- **渐进结构化**：可以从纯散文开始，逐步添加标题、步骤、规则，不强制完整结构
- **版本控制友好**：Markdown diff 对人类友好，代码审查时一眼看出工作流变化

---

## 6.3 Skill 格式与数据结构

### Skill Markdown 文件的格式规范

Skill 文件遵循固定的目录结构：

```
.claude/skills/<skill-name>/SKILL.md
```

文件格式为 frontmatter + Markdown 正文：

```markdown
---
name: my-skill
description: 一句话描述这个 Skill 做什么
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  当用户想要... 时使用。例如："cherry-pick to release"、"hotfix"。
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## 步骤

### 1. 第一步
具体操作...

**成功标准**: 能证明这步完成的检查点
```

### frontmatter 字段详解

以下是 `parseSkillFrontmatterFields` 函数（`loadSkillsDir.ts:184`）解析的全部字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 显示名称（可与目录名不同） |
| `description` | string | 一句话描述，显示在 `/help` 中 |
| `allowed-tools` | string[] | 白名单工具列表，支持 `Bash(git:*)` 前缀模式 |
| `argument-hint` | string | 用户触发时的参数提示，如 `"<branch-name>"` |
| `arguments` | string[] | 参数名列表，用于 `$arg_name` 变量替换 |
| `when_to_use` | string | 告诉 AI 何时主动调用此 Skill，含触发短语 |
| `version` | string | Skill 版本号 |
| `model` | string | 模型覆盖，如 `opus`、`sonnet`，`inherit` 表示继承 |
| `disable-model-invocation` | boolean | 禁止 AI 主动调用，只能用户手动触发 |
| `user-invocable` | boolean | 是否在 `/help` 中可见（默认 `true`） |
| `context` | `"fork"` | 设置后在独立 sub-agent 中执行 |
| `agent` | string | 指定 agent 类型 |
| `effort` | EffortValue | 影响模型思考深度 |
| `paths` | string[] | gitignore 语法的路径模式，用于条件激活 |
| `hooks` | HooksSettings | Skill 执行期间的 hook 配置 |
| `shell` | FrontmatterShell | 内联 shell 命令执行配置 |

### SkillDefinition 类型

`bundledSkills.ts` 中定义了 `BundledSkillDefinition`（第 12-41 行），而文件系统 Skill 对应的是 `Command` 类型（`src/types/command.js`）。两者在 `createSkillCommand`（`loadSkillsDir.ts:269`）中汇聚为统一的 `Command` 对象：

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

## 6.4 Skill 加载机制

### loadSkillsDir 的完整加载流程

`getSkillDirCommands`（`loadSkillsDir.ts:638`）是整个加载流程的入口，使用 `lodash-es/memoize` 缓存结果，避免重复 I/O：

```
启动时
  ├── policySettings: ~/.claude-managed/.claude/skills/（企业管控）
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/（从 cwd 向上到 home）
  ├── additionalDirs: --add-dir 指定的额外目录
  └── legacyCommands: .claude/commands/（向后兼容）

会话中（动态发现）
  └── 用户读写文件时 → discoverSkillDirsForPaths() → addSkillDirectories()
```

加载结果通过 `realpath` 去重（`loadSkillsDir.ts:728-763`），避免 symlink 导致的重复加载。

### 多来源加载优先级

代码注释明确说明加载优先级（`loadSkillsDir.ts:677-714`）：

```
managed（企业策略） < user（用户级） < project（项目级） < additional（--add-dir）
```

这是"越具体优先级越高"的原则：项目级覆盖用户级，因为项目有特定需求。

**特殊情况：**
- `--bare` 模式：跳过自动发现，只加载 `--add-dir` 显式指定的目录
- `skillsLocked`（plugin-only policy）：禁止加载用户/项目级 Skill，只允许 Plugin 来源
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS` 环境变量：跳过 managed 级 Skill

### Skill 发现与匹配逻辑

**静态发现**（启动时）：`getSkillDirCommands` 扫描各级 `~/.claude/skills/` 目录，只支持目录格式（`skill-name/SKILL.md`），不支持单个 `.md` 文件。

**动态发现**（会话中）：当用户读写文件时，`discoverSkillDirsForPaths`（`loadSkillsDir.ts:861`）沿文件路径向上游走，在每个目录检查是否存在 `.claude/skills/`，发现后通过 `addSkillDirectories` 加载。被 `.gitignore` 标记的目录会被跳过（防止 `node_modules` 中的 Skill 污染）。

**条件激活**（paths frontmatter）：带有 `paths` 字段的 Skill 初始不对模型可见，存入 `conditionalSkills` Map。当用户操作匹配路径的文件时，`activateConditionalSkillsForPaths`（`loadSkillsDir.ts:997`）使用 `ignore` 库（gitignore 语法）匹配，命中则移入 `dynamicSkills` 激活。

---

## 6.5 SkillTool 执行流程

### 从 /skill-name 到执行的完整路径

`SkillTool`（`tools/SkillTool/SkillTool.ts:330`）是一个标准 `Tool` 实现，AI 通过调用这个工具来执行 Skill。完整执行路径如下：

```
用户键入 /skill-name 或 AI 决定调用 SkillTool
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── 去掉前导斜杠（兼容处理）
  │     ├── 检查 _canonical_ 前缀（远程 Skill，实验性）
  │     ├── findCommand() 查找注册的 Command
  │     ├── 检查 disableModelInvocation 标志
  │     └── 确认 type === 'prompt'
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── 检查 deny 规则
  │     ├── 检查远程 canonical Skill（自动允许）
  │     ├── 检查 allow 规则
  │     ├── skillHasOnlySafeProperties() → 自动允许安全 Skill
  │     └── 默认：弹窗询问用户（behavior: 'ask'）
  │
  └── call (SkillTool.ts:580)
        ├── 检查 context === 'fork' → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent()（独立 sub-agent）
        └── 否则（inline）→ processPromptSlashCommand()
              └── 注入 newMessages + contextModifier 到当前会话
```

### Skill 上下文注入

Inline 执行时，`call` 返回 `newMessages` 和 `contextModifier`（`SkillTool.ts:767-840`）：

- **newMessages**：Skill 展开后的消息列表，注入当前对话上下文
- **contextModifier**：修改 `ToolUseContext` 的函数，用于：
  - 叠加 `allowedTools`（Skill 声明的工具权限）
  - 覆盖 `mainLoopModel`（如果 Skill 指定了 model）
  - 覆盖 `effortValue`（如果 Skill 指定了 effort）

值得注意的是，`contextModifier` 采用链式调用模式（`SkillTool.ts:777`），正确处理多个 contextModifier 叠加的情况，而不是简单覆盖。

### Skill 变量替换

`createSkillCommand` 中的 `getPromptForCommand`（`loadSkillsDir.ts:343-398`）在返回 Skill 内容前执行以下替换：

1. **参数替换**：`$arg_name` → `substituteArguments()` 注入用户传入的参数
2. **目录变量**：`${CLAUDE_SKILL_DIR}` → Skill 文件所在目录的绝对路径
3. **Session ID**：`${CLAUDE_SESSION_ID}` → 当前会话 ID
4. **Shell 命令执行**：`!``command`` ` → 执行结果内联（仅限非 MCP Skill）

MCP Skill 禁用了 Shell 命令执行（`loadSkillsDir.ts:372`），防止远端不受信任 Skill 注入任意 shell 命令。

### Skill 与工具的交互

在 Forked 执行模式下（`executeForkedSkill`，`SkillTool.ts:121`），Skill 在完全隔离的 sub-agent 中运行：

- 通过 `runAgent()` 启动独立 agent，有独立的 token budget
- 执行过程中的 tool use 消息通过 `onProgress` 回调上报，UI 可以展示进度
- 执行结果通过 `extractResultText` 提取最终文本，返回给父 agent
- 通过 `clearInvokedSkillsForAgent` 释放内存（`SkillTool.ts:286`）

---

## 6.6 Bundled Skills 完整清单与分析

内置 Skill 通过 `registerBundledSkill()`（`bundledSkills.ts:55`）注册，在 CLI 启动时初始化。以下是全部 17 个内置 Skill 的分析：

### 1. `update-config`（`updateConfig.ts`，475 行）

**功能**：配置 Claude Code 的 `settings.json`，包括 Permissions、Hooks、Model、MCP 等所有配置项。

**特点**：Skill 正文动态生成——使用 `toJSONSchema(SettingsSchema())` 从 Zod schema 自动生成 JSON Schema 文档，确保文档永远与实际类型同步。包含完整的 Hooks 文档（所有 Hook 事件、Hook 类型、JSON 输出格式）。

**触发场景**：用户想配置行为自动化、权限规则、环境变量、模型设置时。

### 2. `schedule`（`scheduleRemoteAgents.ts`，447 行）

**功能**：管理远程定时 Agent（cron 触发器），创建、更新、列出、运行定时任务。

**特点**：调用前先检查多个前提条件（OAuth tokens、仓库信息、MCP connectors、cloud environments），并将这些动态信息注入到 Skill 提示词中。通过 `AskUserQuestion` 工具与用户交互。

**触发场景**：用户想创建定时运行的 Claude Code agent（如每日代码审查、自动报告）。

### 3. `keybindings-help`（`keybindings.ts`，339 行）

**功能**：帮助用户自定义键盘快捷键，修改 `~/.claude/keybindings.json`。

**特点**：通过 `generateContextsTable()`、`generateActionsTable()` 从代码常量动态生成文档，并通过 `generateReservedShortcuts()` 列出不可重绑定的快捷键，防止用户误操作。

**触发场景**：用户想重新绑定快捷键、添加组合键、修改提交键。

### 4. `lorem-ipsum`（`loremIpsum.ts`，282 行）

**功能**：生成固定数量的单 token 词汇占位文本，用于 token 计数和性能测试。

**特点**：使用经过 API 验证的单 token 词汇列表，确保 `lorem` 参数可精确控制 token 数量。常用于基准测试和 token 计费分析。

**触发场景**：需要精确 token 数量的测试文本。

### 5. `skillify`（`skillify.ts`，197 行）

**功能**：将当前会话的操作过程自动转化为可复用的 SKILL.md 文件。

**特点**：这是 Skill 系统的"自我繁殖"机制。通过读取 session memory 和用户消息历史，引导用户用 4 轮 `AskUserQuestion` 对话确认工作流名称、步骤、参数、触发条件，最终生成标准格式的 SKILL.md 并写入磁盘。

**限制**：仅对 `USER_TYPE === 'ant'` 可用（Anthropic 内部员工）。

**触发场景**：会话结束时，用户想把刚才完成的操作流程固化为可复用 Skill。

### 6. `claude-api`（`claudeApi.ts`，196 行 + `claudeApiContent.ts`，220 行）

**功能**：帮助开发者使用 Claude API 或 Anthropic SDK 构建应用。

**特点**：
- 自动检测当前项目语言（通过扫描文件后缀，支持 Python/TypeScript/Java/Go/Ruby/C#/PHP/curl）
- 延迟加载（247KB 的 `.md` 内容在调用时才加载），避免影响启动时间
- 包含语言特定的 API 文档、Agent SDK patterns、streaming 等
- 通过 `files` 机制将文档写入临时目录，模型可用 Read/Grep 工具按需读取

**触发场景**：代码中 import `anthropic` 或用户询问如何使用 Claude API。

### 7. `batch`（`batch.ts`，124 行）

**功能**：将大规模代码变更（如迁移、重构、批量重命名）分解为 5-30 个并行 worktree agent 执行。

**特点**：三阶段执行模型——Plan（进入 Plan Mode 深度研究分解）→ Spawn Workers（并行启动带 `isolation: "worktree"` 的 background agent）→ Track Progress（实时渲染状态表格）。每个 worker 都在独立 git worktree 中工作，互不影响，完成后开 PR。

**触发场景**：大规模代码迁移、全库重构、批量修改。

### 8. `loop`（`loop.ts`，92 行）

**功能**：以固定间隔重复执行一个 prompt 或斜杠命令。

**特点**：智能解析时间间隔（支持 `5m`、`2h` 前缀格式和 `every 20m` 后缀格式），将其转换为 cron 表达式，调用 `ScheduleCronTool` 注册定时任务。设置后立即执行一次，不等待第一次定时触发。

**触发场景**：用户想定期检查部署状态、周期性运行某个 Skill。

### 9. `remember`（`remember.ts`，82 行）

**功能**：审查 auto-memory 条目，提议将其提升到 `CLAUDE.md`、`CLAUDE.local.md` 或团队 memory 中。

**特点**：采用"先提议后确认"原则，不直接修改文件，而是展示分类报告（待提升/待清理/存疑/无需操作），等用户审批后再执行。区分项目级约定（CLAUDE.md）、个人偏好（CLAUDE.local.md）和组织级知识（团队 memory）。

**限制**：仅对 `USER_TYPE === 'ant'` 且 auto-memory 功能启用时可用。

**触发场景**：用户想整理 memory，避免 auto-memory 无限积累。

### 10. `simplify`（`simplify.ts`，69 行）

**功能**：对当前 git diff 进行三维度代码审查（代码复用、代码质量、效率），并直接修复发现的问题。

**特点**：同时启动三个并行 sub-agent，分别负责：
- **代码复用 Agent**：发现重复造轮子，指向现有工具函数
- **代码质量 Agent**：发现冗余状态、参数膨胀、拷贝粘贴、泄漏抽象等
- **效率 Agent**：发现不必要计算、缺失并发、N+1 模式、内存泄漏等

三 agent 完成后合并发现并直接修复，不只是报告。

**触发场景**：完成一段代码后的质量复查，也被 `batch` Skill 的 worker 流程自动调用。

### 11. `debug`（`debug.ts`）

**功能**：诊断当前 Claude Code 会话的调试日志，帮助排查问题。

**特点**：通过 tail 读取（最多 64KB）调试日志的最后几行，避免长会话中日志文件过大导致内存峰值。对非 Anthropic 员工会先启用 debug logging 再读取。标记为 `disableModelInvocation: true`，防止 AI 自动调用（只能用户手动触发）。

### 12. `stuck`（`stuck.ts`）

**功能**：诊断机器上其他冻结或卡顿的 Claude Code 进程，并将报告发到 Slack 频道。

**特点**：Anthropic 内部诊断工具。检测高 CPU（≥90% 持续）、D 状态（I/O 挂起）、T 状态（Ctrl+Z 停止）、Z 状态（僵尸进程）、高内存（≥4GB）等异常。使用两消息结构发送 Slack 报告（顶层摘要 + 线程详情）。

### 13. `verify`（`verify.ts`）

**功能**：通过运行应用程序验证代码变更是否符合预期。

**特点**：从 `verifyContent.ts` 读取 Skill 正文（SKILL.md 解析），通过 `files` 机制将辅助文件写入临时目录。仅对 `USER_TYPE === 'ant'` 可用。

### 14. `claudeInChrome`（`claudeInChrome.ts`）

**功能**：启动一个与真实 Chrome 浏览器连接的 headless 会话，带有 Side Panel 扩展，Claude 可实时控制浏览器。

### 15. `claudeCodeGuide`（内嵌于 `AgentTool` 系统）

用于 Claude Code 内部引导流程。

---

## 6.7 Skill 与 Command 的关系

### 两者的边界

在 Claude Code 的设计中，Skill 和 Command 曾经是不同概念，但现在已经统一：

- **历史上**：`/commands/` 目录存放简单的 prompt 命令（`.md` 文件），`/skills/` 目录存放更复杂的、有目录结构的工作流（`skill-name/SKILL.md`）
- **现在**：两者都被 `loadSkillsDir.ts` 加载，统一转换为 `Command` 类型，`/commands/` 被标记为 `loadedFrom: 'commands_DEPRECATED'`（`loadSkillsDir.ts:608`）

目前的实际差异仅在加载路径：
- `/skills/skill-name/SKILL.md`：新格式，推荐使用，支持 `baseDir`（Skill 可以携带辅助文件）
- `/commands/skill-name.md` 或 `/commands/skill-name/SKILL.md`：旧格式，向后兼容

### 什么时候用 Skill，什么时候用 Command

| 场景 | 推荐方式 |
|------|---------|
| 多文件工作流（Skill 附带辅助资源文件） | `/skills/` 目录格式 |
| 简单的 prompt 复用（单个 md 文件即可） | 仍可用 `/commands/`（兼容） |
| 需要 `${CLAUDE_SKILL_DIR}` 变量 | 必须用 `/skills/` 目录格式 |
| 需要 `files:` 嵌入资源（bundled skill） | `BundledSkillDefinition.files` |
| 内置到 CLI 二进制 | `registerBundledSkill()` |

---

## 6.8 设计决策分析

### 为什么选择 Markdown 而非 JSON/YAML

Skill 的执行指令（正文）用自然语言写，AI 才能理解和遵循。JSON/YAML 只能编码结构化数据，无法直接写"先搜索相关文件，再分析依赖关系，注意不要修改测试文件"这类复杂指令。

Markdown 兼顾了两者：frontmatter（YAML）负责结构化的元数据，正文（Markdown）负责人类可读的执行指令。这是一种实用主义的格式选择。

### Skill 的权限控制

权限控制采用"白名单 + 询问"机制（`SkillTool.ts:871-900`）：

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

`skillHasOnlySafeProperties()` 检查一个 Skill 是否只用了"安全属性"——如果 Skill 没有声明 `allowedTools`、`hooks`、`paths` 等敏感属性，则自动允许执行，无需用户确认。这是良好的安全设计：新增的属性默认不安全，需要明确审查后才能加入白名单。

### 安全文件写入机制

内置 Skill 通过 `files` 字段嵌入辅助文件，写入磁盘时使用严格的安全措施（`bundledSkills.ts:171-194`）：

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

使用 `O_NOFOLLOW | O_EXCL` 防止 symlink 攻击，文件权限为 `0o600`（仅所有者读写）。写入目录包含每次进程启动的随机 nonce，防止路径预测攻击。

### MCP Skill 的集成策略

MCP Skill 通过 `mcpSkillBuilders.ts` 实现了一个优雅的依赖倒置（`mcpSkillBuilders.ts:1-43`）：

MCP 发现逻辑（`mcpSkills.ts`）需要使用 `createSkillCommand` 和 `parseSkillFrontmatterFields`，但直接 import 会造成循环依赖。解决方案是：

1. `loadSkillsDir.ts` 在模块初始化时调用 `registerMCPSkillBuilders()` 注册这两个函数
2. `mcpSkills.ts` 在需要时通过 `getMCPSkillBuilders()` 取用

这个设计还解决了 Bun 打包的技术限制：Bun bundle 中动态 import 使用变量（非字面量）无法解析，所以不能用 `await import(variable)` 方式，只能用这种注册表模式。

---

## 6.9 可迁移模式

### Doramagic Skill 系统对比

| 维度 | Claude Code Skill | Doramagic Skill |
|------|------------------|-----------------|
| 文件格式 | `SKILL.md`（Markdown + YAML frontmatter） | `SKILL.md`（相同格式）|
| 目录结构 | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| 执行引擎 | SkillTool（AI 工具调用） | OpenClaw 工具调用 |
| 来源优先级 | policy < user < project < plugin | OpenClaw 平台规则 |
| 内置 Skill | 15+ 个，编译到二进制 | 正在构建 |
| 参数替换 | `$arg_name`，frontmatter `arguments` | 相同机制 |
| 执行上下文 | inline / fork（子 agent） | inline（现阶段） |
| 条件激活 | `paths` frontmatter | 暂未实现 |
| 动态发现 | 文件操作触发自动发现 | 暂未实现 |

### 可借鉴的核心模式

**1. `skillify` 模式：工作流的自我繁殖**

Claude Code 的 `skillify` Skill 是一个极其优雅的设计——让 AI 分析自己刚才执行的操作，通过对话引导用户把它固化为可复用 Skill。Doramagic 同样可以实现一个 `/dora-skillify`，把一次成功的知识提取过程固化为项目特定的 Skill。

**2. `when_to_use` 的 AI 主动调用机制**

`when_to_use` frontmatter 字段让 AI 知道何时主动调用 Skill，不需要用户明确输入斜杠命令。Doramagic 的 Skill 也应该重视这个字段，让知识提取可以在合适时机自动触发。

**3. 动态技能发现与条件激活**

按文件路径激活 Skill 的机制非常适合 Doramagic 的项目特定知识场景：当用户操作某个领域的文件时，自动激活对应领域的提取 Skill（如操作 TypeScript 文件时激活前端架构分析 Skill）。

**4. `files` 机制的辅助资源管理**

内置 Skill 通过 `files` 字段把参考文档、示例代码嵌入 Skill 包，模型按需读取而非一次性注入上下文。Doramagic 的大型 Skill（如 Soul Extractor）可以采用此模式管理提取模板和参考资料。

**5. 安全模型：allowedTools 白名单 + 自动允许安全 Skill**

Skill 只能使用 frontmatter 中声明的工具。Claude Code 进一步区分"安全 Skill"（无特殊权限）和"需确认 Skill"（有 allowedTools/hooks），自动允许前者降低摩擦。这个权限模型值得 OpenClaw 平台借鉴。

---

## 6.10 源码索引

| 文件 | 行数 | 作用 |
|------|------|------|
| `skills/loadSkillsDir.ts` | 1,087 | Skill 加载核心：发现、解析、去重、条件激活、动态发现 |
| `skills/bundledSkills.ts` | 220 | 内置 Skill 注册表、文件提取、安全写入 |
| `tools/SkillTool/SkillTool.ts` | 1,108 | Skill 执行工具：验证、权限、inline/fork 执行 |
| `skills/mcpSkillBuilders.ts` | 44 | MCP Skill 构建器注册表（解循环依赖） |
| `skills/bundled/updateConfig.ts` | 475 | update-config：settings.json 配置帮手 |
| `skills/bundled/scheduleRemoteAgents.ts` | 447 | schedule：定时远程 agent 管理 |
| `skills/bundled/keybindings.ts` | 339 | keybindings-help：键盘快捷键配置 |
| `skills/bundled/loremIpsum.ts` | 282 | lorem-ipsum：精确 token 计数的占位文本 |
| `skills/bundled/skillify.ts` | 197 | skillify：会话工作流自动固化为 Skill |
| `skills/bundled/claudeApi.ts` | 196 | claude-api：Claude API 开发帮手（多语言） |
| `skills/bundled/claudeApiContent.ts` | 220 | claude-api 的 247KB 文档内容（构建时内联） |
| `skills/bundled/batch.ts` | 124 | batch：大规模并行 worktree 变更 |
| `skills/bundled/loop.ts` | 92 | loop：按间隔重复执行 prompt |
| `skills/bundled/remember.ts` | 82 | remember：memory 审查与提升 |
| `skills/bundled/simplify.ts` | 69 | simplify：三维度代码审查并修复 |
| `skills/bundled/debug.ts` | 约 60 | debug：会话调试日志诊断 |
| `skills/bundled/stuck.ts` | 约 60 | stuck：进程冻结诊断 + Slack 报告 |
| `skills/bundled/verify.ts` | 约 30 | verify：运行应用验证代码变更 |
| `skills/bundled/claudeInChrome.ts` | 约 40 | claude-in-chrome：Chrome 浏览器控制 |
| `skills/bundled/index.ts` | - | 所有内置 Skill 的注册入口 |

**关键函数索引：**

| 函数 | 文件:行号 | 说明 |
|------|----------|------|
| `getSkillDirCommands` | `loadSkillsDir.ts:638` | 主加载入口（memoized） |
| `parseSkillFrontmatterFields` | `loadSkillsDir.ts:184` | frontmatter 字段解析 |
| `createSkillCommand` | `loadSkillsDir.ts:269` | 构建 Command 对象 |
| `loadSkillsFromSkillsDir` | `loadSkillsDir.ts:407` | 从 `/skills/` 目录加载 |
| `discoverSkillDirsForPaths` | `loadSkillsDir.ts:861` | 动态发现 Skill 目录 |
| `activateConditionalSkillsForPaths` | `loadSkillsDir.ts:997` | 条件 Skill 激活 |
| `registerBundledSkill` | `bundledSkills.ts:55` | 注册内置 Skill |
| `executeForkedSkill` | `SkillTool.ts:121` | Fork 模式执行 |
| `skillHasOnlySafeProperties` | `SkillTool.ts:871+` | 安全 Skill 判断 |
| `registerMCPSkillBuilders` | `mcpSkillBuilders.ts:31` | MCP 构建器注册 |
