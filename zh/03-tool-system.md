# 第 3 章：工具系统

## 3.1 概述与定位

Claude Code 的工具系统是整个产品的执行层。LLM 负责推理和决策，但真正的副作用——读取文件、运行命令、搜索代码、访问网络——全部通过工具系统完成。工具系统是 LLM 意图与真实世界之间的唯一通道。

从规模来看，这是一个相当庞大的子系统：
- 源码快照中可见工具目录共 **40+ 个子目录**，涵盖文件操作、代码执行、Agent 协调、MCP 集成、任务管理等类别
- 核心抽象文件 `Tool.ts` 有 792 行，工具注册文件 `tools.ts` 有 389 行，工具执行引擎 `services/tools/toolExecution.ts` 有 1,745 行
- 工具结果存储模块 `utils/toolResultStorage.ts` 有 1,040 行，独立处理 token 预算问题

这个规模说明一个事实：**工具系统不是 Claude Code 的配件，而是其核心工程资产**。整个产品的可靠性、安全性和可扩展性很大程度上由工具系统的设计质量决定。

竞品分析中（cc-notebook）没有独立的工具系统章节，这是一个明显的分析盲区——本章填补这一空白。

---

## 3.2 理论基础

### 自描述工具（Self-Describing Tools）模式

传统的 API 调用中，调用方需要提前知道接口规范。Claude Code 的工具系统采用了不同的设计哲学：**每个工具自我描述自己的能力、输入格式和使用约束**。

这体现在 `Tool` 类型的几个核心字段：

```typescript
// Tool.ts:300-310（简化）
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // 3-10个词的能力摘要，供 ToolSearch 关键词匹配
  description(input, options): Promise<string>   // 动态生成描述
  prompt(options): Promise<string>               // 工具的完整系统提示
  inputSchema: Input           // Zod schema，既是文档又是验证器
  outputSchema?: z.ZodType
  // ...
}
```

`description()` 和 `prompt()` 是异步方法，这意味着工具的自描述可以**动态生成**——根据当前权限上下文、已安装的工具、环境状态来调整提示词内容。这不是静态文档，而是运行时生成的上下文感知描述。

### 插件架构与依赖注入

工具系统本质上是一个插件架构。每个工具通过 `buildTool()` 工厂函数构建，实现了统一的 `Tool` 接口，但彼此完全解耦。新增工具只需要：

1. 创建工具目录（如 `tools/MyTool/`）
2. 实现 `ToolDef` 接口
3. 在 `tools.ts` 的 `getAllBaseTools()` 中注册

工具本身不依赖彼此（循环依赖通过 lazy require 打破），但都依赖 `ToolUseContext`——这是一个贯穿整个执行链的上下文对象，包含权限状态、消息历史、应用状态等。

```typescript
// Tool.ts:167-172（简化）
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

`ToolUseContext` 的设计是典型的依赖注入：工具执行时需要的所有外部依赖都通过 context 传入，工具本身是无状态的纯函数式组件。这使得测试、隔离和子代理执行都变得可行。

### Function Calling 在 LLM 中的角色

Claude Code 遵循 Anthropic API 的 Function Calling 协议。LLM 在推理过程中可以输出 `tool_use` 块，指定要调用的工具名称和参数；执行结果以 `tool_result` 块的形式返回给 LLM，作为下一轮推理的输入。

这个循环的关键约束是：**工具定义（名称 + 输入 schema）必须在 system prompt 中发送给 LLM**，占用宝贵的上下文 token。当工具数量增加到 40+，加上 MCP 第三方工具，这个开销变得不可忽视——这直接催生了 3.6 节描述的 ToolSearch 延迟加载机制。

---

## 3.3 架构与数据结构

### buildTool() 统一抽象

`buildTool()` 是工具系统的核心工厂函数，定义在 `Tool.ts:756-769`：

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

它做一件事：将用户提供的 `ToolDef`（允许省略可选字段）与 `TOOL_DEFAULTS`（安全的默认值）合并，返回一个完整的 `Tool`。

默认值（`Tool.ts:729-742`）体现了**失败安全**（fail-closed）设计哲学：

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // 默认不并发安全
  isReadOnly: (_input?) => false,            // 默认假定会写入
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // 默认允许，由通用权限系统处理
  toAutoClassifierInput: (_input?) => '',    // 默认跳过安全分类器
  userFacingName: (_input?) => '',
}
```

