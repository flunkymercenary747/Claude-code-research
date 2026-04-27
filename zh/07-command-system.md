# 第 7 章：命令系统

## 7.1 概述与定位

Claude Code 的命令系统是用户与 REPL 交互的核心入口。每当用户在输入框键入 `/` 时，触发的就是这套系统。它承担着三个角色：

1. **UI 控制层**：直接操作终端界面状态，不经过 LLM（如 `/clear`、`/theme`、`/vim`）
2. **会话管理层**：管理对话历史、上下文压缩与恢复（如 `/compact`、`/resume`、`/branch`）
3. **能力扩展层**：将复杂任务委派给模型执行，通过 Prompt 展开机制（如 `/review`、`/skills`）

命令系统的边界设计体现了清晰的关注点分离：命令负责"触发"，工具（Tools）负责"执行"，LLM 负责"决策"。一个 `/review` 命令不会直接调用 git，而是把审查 prompt 注入对话流，让模型驱动后续的工具调用链。

---

## 7.2 理论基础

### 命令模式（Command Pattern）

系统的设计与经典 GoF 命令模式高度吻合：

- **Command 接口**：`Command` 联合类型（`PromptCommand | LocalCommand | LocalJSXCommand`），统一封装请求
- **ConcreteCommand**：每个 `commands/<name>/index.ts` 文件是一个具体命令实现
- **Invoker**：REPL 的 `processSlashCommand` 负责调度执行
- **Receiver**：`ToolUseContext`（对话状态）、`AppState`（应用状态）是被操作的对象

但 Claude Code 对经典模式做了两处关键扩展：

**惰性加载**：命令通过 `load(): Promise<Module>` 延迟加载实现，而非在注册时立即实例化。这将启动开销分摊到首次调用，对于包含重型依赖的命令（如 `/insights` 的 113KB HTML 渲染模块）意义重大。

**类型化返回值**：命令不是无返回值的动作（void），而是返回结构化结果（`LocalCommandResult`），由上层 REPL 决定如何渲染，实现了执行与展示的解耦。

### REPL 命令处理的设计模式

Claude Code 采用的 REPL 命令处理遵循两个核心原则：

**Immediate vs Queued**：命令对象上的 `immediate?: boolean` 字段决定命令是否绕过消息队列立即执行。`/clear`、`/exit` 等界面操作需要立即响应，而 `/compact` 这类涉及 API 调用的操作则进入队列有序处理。

**Auth-gated 可用性**：不同于运行时 feature flag（`isEnabled()`），`availability` 字段在命令列表过滤阶段就生效，确保未授权用户甚至看不到特定命令的存在（如仅限 claude.ai 订阅者的命令）。

---

## 7.3 命令注册机制

### commands.ts 的注册流程

命令注册的核心逻辑集中在 `commands.ts`（754 行），整体分为四个层次：

**第一层：静态内置命令**

```typescript
// commands.ts:240-310（核心片段）
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... 约 60 个内置命令
])
```

`COMMANDS` 函数包裹在 `memoize` 中而非模块级常量，原因是部分命令在注册时需要读取配置文件，而配置在模块初始化时尚不可用。

**第二层：Feature-flag 条件命令**

```typescript
// commands.ts:68-112（条件导入片段）
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

这些命令通过 `bun:bundle` 的 `feature()` 函数进行 dead code elimination，在构建时直接裁剪掉未启用的命令，而非运行时判断。

**第三层：内部专用命令**

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

这些命令仅在 `USER_TYPE === 'ant'`（Anthropic 内部用户）且非 demo 模式时注册，是内部工具与调试命令的隔离机制。

**第四层：动态加载命令**

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

Skills、插件命令和工作流命令三者并行异步加载，并按优先级排列：bundled skills 优先级最高，内置命令最低。这确保用户自定义命令可以覆盖（shadow）同名内置命令。

### 命令的类型定义

`types/command.ts` 定义了三种互斥的命令类型，构成 `Command` 联合类型：

```typescript
// types/command.ts（核心联合类型）
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| 类型 | 描述 | 典型命令 |
|------|------|----------|
| `PromptCommand` | 展开为 prompt 注入对话流，由模型执行 | `/review`, `/skills`, 所有 Skill |
| `LocalCommand` | 纯本地同步执行，返回文本结果 | `/compact`, `/context` |
| `LocalJSXCommand` | 渲染 Ink React UI 组件 | `/model`, `/resume`, `/config` |