值得注意的是 `isConcurrencySafe` 默认为 `false`——意味着系统宁可串行执行两个工具，也不冒险并发执行可能有副作用的操作。只有明确声明 `isConcurrencySafe: () => true` 的工具（如 GrepTool、GlobTool 等只读工具）才会被并行调度。

### 工具的核心类型定义

`Tool` 接口的方法可以分为几个功能域（`Tool.ts:297-580`）：

**执行域**
- `call(args, context, canUseTool, parentMessage, onProgress)` — 核心执行方法，返回 `Promise<ToolResult<Output>>`
- `validateInput(input, context)` — 执行前验证，返回 `ValidationResult`
- `checkPermissions(input, context)` — 权限检查，独立于通用权限系统

**描述域**（工具自描述能力）
- `description(input, options)` — 简短描述，用于 API 的 tools 列表
- `prompt(options)` — 完整的系统提示，告诉模型如何使用这个工具
- `searchHint` — 3-10 词的能力摘要，专供 ToolSearch 关键词匹配

**渲染域**（React 组件，仅 REPL 模式）
- `renderToolUseMessage(input, options)` — 工具调用开始时的 UI
- `renderToolResultMessage(content, progressMessages, options)` — 工具结果的 UI
- `renderToolUseProgressMessage(progressMessages, options)` — 执行中的进度 UI
- `renderToolUseRejectedMessage(input, options)` — 被拒绝时的 UI

**元数据域**
- `isConcurrencySafe(input)` — 声明是否可并发执行
- `isReadOnly(input)` — 声明是否只读（影响权限判断）
- `isDestructive(input)` — 声明是否不可逆（删除、覆盖、发送）
- `shouldDefer` — 是否延迟加载（由 ToolSearch 按需加载）
- `alwaysLoad` — 始终加载到 prompt（不延迟）
- `maxResultSizeChars` — 工具结果持久化到磁盘的触发阈值

`ToolResult<T>` 的结构（`Tool.ts:289-298`）也值得关注：

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

`contextModifier` 允许工具执行后修改上下文（但注释明确指出：**只有非并发安全的工具才会执行 contextModifier**——这是一个重要的并发安全约束）。

### 工具注册与发现机制

`tools.ts` 中的 `getAllBaseTools()` 是工具注册的单一真相源（`tools.ts:108-186`）。这个函数返回当前环境可用的所有内置工具，并通过多层条件控制工具的可用性：

**环境条件**（process.env）：
- `USER_TYPE === 'ant'` — Anthropic 内部工具（ConfigTool、TungstenTool、REPLTool）
- `NODE_ENV === 'test'` — 测试工具（TestingPermissionTool）
- `ENABLE_LSP_TOOL` — LSP 集成工具
- `CLAUDE_CODE_VERIFY_PLAN` — 计划验证工具

**Feature Flag 条件**（`feature()` from `bun:bundle`）：
- `PROACTIVE` / `KAIROS` — SleepTool（主动行为）
- `AGENT_TRIGGERS` — ScheduleCronTool 等定时任务工具
- `COORDINATOR_MODE` — 协调者模式相关工具
- `WEB_BROWSER_TOOL` — 浏览器工具
- `WORKFLOW_SCRIPTS` — 工作流工具

**运行时条件**：
- `isToolSearchEnabledOptimistic()` — 是否加入 ToolSearchTool
- `isTodoV2Enabled()` — 是否加入任务管理工具集
- `isAgentSwarmsEnabled()` — 是否加入团队协作工具
- `hasEmbeddedSearchTools()` — 若内置了 bfs/ugrep，则不加 GlobTool/GrepTool

工具的 **去重与排序**（`assembleToolPool()`，`tools.ts:218-248`）采用了精心设计的策略：内置工具和 MCP 工具分别排序后拼接，内置工具作为前缀，MCP 工具追加在后。这是为了保持系统提示的稳定性（prompt cache stability）——Anthropic 服务器在固定位置设置缓存断点，如果内置工具和 MCP 工具混合排序，任何新增 MCP 工具都会破坏缓存。

---

## 3.4 工具分类清单

根据 `tools/` 目录结构和 `tools.ts` 注册逻辑，可梳理出完整工具清单：

### 文件操作（File Operations）

| 工具 | 目录 | 功能 | 并发安全 |
|------|------|------|---------|
| FileReadTool | `FileReadTool/` | 读取文件，支持 PDF/图片/Notebook，分页读取 | 是 |
| FileEditTool | `FileEditTool/` | 精确字符串替换，支持 replace_all | 否 |
| FileWriteTool | `FileWriteTool/` | 写入/创建文件 | 否 |
| GlobTool | `GlobTool/` | 按 glob 模式查找文件 | 是 |
| GrepTool | `GrepTool/` | ripgrep 正则内容搜索 | 是 |
| NotebookEditTool | `NotebookEditTool/` | Jupyter Notebook 单元格编辑 | 否 |

### 代码执行（Execution）

| 工具 | 目录 | 功能 | 备注 |
|------|------|------|------|
| BashTool | `BashTool/` | Shell 命令执行，支持后台任务、沙箱 | 核心工具 |
| PowerShellTool | `PowerShellTool/` | Windows PowerShell 执行 | 条件启用 |
| REPLTool | `REPLTool/` | 隔离 VM 环境中的 REPL 执行 | Ant 内部 |

### Agent 协调（Agent Orchestration）

| 工具 | 目录 | 功能 |
|------|------|------|
| AgentTool | `AgentTool/` | 启动子代理（subagent），支持并行执行 |
| SendMessageTool | `SendMessageTool/` | 向其他 Agent 发送消息 |
| TeamCreateTool | `TeamCreateTool/` | 创建 Agent 团队 |
| TeamDeleteTool | `TeamDeleteTool/` | 删除 Agent 团队 |
| TaskCreateTool | `TaskCreateTool/` | 创建后台任务 |
| TaskGetTool | `TaskGetTool/` | 获取任务状态 |
| TaskUpdateTool | `TaskUpdateTool/` | 更新任务状态 |
| TaskListTool | `TaskListTool/` | 列出所有任务 |
| TaskStopTool | `TaskStopTool/` | 停止任务 |
| TaskOutputTool | `TaskOutputTool/` | 获取任务输出 |

### 上下文与工具发现（Context & Discovery）

| 工具 | 目录 | 功能 |
|------|------|------|
| SkillTool | `SkillTool/` | 加载和执行 Skill（~/.claude/skills/） |
| ToolSearchTool | `ToolSearchTool/` | 搜索延迟加载的工具 |
| MCPTool（动态生成）| `MCPTool/` | MCP 服务器工具（运行时动态注册） |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | 列出 MCP 资源 |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | 读取 MCP 资源 |
| LSPTool | `LSPTool/` | LSP 语言服务器集成 |

### 计划与状态（Planning & State）

| 工具 | 目录 | 功能 |
|------|------|------|
| EnterPlanModeTool | `EnterPlanModeTool/` | 进入计划模式（只读，不执行） |
| ExitPlanModeTool | `ExitPlanModeTool/` | 退出计划模式 |
| EnterWorktreeTool | `EnterWorktreeTool/` | 进入 git worktree 隔离环境 |
| ExitWorktreeTool | `ExitWorktreeTool/` | 退出 worktree 环境 |
| TodoWriteTool | `TodoWriteTool/` | 写入 Todo 列表（显示在侧边栏）|
| BriefTool | `BriefTool/` | 生成会话摘要 |

### 网络访问（Network）

| 工具 | 目录 | 功能 |
|------|------|------|
| WebFetchTool | `WebFetchTool/` | HTTP 抓取，HTML→Markdown 转换，域名安全检查 |
| WebSearchTool | `WebSearchTool/` | 网络搜索 |

### 系统与调度（System & Scheduling）