`CommandBase` 是三者共享的基础字段集合，包含：

```typescript
// types/command.ts（CommandBase 核心字段）
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // 运行时 feature flag 检查
  isHidden?: boolean                      // 隐藏于 typeahead
  argumentHint?: string                   // 参数提示文本
  whenToUse?: string                      // 模型调用场景描述
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // 绕过队列立即执行
  isSensitive?: boolean                   // 参数从历史记录中脱敏
}
```

### 命令分类（内置 vs 插件 vs 用户自定义）

```
命令来源层次（优先级从高到低）
├── bundledSkills        # 随 Claude Code 打包的内置 Skills
├── builtinPluginSkills  # 已启用内置插件提供的 Skills
├── skillDirCommands     # 用户 .claude/skills/ 目录下的 Skills
├── workflowCommands     # 工作流脚本命令（feature: WORKFLOW_SCRIPTS）
├── pluginCommands       # 第三方插件注册的命令
├── pluginSkills         # 第三方插件提供的 Skills
└── COMMANDS()           # 硬编码内置命令（最低优先级）
```

---

## 7.4 命令分类完整清单

以下基于 `commands.ts` 的 `ls` 输出和注册列表整理。

### 会话管理类

| 命令 | 描述 |
|------|------|
| `/compact [instructions]` | 压缩对话历史，释放上下文窗口 |
| `/resume` | 从历史 session 列表中选择并恢复对话 |
| `/branch [title]` | 从当前对话分叉出新 session |
| `/rewind` | 回退到对话中的某个历史节点 |
| `/clear` | 清空当前对话记录 |
| `/session` | 显示当前 session 信息 |
| `/rename` | 重命名当前 session |
| `/summary` | 生成当前对话摘要（内部命令） |
| `/export` | 导出对话内容 |
| `/copy` | 复制最后一条消息到剪贴板 |

### 开发工具类

| 命令 | 描述 |
|------|------|
| `/review [PR#]` | 本地代码审查（调用 `gh pr diff`） |
| `/ultrareview [PR#]` | 云端深度代码审查（10-20分钟，bughunter 驱动） |
| `/commit` | 提交代码变更（内部命令） |
| `/commit-push-pr` | 提交 + Push + 创建 PR（内部命令） |
| `/diff` | 查看当前 git diff |
| `/init` | 初始化项目（生成 CLAUDE.md） |
| `/add-dir` | 添加额外工作目录 |
| `/hooks` | 管理事件钩子配置 |
| `/files` | 列出会话中追踪的文件 |
| `/pr_comments` | 查看 PR 评论 |
| `/issue` | 创建/查看 GitHub Issue（内部命令） |
| `/autofix-pr` | 自动修复 PR 中的问题（内部命令） |

### 配置类

| 命令 | 描述 |
|------|------|
| `/model [name]` | 切换对话模型（带交互式选择器） |
| `/config` | 查看/修改配置项 |
| `/theme` | 切换终端主题 |
| `/vim` | 切换 vim 输入模式 |
| `/keybindings` | 管理快捷键绑定 |
| `/permissions` | 查看/修改工具权限 |
| `/privacy-settings` | 管理隐私设置 |
| `/output-style` | 设置输出格式偏好 |
| `/effort` | 设置响应努力级别 |
| `/fast` | 切换快速模式 |
| `/plan` | 切换 Plan 模式（只规划不执行） |
| `/sandbox-toggle` | 切换沙箱模式 |

### 调试与诊断类

| 命令 | 描述 |
|------|------|
| `/doctor` | 诊断配置与环境问题 |
| `/cost` | 显示当前 session 的 token 消耗与费用 |
| `/context` | 显示上下文窗口使用详情（按类别分表） |
| `/stats` | 显示使用统计数据 |
| `/usage` | 显示 API 用量信息 |
| `/insights` | 生成历史 session 使用分析报告（惰性加载 113KB 模块） |
| `/heapdump` | 生成内存堆快照（调试用） |
| `/debug-tool-call` | 调试工具调用（内部命令） |
| `/perf-issue` | 记录性能问题（内部命令） |
| `/ant-trace` | Anthropic 内部追踪（内部命令） |