| 工具 | 目录 | 功能 | 条件 |
|------|------|------|------|
| ConfigTool | `ConfigTool/` | 读写配置 | Ant 内部 |
| SleepTool | `SleepTool/` | 等待（主动模式） | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | 合成输出（特殊用途） | — |
| ScheduleCronTool | `ScheduleCronTool/` | 创建/删除/列出定时任务 | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | 远程触发器 | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | 向用户提问（交互式） | — |

---

## 3.5 工具执行流程

### 从 LLM tool_use 到工具执行的完整流程

工具执行的入口是 `services/tools/toolExecution.ts` 中的 `runToolUse()` 函数（`toolExecution.ts:298-428`），这是一个 async generator：

```
LLM 输出 tool_use 块
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — 查找工具，支持别名（向后兼容已改名工具）
    ↓
abortController.signal.aborted? → 返回 CANCEL_MESSAGE
    ↓
streamedCheckPermissionsAndCallTool() [返回 AsyncIterable]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — Zod 类型验证
  2. tool.validateInput(input, context)  — 工具自定义验证
  3. runPreToolUseHooks()                — 执行 PreToolUse hooks
  4. canUseTool()                        — 权限检查（可能弹出 UI 确认）
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — 持久化大结果
  7. runPostToolUseHooks()               — 执行 PostToolUse hooks
    ↓
yield MessageUpdateLazy（包含 tool_result）
    ↓
下一轮 LLM 推理
```

一个重要的向后兼容设计（`toolExecution.ts:350-360`）：工具改名时，旧名称作为 `aliases` 保留。当工具在 `options.tools` 中找不到时，系统还会去 `getAllBaseTools()` 中查找别名匹配——确保历史 transcript 中的旧工具名仍然可以执行。

### 流式工具执行（Streaming Tool Execution）

工具执行通过 `Stream<MessageUpdateLazy>` 实现流式化（`toolExecution.ts:500-535`）：

```typescript
// toolExecution.ts:500-535（简化）
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // 进度消息
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // 最终结果
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

流式设计的意义：UI 可以在工具仍在执行时实时显示进度（例如 BashTool 的实时输出、AgentTool 的子代理进度）。进度消息和最终结果通过同一个 `Stream` 管道传递，简化了消费方的代码。

### 并发工具执行

Claude Code 支持 LLM 在单个响应中输出多个 `tool_use` 块，并行执行。并发的前提是：**所有工具都声明 `isConcurrencySafe: () => true`**。

执行并行时，`contextModifier` 不会执行（如 `ToolResult` 注释所说："contextModifier is only honored for tools that aren't concurrency safe"）。这是一个重要的安全约束：修改全局上下文的操作不能在并发环境中进行。

典型的并发安全工具：GrepTool、GlobTool、FileReadTool（均声明 `isConcurrencySafe: () => true`）。

---

## 3.6 ToolSearch — 延迟加载机制

### 为什么需要 ToolSearch（提示词膨胀问题）

每个工具的定义（名称 + JSON Schema + 描述）在发送给 LLM 时都会占用 token。当工具数量超过一定阈值（实验表明约 40-60 个工具），带来的问题是：

1. **token 成本上升**：每次 API 调用都携带大量工具定义
2. **注意力稀释**：LLM 面对数十个工具时，可能对每个工具的注意力都下降
3. **prompt cache 失效风险**：工具列表变化（如 MCP 工具动态加入）会使缓存失效

ToolSearch 的解决方案是**按需加载**：大多数工具以 `shouldDefer: true` 标记，不在初始提示词中发送完整 schema，只在被搜索发现后才加载。

### deferred tools 的注册与发现

工具通过 `shouldDefer` 字段声明是否延迟加载（`Tool.ts:456-462`）：

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

`isDeferredTool()` 函数（定义在 `tools/ToolSearchTool/prompt.ts`）判断一个工具是否应该被延迟：设置了 `shouldDefer: true` 且没有 `alwaysLoad: true` 的工具会被标记为 deferred。

ToolSearchTool 本身**永远不延迟**——它必须在第一轮就可用，否则其他工具无法被发现。

### 按需加载的实现

ToolSearchTool 的 `call()` 方法（`ToolSearchTool.ts:221-302`）支持两种查询模式：

**直接选择模式**（`select:` 前缀）：
```
query: "select:NotebookEdit"     → 直接返回 NotebookEditTool
query: "select:Read,Edit,Grep"   → 批量选择多个工具
```

**关键词搜索模式**：
```
query: "jupyter notebook"        → 关键词匹配，返回 NotebookEditTool 等
query: "mcp__github"             → MCP 服务器前缀匹配
```

搜索评分算法（`ToolSearchTool.ts:155-198`）：

```
工具名精确匹配部分（MCP）: +12 分
工具名精确匹配部分（普通）: +10 分
工具名部分包含关键词（MCP）: +6 分
工具名部分包含关键词（普通）: +5 分
工具名全匹配兜底: +3 分
searchHint 词边界匹配: +4 分（精心策划的能力摘要，信号强）
描述文本词边界匹配: +2 分
```

`searchHint` 字段的权重（+4 分）高于描述文本（+2 分），这鼓励工具开发者提供精准的能力摘要。例如 GrepTool 的 `searchHint: 'search file contents with regex (ripgrep)'`，FileEditTool 的 `searchHint: 'modify file contents in place'`。

搜索结果通过 `tool_reference` 块返回给 LLM（`ToolSearchTool.ts:330-352`），这是 Anthropic API 的特殊扩展，告知服务器端"请将这些工具的完整 schema 注入到当前对话的工具列表中"。

---

## 3.7 工具结果存储

### 磁盘存储策略

工具执行的结果可能非常大（例如读取一个 10MB 的日志文件，运行一个输出大量内容的命令）。直接将大结果放入消息历史会造成 token 浪费，并使后续请求的 context 膨胀。

`utils/toolResultStorage.ts` 实现了**按需持久化**策略：

1. 计算结果大小（`contentSize()`）
2. 对比工具的 `maxResultSizeChars` 阈值（通过 `getPersistenceThreshold()` 解析）
3. 超过阈值的结果写入 `~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`
4. 替换为包含文件路径 + 预览的消息

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

`PREVIEW_SIZE_BYTES = 2000`（约 2KB），预览截取到最后一个换行符，避免截断中间的行。

一个关键的**幂等性**设计（`toolResultStorage.ts:145-158`）：写入时使用 `{ flag: 'wx' }`（exclusive create），如果文件已存在则忽略写入错误，直接使用已有文件生成预览。这保证了 microcompact 重放历史消息时不会重复写入，也不会因 EEXIST 报错。

FileReadTool 有一个特殊处理：`maxResultSizeChars: Infinity`——读取工具的结果永远不持久化到磁盘。原因在注释中说明："persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits"（持久化会产生循环：Read 读文件，结果太大被持久化为文件，模型再用 Read 去读那个文件……）。

### Token 预算管理

`toolResultStorage.ts` 还实现了一个更宏观的**消息级别工具结果预算（per-message aggregate budget）**。这是由 `ContentReplacementState` 机制驱动的（`toolResultStorage.ts:395-440`）：

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // 已经过预算检查的 tool_use_id（结果冻结）
  replacements: Map<string, string>  // 被替换的 ID → 替换后的内容字符串
}
```

核心约束：**结果一旦被判断（替换或不替换），就永远不会改变**（通过 `seenIds` 集合保证）。这是为了 prompt cache 稳定性——同一个 tool_use_id 的处理结果在整个会话中必须保持一致，否则缓存会因为内容变化而失效。

预算限制由 GrowthBook feature flag `tengu_hawthorn_window` 动态控制，在超过整条消息的工具结果总量上限时，系统会将最大的工具结果替换为磁盘持久化版本，直到总量降到预算内。

---

## 3.8 设计决策分析

### 自描述 vs 外部注册的 tradeoff

Claude Code 选择了**自描述**模式（每个工具携带自己的 schema、描述、提示词、渲染逻辑），而不是将这些信息集中在一个注册表中。

优点：
- **工具完全自包含**：新增工具只需一个目录，不需要修改中央注册表的逻辑
- **描述可以动态生成**：`description()` 和 `prompt()` 是异步函数，可以根据环境、权限、安装状态动态调整内容
- **渲染逻辑与工具共存**：React 渲染组件直接在工具文件中，改工具行为和改 UI 是同一个 PR