### 身份与服务类

| 命令 | 描述 |
|------|------|
| `/login` | 登录 Claude.ai 账户 |
| `/logout` | 退出登录 |
| `/upgrade` | 升级到更高套餐 |
| `/install-github-app` | 安装 GitHub App |
| `/install-slack-app` | 安装 Slack App |
| `/ide` | IDE 集成管理 |
| `/terminalSetup` | 终端集成配置 |
| `/mobile` | 显示移动端连接二维码 |
| `/chrome` | Chrome 扩展管理 |
| `/desktop` | 桌面应用管理 |

### 高级功能类

| 命令 | 描述 |
|------|------|
| `/mcp` | MCP 服务器管理（列出/启动/重启） |
| `/skills` | Skills 管理（列出/安装/更新） |
| `/tasks` | 后台任务管理 |
| `/agents` | 子代理管理 |
| `/memory` | 项目记忆文件管理（CLAUDE.md） |
| `/plan` | 进入规划模式 |
| `/thinkback` | 回溯模型思考过程 |
| `/thinkback-play` | 播放思考回溯动画 |
| `/advisor` | AI 顾问模式 |
| `/plugin` | 插件管理 |
| `/reload-plugins` | 重新加载插件 |
| `/passes` | 多轮审查 passes 管理 |
| `/feedback` | 向 Anthropic 发送反馈 |
| `/btw` | 添加附注消息 |
| `/tag` | 给对话打标签 |
| `/stickers` | 显示贴纸（彩蛋功能） |

Feature-flag 条件命令（默认不可见）：

| 命令 | Feature Flag | 描述 |
|------|-------------|------|
| `/ultraplan` | `ULTRAPLAN` | 云端超级规划（长时间异步） |
| `/voice` | `VOICE_MODE` | 语音输入模式 |
| `/bridge` | `BRIDGE_MODE` | 远程控制桥接模式 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 脚本工作流命令 |
| `/peers` | `UDS_INBOX` | 对等 session 通信 |
| `/fork` | `FORK_SUBAGENT` | 显式创建子代理 |
| `/buddy` | `BUDDY` | Buddy 协作模式 |

---

## 7.5 命令执行流程

### 从用户输入 "/" 到命令执行的完整路径

```
用户输入 "/compact some instructions"
        │
        ▼
    REPL 输入处理器
    检测到 "/" 前缀
        │
        ▼
    getCommands(cwd)                    ← 从所有来源聚合命令列表
    findCommand("compact", commands)     ← 按 name / aliases 查找
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← 检查 auth 类型门控
    isCommandEnabled(cmd)               ← 检查 feature flag / isEnabled()
        │
        ├── 检查 cmd.immediate          ← true: 绕过队列立即执行
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        返回 LocalCommandResult
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        渲染 React 组件到终端
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   返回 ContentBlockParam[]
                                   注入对话流 → 触发模型推理
```

### 命令参数解析

命令系统没有内置统一的参数解析框架——这是一个刻意的设计选择。每个命令自行处理其 `args: string` 参数，保持了极大的灵活性：

- `/compact` 直接使用 `args.trim()` 作为自定义压缩指令
- `/review` 用 `/^\d+$/.test(prNumber)` 判断是否为 PR 编号
- `/model` 在有 args 时走 `SetModelAndClose` 直接设置，无 args 时渲染交互式 `ModelPickerWrapper`
- `/resume` 支持 session ID（UUID）、自定义标题，或无参数时打开列表选择器

这种设计避免了统一解析层的复杂性，代价是每个命令需要自行处理边界情况。

### 命令输出渲染

`LocalCommandResult` 的三种类型对应不同渲染路径：

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // 渲染为文本消息
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // 触发上下文替换逻辑
  | { type: 'skip' }                      // 不渲染任何内容