缺点：
- **工具接口膨胀**：`Tool` 类型有 40+ 个方法/字段，新工具作者需要了解大量接口细节
- **重复代码**：每个工具都有 `renderToolUseMessage`、`renderToolResultMessage` 等渲染方法，模式高度相似
- **`buildTool()` 无法完全消除**：提供了默认值，但仍有很多方法需要每个工具自己实现

实际上，Claude Code 通过**共享 UI 组件**（如 `tools/shared/`）和**模式提取**（如 `lazySchema()`）来缓解代码重复，但根本性的接口复杂度仍然存在。

### 为什么某些工具是延迟加载的

ToolSearch 的延迟加载决策遵循一个原则：**第一轮对话可能不需要的工具都应该延迟**。

`alwaysLoad` 的工具（永远不延迟）应该满足：模型在第一轮就需要知道它的存在。典型例子是 AgentTool、BashTool、FileReadTool——这些是任何编程任务的基础工具。

`shouldDefer` 的工具（延迟加载）通常是：特定场景才需要的工具（NotebookEditTool 只有 Jupyter 任务才需要）、大量 MCP 工具（用户安装了几十个 MCP 服务器，但每次对话只用到其中少数）。

MCP 工具默认会根据工具数量触发 ToolSearch 机制，但可以通过在工具 metadata 中设置 `_meta['anthropic/alwaysLoad']` 强制不延迟。

### 工具权限的分层设计

工具权限采用**三层防御**设计：

1. **Zod 类型验证**（`checkPermissionsAndCallTool` 第一步）：工具的 inputSchema 对参数类型进行严格验证，LLM 生成错误类型的参数会被拒绝并返回错误提示
2. **工具自定义验证**（`validateInput()`）：工具实现自己的业务逻辑验证，例如 FileEditTool 检查 old_string 和 new_string 不相同，检查文件大小是否超过 1GiB
3. **通用权限系统**（`canUseTool()` + `checkPermissions()`）：基于用户设置的 allow/deny 规则、工具是否只读、是否破坏性操作等进行最终判断，可能弹出交互式确认

这三层是顺序执行的，任何一层失败都会短路，不进入下一层。

---

## 3.9 可迁移模式

### 自描述工具模式的通用设计

Claude Code 的工具系统提炼出的最具迁移价值的模式是：**工具即自包含插件**。

核心原则：
1. **Schema 即文档即验证器**：用 Zod schema 定义输入，自动生成 JSON Schema 给 LLM，同时在运行时验证 LLM 的输出
2. **工厂函数 + 安全默认值**：`buildTool()` 提供失败安全的默认行为（默认不并发安全、默认只读为假），工具开发者只需声明自己的例外
3. **searchHint 精简摘要**：3-10 词的能力描述，专门为关键词搜索优化，与完整描述分离
4. **能力声明优于运行时判断**：`isReadOnly()`、`isConcurrencySafe()`、`isDestructive()` 让调度器无需执行工具就能做出调度决策

### Doramagic 的工具系统（Brick 系统）可借鉴之处

Doramagic 的 Brick 系统（278+ 积木）与 Claude Code 的工具系统有深层相似性，但也有本质区别：

**相似点**：
- 都是"插件式"架构：每个 Brick/Tool 是自包含的功能单元
- 都需要描述机制：让 LLM 知道何时使用哪个工具/积木
- 都有分类体系：按功能领域组织

**可借鉴的具体模式**：

1. **`searchHint` 类比 Brick 的 tags**：Claude Code 为每个工具提供 3-10 词的精简能力描述，专门用于搜索匹配。Doramagic 的积木目前用 tags 和 categories 组织，可以增加一个 `hint` 字段，专门优化模型的积木发现效率。

2. **延迟加载 → 积木按需激活**：Claude Code 的 deferred tools 机制说明：把所有积木都塞进系统提示并不是好主意。Doramagic 可以参考 `shouldDefer` 的设计，将不常用的积木（领域专用积木）设为延迟加载，只在模型明确需要时才激活。