```

`LocalJSXCommand` 通过 `onDone()` 回调将结果传递给 REPL：

```typescript
// types/command.ts（LocalJSXCommandOnDone）
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // 消息显示方式
    shouldQuery?: boolean                   // 是否立即触发模型查询
    metaMessages?: string[]                 // 插入模型可见但用户不可见的消息
    nextInput?: string                      // 自动填入下一条输入
    submitNextInput?: boolean               // 是否自动提交
  },
) => void
```

`display: 'system'` 表示以系统消息样式显示（灰色斜体），`display: 'user'` 则以普通用户消息显示，`display: 'skip'` 完全不显示。

---

## 7.6 代表性命令深度分析

### /compact 命令的实现细节

`/compact` 是命令系统中逻辑最复杂的命令之一，承担着对话历史压缩的核心职责。

**执行决策树**（`commands/compact/compact.ts`）：

```
/compact [instructions]
    │
    ├── 有无自定义指令?
    │   └── 无指令 → trySessionMemoryCompaction()   ← 优先尝试 Session Memory 压缩
    │                  成功则直接返回，最快路径
    │
    ├── isReactiveOnlyMode() ?
    │   └── 是 → compactViaReactive()               ← 响应式压缩路径（新架构）
    │               并行执行: executePreCompactHooks + getCacheSharingParams
    │               调用: reactiveCompactOnPromptTooLong()
    │
    └── 否 → 传统压缩路径
              microcompactMessages()                  ← 先微压缩减少 tokens
              compactConversation()                   ← 主压缩（摘要生成）
              setLastSummarizedMessageId(undefined)   ← 重置追踪指针
```

关键设计点：压缩前必须调用 `getMessagesAfterCompactBoundary(messages)` 过滤掉 REPL 为 UI scrollback 保留的已裁剪消息——这些消息不应出现在摘要中。

压缩成功后的清理序列固定为：
1. `setLastSummarizedMessageId(undefined)` — 重置消息指针
2. `suppressCompactWarning()` — 抑制"上下文即将耗尽"警告
3. `getUserContext.cache.clear?.()` — 清除用户上下文缓存
4. `runPostCompactCleanup()` — 触发后压缩钩子

**Reactive Compact 路径**利用了并行优化：

```typescript
// compact.ts:compactViaReactive（核心并行段）
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // 执行预压缩钩子（可能启动子进程）
  getCacheSharingParams(context, messages),  // 构建系统 prompt（遍历所有工具）
])
```

两者互相独立，并行执行显著减少了等待时间。

### /model 命令的模型切换逻辑

`/model` 是 `local-jsx` 类型，通过 React 组件渲染交互式选择器。

**两种执行路径**：

- **有参数**（`/model claude-sonnet-4-6`）：渲染 `SetModelAndClose` 组件，`useEffect` 中异步执行模型验证，通过 `onDone()` 立即完成
- **无参数**（`/model`）：渲染 `ModelPickerWrapper` 组件，展示完整的 `ModelPicker` 交互式界面

**模型切换的状态更新**：

```typescript
// model.tsx:handleSelect（核心状态更新）
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // 清除 session 级临时覆盖
}))
```

**模型验证层次**（从快到慢）：
1. 检查 `isModelAllowed(model)` — 组织限制白名单
2. 检查 `isOpus1mUnavailable(model)` — 1M 上下文特权检查
3. 检查 `isKnownAlias(model)` — 已知别名直接通过（跳过 API 验证）
4. `validateModel(model)` — 调用 API 验证自定义模型名

Fast Mode 与模型切换存在联动：若新模型不支持 Fast Mode，会自动关闭；若支持且已启用，则在确认消息中显示 "Fast mode ON"。

### /review 命令的代码审查流程

`/review` 展示了 `PromptCommand` 类型的典型用法——用一个简洁的 prompt 模板驱动完整的审查流程：

```typescript
// review.ts:LOCAL_REVIEW_PROMPT（完整 prompt 模板）
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

命令本身只有 4 行关键代码，其余全部由模型完成——这正是 `PromptCommand` 的设计哲学：**命令定义 WHAT，模型决定 HOW**。

与之对比的是 `/ultrareview`（`local-jsx` 类型），执行的是完全不同的路径：

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← 检查免费额度 / Extra Usage 余额
    │   ├── Team/Enterprise → 直接通过
    │   ├── 有免费次数 → 通过，附提示
    │   └── 额度耗尽 → 显示超额确认对话框
    │
    └── launchRemoteReview()
        ├── PR 模式 → teleportToRemote(branchName: "refs/pull/N/head")
        └── 分支模式 → git merge-base → git diff 检查 → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → 返回任务 URL，模型通知用户
```

`/ultrareview` 将代码审查任务"传送"到云端运行，在本地注册 `RemoteAgentTask` 后立即返回，通过轮询机制接收结果——这是一种异步任务委派模式，完全不同于本地命令的同步执行模型。

---

## 7.7 命令与 Skill 的边界

### 两者的异同

| 维度 | 命令（Command） | Skill |
|------|---------------|-------|
| 定义方式 | TypeScript 代码，硬编码逻辑 | Markdown 文件，frontmatter + prompt 内容 |
| 加载时机 | 启动时静态注册（内置）或异步加载（插件） | 运行时从文件系统扫描 |
| 执行类型 | `local` / `local-jsx` / `prompt` | 仅 `prompt`（展开为 prompt） |
| 模型可调用 | 大多数内置命令禁止模型调用（`source: 'builtin'`） | 设计上支持模型通过 SkillTool 调用 |
| 用户可见性 | 所有命令出现在 `/` typeahead | 取决于 `userInvocable` 和 `hasUserSpecifiedDescription` |
| 上下文感知 | 通过 `ToolUseContext` 访问完整应用状态 | 仅能使用 prompt 内容，无直接状态访问 |
| 来源标识 | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### 设计选择背后的考量

**为什么内置命令不用 Markdown Skill？**

内置命令需要访问应用状态（`AppState`）、调用 Node.js API（文件系统、加密）、渲染 React 组件——这些能力远超 prompt 模板所能表达的范围。`/compact` 需要调用 4 个不同的压缩策略；`/model` 需要渲染交互式 UI；`/resume` 需要读写 session 文件。这些都必须是代码。

**SkillTool 的过滤逻辑**揭示了边界的精确划定：

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← 内置命令被排除在外
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`** 是核心规则：内置命令被显式排除在模型可调用列表之外。这防止了模型通过 SkillTool 绕过权限检查直接操作会话状态。

**远程安全命令集（REMOTE_SAFE_COMMANDS）**进一步细化了这个边界：

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

仅 20 个命令在 `--remote` 模式下可用——这些命令不依赖本地文件系统、git、IDE，是纯 TUI 状态操作，可以安全地在远程 bridge 会话中执行。

---

## 7.8 设计决策分析

**决策一：三种命令类型而非统一接口**

`local` / `local-jsx` / `prompt` 的三分法看似增加了复杂度，但每种类型解决了一个不同的核心问题：
- `local` 处理有副作用但无 UI 的操作（需要返回结构化数据）
- `local-jsx` 处理需要交互式界面的操作（依托 Ink 渲染树）
- `prompt` 处理可以委派给模型的操作（最低耦合度）

若强行统一为单一接口，要么所有命令都需要处理 React 渲染（不必要的依赖），要么失去类型安全性。

**决策二：memoize by cwd 而非全局单例**

`loadAllCommands = memoize(async (cwd: string) => ...)` 以工作目录为 cache key，意味着不同目录下的 Claude Code 实例拥有独立的命令缓存。这支持了 monorepo 和多项目场景下每个目录拥有独立 Skills 集合的需求。

**决策三：不做统一参数解析**

这是有意为之的"宽松设计"。统一解析框架（如 commander.js）会强制每个命令声明完整的参数 schema，这对于 `/compact` 这类"自由文本指令"命令毫无意义。保留原始字符串让命令自行决定如何解析，以灵活性换取了一致性。

**决策四：Availability vs isEnabled 的两层门控**

两层门控解决了不同生命周期的可见性问题：
- `availability` 在命令列表构建时过滤，结果缓存，适合静态的 auth 类型检查
- `isEnabled()` 每次 `getCommands()` 调用时重新评估（不缓存），适合动态的 feature flag 检查

注释中特别说明了 `isEnabled()` 不被 memoize 的原因：`/login` 执行后 auth 状态变化，必须立即反映在命令列表中。

**决策五：内部命令不做单独包管理**

`INTERNAL_ONLY_COMMANDS` 直接通过环境变量 `USER_TYPE === 'ant'` 控制可见性，而非通过单独的 npm 包。这简化了构建复杂度，代价是外部构建时需要通过 dead code elimination 裁剪这部分代码（`filter(Boolean)` 对 `null` 条件命令同样有效）。

---

## 7.9 可迁移模式

### 模式一：命令类型三分法

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**适用场景**：任何需要同时支持"纯逻辑"、"UI 交互"和"LLM 委派"的命令系统。三者的边界非常清晰，可以直接移植到其他 REPL/CLI 框架。