3. **`maxResultSizeChars` → 积木输出预算**：每个工具自己声明结果的最大 token 预算，超过则压缩。Doramagic 的积木输出（提取的知识 JSON）同样可能很大，参考这个机制实现"摘要优先，详情按需"的输出策略。

4. **`isConcurrencySafe` → 积木并行声明**：Doramagic 的知识提取管线中，多个积木可能同时作用于同一个代码库。明确声明积木的并发安全性，调度器可以自动决定哪些积木可以并行运行，哪些需要串行。

5. **三层权限防御 → 积木运行安全**：对于 Doramagic 作为 OpenClaw Skill 运行的场景，Brick 执行的合法性校验可以参考这个三层设计：schema 验证 → 业务验证 → 平台权限检查。

**本质差异**：Claude Code 的工具主要面向**确定性操作**（读文件、执行命令），输出是可精确定义的。Doramagic 的积木面向**知识提取**，输出是语义性的。这意味着 Doramagic 无法像 Claude Code 那样用 Zod schema 严格验证积木输出——这正是 Doramagic "代码说事实，AI 说故事"架构原则的意义所在：确定性的骨架（facts 提取）可以用 schema 约束，非确定性的解读（stories 生成）则不需要。

---

## 3.10 源码索引

| 文件 | 行数 | 关键内容 |
|------|------|---------|
| `src/Tool.ts` | 792 | `Tool` 类型定义、`buildTool()`、`ToolUseContext`、`ToolResult`、`TOOL_DEFAULTS` |
| `src/tools.ts` | 389 | `getAllBaseTools()`、`getTools()`、`assembleToolPool()`、`getMergedTools()`、`filterToolsByDenyRules()` |
| `src/services/tools/toolExecution.ts` | 1,745 | `runToolUse()`、`checkPermissionsAndCallTool()`、`streamedCheckPermissionsAndCallTool()`、`buildSchemaNotSentHint()` |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 471 | `searchToolsWithKeywords()`、`parseToolName()`、关键词评分算法、`select:` 前缀直接选择 |
| `src/utils/toolResultStorage.ts` | 1,040 | `persistToolResult()`、`buildLargeToolResultMessage()`、`ContentReplacementState`、`enforceToolResultBudget()` |
| `src/tools/BashTool/BashTool.tsx` | ~1,800+ | `isSearchOrReadBashCommand()`、沙箱、后台任务、进度显示 |
| `src/tools/FileEditTool/FileEditTool.ts` | ~500+ | 字符串替换、大文件保护（1GiB 限制）、secret 检测 |
| `src/tools/FileReadTool/FileReadTool.ts` | ~600+ | 多格式支持（PDF/图片/Notebook）、token 计数、`maxResultSizeChars: Infinity` |
| `src/tools/GrepTool/GrepTool.ts` | ~400+ | ripgrep 集成、head_limit/offset 分页、`DEFAULT_HEAD_LIMIT = 250` |
| `src/tools/WebFetchTool/utils.ts` | ~450+ | 域名黑名单检查、LRU 缓存（50MB/15min）、HTML→Markdown 转换 |
| `src/tools/MCPTool/classifyForCollapse.ts` | ~350 | MCP 工具的 search/read 分类（Slack/GitHub/Linear/Jira 等 20+ 服务商预置规则） |

**关键常量**（分散在多个文件）：
- `PREVIEW_SIZE_BYTES = 2000`（toolResultStorage.ts）— 大结果预览大小
- `DEFAULT_HEAD_LIMIT = 250`（GrepTool.ts）— grep 默认结果上限
- `MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024`（WebFetchTool/utils.ts）— 网络抓取 10MB 上限
- `FETCH_TIMEOUT_MS = 60_000`（WebFetchTool/utils.ts）— HTTP 请求 60 秒超时
- `CACHE_TTL_MS = 15 * 60 * 1000`（WebFetchTool/utils.ts）— URL 缓存 15 分钟
- `PROGRESS_THRESHOLD_MS = 2000`（BashTool.tsx）— 超过 2 秒显示进度
- `MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024`（FileEditTool.ts）— 文件编辑 1GiB 限制