**核心价值**：类型系统强制执行了职责分离，不需要运行时 isinstance 检查。

### 模式二：惰性加载 + 按 cwd memoize

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

**适用场景**：命令数量庞大（>50），或部分命令依赖重型模块的 CLI 工具。

**实现要点**：memoize key 必须包含所有影响命令集合的因素（此处是 cwd），cache 失效时机要对应实际状态变化（此处是 `clearCommandsCache()`）。

### 模式三：多源命令聚合 + 优先级排序

```typescript
return [
  ...bundledSkills,       // 最高优先级（可覆盖同名内置命令）
  ...pluginCommands,
  ...COMMANDS(),          // 最低优先级（可被覆盖）
]
```

**适用场景**：支持插件生态的 CLI 工具，需要让第三方扩展能够覆盖（override）内置行为。

**注意事项**：`findCommand` 返回列表中的第一个匹配项，因此数组顺序即是优先级顺序，设计时需明确记录。

### 模式四：Auth-gated 命令可见性

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

**适用场景**：SaaS 产品中需要对不同订阅层级的用户展示不同功能集。

**关键设计**：在命令列表过滤阶段拦截，而非在执行阶段报错——用户不会看到无法使用的命令，减少认知负担。

### 模式五：Bridge Safe / Remote Safe 白名单

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([session, exit, clear, ...])
```

**适用场景**：需要在受限环境（远程会话、沙箱、mobile bridge）下运行的命令系统。

**实现思路**：比黑名单更安全——新增命令默认不可用于受限环境，需要显式加入白名单。这避免了因疏忽导致敏感命令在错误环境中暴露。

---

## 7.10 源码索引

| 文件路径 | 行数 | 内容 |
|---------|------|------|
| `src/commands.ts` | 754 | 命令注册、聚合、过滤、查找的全部入口逻辑 |
| `src/types/command.ts` | ~250 | Command 联合类型定义、CommandBase、各子类型详细声明 |
| `src/commands/compact/compact.ts` | 287 | /compact 三路径实现（session memory / reactive / traditional） |
| `src/commands/model/model.tsx` | 296 | /model 交互式选择器 + 直接设置两路径，React Compiler 编译输出 |
| `src/commands/review.ts` | ~50 | /review（prompt 类型）和 /ultrareview（local-jsx 类型）入口 |
| `src/commands/review/reviewRemote.ts` | 316 | /ultrareview 远程启动逻辑：teleport、overage gate、任务注册 |
| `src/commands/resume/resume.tsx` | 274 | /resume session 列表选择器 UI |
| `src/commands/branch/branch.ts` | 296 | /branch 对话分叉：JSONL 复制、sessionId 重写、冲突处理 |
| `src/commands/context/context-noninteractive.ts` | 325 | /context 非交互路径：分类 token 统计，Markdown 表格渲染 |
| `src/skills/loadSkillsDir.ts` | — | Skills 目录扫描与动态加载逻辑 |
| `src/skills/bundledSkills.ts` | — | 随产品打包的内置 Skills 注册 |
| `src/plugins/builtinPlugins.ts` | — | 内置插件的 Skill 命令提取 |
| `src/utils/plugins/loadPluginCommands.ts` | — | 第三方插件命令加载与缓存 |

**关键函数索引**：

| 函数 | 文件 | 用途 |
|------|------|------|
| `getCommands(cwd)` | commands.ts | 返回当前用户可用的全部命令（主入口） |
| `findCommand(name, commands)` | commands.ts | 按名称/别名查找命令 |
| `meetsAvailabilityRequirement(cmd)` | commands.ts | auth 类型门控检查 |
| `getSkillToolCommands(cwd)` | commands.ts | 返回模型可调用的 Skill 命令集 |
| `getSlashCommandToolSkills(cwd)` | commands.ts | 返回用户可通过 / 触发的 Skill 集 |
| `isBridgeSafeCommand(cmd)` | commands.ts | 判断命令是否可在 bridge 模式执行 |
| `formatDescriptionWithSource(cmd)` | commands.ts | 用户界面中附加来源标注的描述格式化 |
| `clearCommandsCache()` | commands.ts | 清除全部命令缓存（含 Skills 和插件） |
