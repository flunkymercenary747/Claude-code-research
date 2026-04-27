# 第 4 章：Agent 编排与多代理架构

## 4.1 概述与定位

Claude Code 的多代理系统是整个产品架构中最复杂的子系统，涵盖约 8,700 行核心代码，跨越 12 个关键模块。这套系统解决了一个根本性的工程问题：**如何让一个单线程 REPL 应用安全、高效地编排多个 LLM Agent 的并发执行**。

系统提供三种递进的协作模式：

| 模式 | 触发方式 | 并发度 | 通信机制 | 隔离级别 |
|------|---------|--------|---------|---------|
| **Subagent（默认）** | AgentTool 调用 | 同步/异步 | 函数返回值 | 进程内 AsyncGenerator |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | 全异步 | `<task-notification>` XML | 独立 AbortController |
| **Team Mode** | `spawnTeammate()` + TeamFile | 持久化并行 | 文件邮箱 + 轮询 | Tmux Pane / InProcess / Remote |

这三种模式并非独立实现，而是共享同一个 `runAgent()` 核心引擎（`runAgent.ts`），通过参数组合实现不同的行为特征——这是整个系统最优雅的设计决策之一。

**源码规模统计：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `AgentTool.tsx` | 1,397 | 统一入口、路由决策、生命周期管理 |
| `runAgent.ts` | 973 | Agent 执行引擎、query() 循环 |
| `loadAgentsDir.ts` | 755 | Agent 定义解析（Markdown/JSON/Plugin） |
| `agentToolUtils.ts` | 686 | 工具过滤、权限、结果序列化 |
| `UI.tsx` | 871 | Agent 进度与结果渲染 |
| `coordinatorMode.ts` | 369 | Coordinator 系统提示与上下文 |
| `SendMessageTool.ts` | 917 | 5 路消息路由 |
| `spawnMultiAgent.ts` | 1,093 | Teammate 生成（Tmux/InProcess） |
| `inProcessRunner.ts` | 1,552 | InProcess 后端完整实现 |
| `teammateMailbox.ts` | 1,183 | 文件邮箱协议 |
| `worktree.ts` | 1,519 | Git Worktree 隔离 |

## 4.2 理论基础

### 4.2.1 Actor 模型与 Agent 编排的关系

Claude Code 的多代理架构是 Actor 模型在 LLM 编排领域的一个务实变体。经典 Actor 模型（Hewitt, 1973）的三个核心原语——**接收消息、创建新 Actor、发送消息**——在代码中有清晰的对应：

| Actor 原语 | Claude Code 实现 | 源码位置 |
|-----------|-----------------|---------|
| 接收消息 | `waitForNextPromptOrShutdown()` 轮询循环 | `inProcessRunner.ts:689-868` |
| 创建 Actor | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| 发送消息 | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

但与纯 Actor 模型有两个关键偏离：

1. **非对称层级**：Leader 拥有全局视图（AppState），Worker 只有自己的 ToolUseContext。这不是对等的 Actor，而是有明确 Leader-Worker 层级的监督树（supervision tree）。
2. **共享状态通道**：InProcess 后端的 Teammate 通过 `setAppStateForTasks` 共享根 AppState store（`runAgent.ts:336-337`），而非纯消息传递。这是对 Actor 模型的务实妥协——在单进程内，共享状态比序列化消息更高效。

### 4.2.2 消息传递 vs 共享内存的并发模型

系统同时使用了两种并发模型，根据隔离级别选择：

**消息传递模型**（Team Mode - Tmux Pane 后端）：
```
Leader → writeToMailbox("worker-1", {...}) → 文件系统 → readMailbox() → Worker
```
通信通过 JSON 文件 + 文件锁实现，`teammateMailbox.ts` 的 `LOCK_OPTIONS` 配置了指数退避重试（10 次重试，5-100ms）来序列化并发写入：

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**共享内存模型**（InProcess 后端）：
```
Leader → setAppState(prev => {...}) → 同一 AppState store ← getAppState() ← Worker
```
InProcess Teammate 通过 `toolUseContext.setAppStateForTasks` 直接读写根 store。竞态通过 React-style 的 `setAppState(prev => {...})` 函数式更新语义避免（虽然底层不是 React，但采用了相同的 CAS 模式）。

### 4.2.3 分布式系统中的协调者模式

Coordinator Mode 的设计映射了分布式系统中经典的 Coordinator 模式（也叫 Master-Worker），但增加了一个独特的约束：**Coordinator 本身是一个 LLM Agent，它的"协调逻辑"不是硬编码的，而是通过 system prompt 编程的**。

`coordinatorMode.ts:126-369` 中定义的 `getCoordinatorSystemPrompt()` 函数返回约 5,000 字符的结构化 prompt，其中包含完整的 Worker 调度策略：

```typescript
// coordinatorMode.ts:161-167 — 关键调度规则
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

这种"通过 prompt 编程协调逻辑"的模式，意味着 Coordinator 的行为可以通过修改 prompt 来调整——研发→综合→实现→验证的四阶段工作流不是代码强制的，而是通过 LLM 的指令遵循能力实现的。这与传统分布式 Coordinator 的硬编码调度逻辑形成了鲜明对比。

## 4.3 架构与数据结构

### 4.3.1 整体架构图（Leader-Worker）

```
                    ┌─────────────────────────────────────────┐
                    │           Human User (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ user input
                    ┌──────────────▼──────────────────────────┐
                    │         Main REPL (query() loop)         │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — 路由决策   │     │
                    │    └──┬─────────┬─────────┬───────┘     │
                    │       │         │         │              │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐     │
                    │  │ Sync   │ │ Async  │ │Teammate │     │
                    │  │Agent   │ │Agent   │ │Spawn    │     │
                    │  │(block) │ │(fire&  │ │         │     │
                    │  │        │ │forget) │ │         │     │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘     │
                    │      │          │         │              │
                    │      └────┬─────┘    ┌────▼──────────┐  │
                    │           │          │  spawnMulti-   │  │
                    │      ┌────▼────┐     │  Agent.ts      │  │
                    │      │runAgent │     └────┬───────────┘  │
                    │      │  .ts    │          │              │
                    │      │         │     ┌────▼──────────┐  │
                    │      │ query() │     │  3 Backends:   │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    通信层：
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → parent collects      │
    │  Async Agent:   <task-notification> XML → user msg   │
    │  Teammate:      文件邮箱 (.claude/teams/*/inboxes/)  │
    │  InProcess:     AppState 共享 + mailbox fallback     │
    │  Remote (ant):  teleportToRemote() → CCR session     │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 AgentDefinition 类型系统

Agent 定义采用了三层联合类型设计：

```typescript
// loadAgentsDir.ts — 核心类型层级

// 基础类型：所有 Agent 共享的字段
type BaseAgentDefinition = {
  agentType: string              // 路由键（如 "Explore", "worker"）
  whenToUse: string              // LLM 选择 Agent 的依据
  tools?: string[]               // 白名单（undefined = 全部）
  disallowedTools?: string[]     // 黑名单
  model?: string                 // 'inherit' | 具体模型名
  effort?: EffortValue           // 推理努力级别
  permissionMode?: PermissionMode // 权限继承策略
  maxTurns?: number              // 最大对话轮次
  background?: boolean           // 始终后台运行
  isolation?: 'worktree' | 'remote' // 隔离模式
  memory?: AgentMemoryScope      // 持久记忆
  omitClaudeMd?: boolean         // 省略 CLAUDE.md（节省 ~5-15 Gtok/week）
  // ...
}

// Built-in Agent：动态 prompt，无静态 systemPrompt
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent：从 Markdown/JSON 加载
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent：来自插件系统
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// 最终联合类型
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

这个设计的精妙之处在于 `getSystemPrompt` 方法：Built-in Agent 接收 `toolUseContext` 参数（可以根据当前工具集动态调整 prompt），而 Custom/Plugin Agent 使用闭包捕获解析时的 Markdown 内容。这意味着：

- **Built-in Agent 的 prompt 是动态的**：每次调用可能不同
- **Custom Agent 的 prompt 是静态的**：由 Markdown 文件定义，但如果启用了 `memory`，会在运行时追加记忆内容（`loadAgentsDir.ts:335-340`）

Agent 定义的加载优先级遵循覆盖链：`builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`，通过 `getActiveAgentsFromList()` 用 Map 实现后来者覆盖（`loadAgentsDir.ts:169-186`）。

### 4.3.3 三种执行后端的统一抽象

三种后端共享一套 `runAgent()` AsyncGenerator 接口，但在进程模型和通信机制上截然不同：

| 维度 | Tmux Pane | InProcess | Remote (ant-only) |
|------|-----------|-----------|-------------------|
| **进程模型** | 独立 Claude CLI 进程 | 同进程 AsyncLocalStorage 隔离 | CCR 远程会话 |
| **启动方式** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **通信** | 文件邮箱轮询 (500ms) | 共享 AppState + 文件邮箱 fallback | HTTP API |
| **权限** | 独立权限上下文 | Leader UI 队列桥接 | 远程独立 |
| **资源开销** | 高（完整进程） | 低（共享 V8 堆） | 极高（远程实例） |
| **生存期** | 独立于 Leader | 绑定 Leader 进程 | 独立 |
| **检测逻辑** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

后端检测和降级在 `spawnMultiAgent.ts:339-375` 实现了一套优雅的降级链：

```
iTerm2 (it2 backend) → Tmux → InProcess fallback
```

如果检测到 iTerm2 但未安装 it2 CLI，系统会弹出交互式 setup prompt（`It2SetupPrompt`），让用户选择安装 it2 或降级到 Tmux。

### 4.3.4 通信协议的数据结构

**文件邮箱消息格式**（`teammateMailbox.ts:42-49`）：

```typescript
type TeammateMessage = {
  from: string       // 发送者名称
  text: string       // 消息内容（可以是纯文本或 JSON 结构化消息）
  timestamp: string  // ISO 时间戳
  read: boolean      // 已读标记
  color?: string     // 发送者颜色标识
  summary?: string   // UI 预览摘要（5-10 词）
}
```

邮箱路径遵循固定格式：`~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**结构化消息类型**（通过 `text` 字段的 JSON 编码传递）：

| 消息类型 | 方向 | 用途 |
|---------|------|------|
| `shutdown_request` | Leader → Worker | 请求关闭 |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | 关闭响应 |
| `idle_notification` | Worker → Leader | 空闲通知 |
| `permission_request` | Worker → Leader | 权限请求 |
| `permission_response` | Leader → Worker | 权限响应 |
| `plan_approval_request` | Worker → Leader | Plan Mode 审批 |
| `plan_approval_response` | Leader → Worker | 审批响应 |
| `sandbox_permission_request` / `_response` | 双向 | 网络沙箱权限 |
| `task_assignment` | Leader → Worker | 任务分配 |
| `team_permission_update` | Leader → Workers | 权限广播 |

## 4.4 核心算法与流程

### 4.4.1 AgentTool 路由决策树（完整）

`AgentTool.call()` 是系统的统一入口点，其路由逻辑在 `AgentTool.tsx:238-764` 中实现。完整决策树如下：

```
AgentTool.call(input) 入口
│
├─ [1] Team name + name 参数都存在？
│   ├─ YES: 是否是 Teammate 尝试嵌套生成？
│   │   ├─ YES: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NO: → spawnTeammate() (return teammate_spawned)
│   └─ NO: 继续
│
├─ [2] 解析 effectiveType (subagent_type)
│   ├─ 显式指定 → 使用指定值
│   ├─ 未指定 + Fork Gate ON → undefined (Fork 路径)
│   └─ 未指定 + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (Fork 路径)
│   ├─ YES: 递归 Fork 检查
│   │   ├─ 已在 Fork 子进程中 → throw
│   │   └─ 通过 → selectedAgent = FORK_AGENT
│   └─ NO: 从 activeAgents 查找
│       ├─ 找到 → selectedAgent = found
│       ├─ 被 permission deny → throw (带 deny rule 信息)
│       └─ 不存在 → throw (列出可用 agents)
│
├─ [4] 解析 effectiveIsolation
│   ├─ 'remote' (ant-only) → teleportToRemote() → return remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → 后续步骤使用 worktreePath
│
├─ [5] 构建 system prompt 和 prompt messages
│   ├─ Fork 路径: 继承父 prompt + buildForkedMessages()
│   └─ Normal: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] shouldRunAsync 判定
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — BUT NOT isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → return { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → 进入 while(true) 循环
│       ├─ Race: nextMessage vs backgroundSignal
│       │   ├─ background wins → 切换到异步执行 (wasBackgrounded=true)
│       │   └─ message wins → yield message, track progress
│       └─ 循环结束 → finalizeAgentTool() → return AgentToolResult
```

### 4.4.2 runAgent() AsyncGenerator 执行流程

`runAgent()` 是整个多代理系统的核心引擎（`runAgent.ts:247-860`），它是一个 `AsyncGenerator<Message, void>`——每 yield 一条 Message，调用者就可以处理它（记录、显示、或推入后台队列）。

**执行流程的关键阶段：**

1. **工具解析**：`resolveAgentTools()` 将 Agent 定义的 `tools` 白名单解析为实际 Tool 对象，同时应用 `disallowedTools` 黑名单（`runAgent.ts:500-502`）

2. **System Prompt 构建**：根据 `override?.systemPrompt` 或 `getAgentSystemPrompt()` 构建，Explore/Plan Agent 跳过 `claudeMd` 和 `gitStatus`，节省 fleet-wide ~5-15 Gtok/week（`runAgent.ts:389-409`）

3. **AbortController 策略**（`runAgent.ts:524-528`）：
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // 外部控制（async 路径）
     : isAsync
       ? new AbortController()      // 异步：独立 controller
       : toolUseContext.abortController  // 同步：共享父 controller
   ```

4. **权限覆盖**（`runAgent.ts:414-497`）：Agent 的 `permissionMode` 会覆盖父级的模式，但 `bypassPermissions`、`acceptEdits`、`auto` 三种父级模式始终优先——这确保了管理员设置的安全策略不会被子 Agent 降级。

5. **核心循环**——直接调用 `query()` 并 yield（`runAgent.ts:748-806`）：
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... 处理 stream_event, attachment, recordable messages
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **清理 finally 块**（`runAgent.ts:816-858`）：MCP 清理、session hooks 清理、prompt cache tracking、文件状态缓存释放、Perfetto 注销、AppState todos 清理、后台 bash 任务 kill——共 9 项清理操作，确保资源不泄漏。

### 4.4.3 异步 Agent 生命周期（fire-and-forget）

异步 Agent 的完整生命周期由 `runAsyncAgentLifecycle()`（`agentToolUtils.ts:322-497`）驱动：

```
registerAsyncAgent() → 注册任务到 AppState
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — 收集所有消息
   │   ├─ agentMessages.push(message)
   │   ├─ 如果 task.retain → 追加到 AppState.tasks[taskId].messages
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — SDK 进度事件
   │
   ├─ finalizeAgentTool() — 提取最终结果
   │
   ├─ completeAsyncAgent() — 标记完成（FIRST，在任何慢操作之前）
   │   │                      ↑ 关键设计：gh-20236 修复
   │   │                        classifyHandoff 和 worktree cleanup 可能 hang
   │   │                        不能阻塞状态转换
   │
   ├─ classifyHandoffIfNeeded() — 安全分类器检查（可选）
   │
   ├─ getWorktreeResult() — worktree 清理
   │
   └─ enqueueAgentNotification() — 以 <task-notification> XML 通知父级
```

**gh-20236 修复**是一个值得记录的设计决策：`completeAsyncAgent()` 被放在 `classifyHandoffIfNeeded()` 和 `getWorktreeResult()` 之前调用。注释明确说明原因：

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 工具过滤与权限继承

工具过滤是一个三层过滤链（`agentToolUtils.ts:66-115`）：

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS — 所有 Agent 禁止使用的工具
Layer 2: CUSTOM_AGENT_DISALLOWED_TOOLS — 仅 Custom Agent 额外禁止的工具
Layer 3: ASYNC_AGENT_ALLOWED_TOOLS — 异步 Agent 的白名单（反转逻辑）
```

特殊例外：
- MCP 工具（`mcp__` 前缀）始终允许
- `ExitPlanMode` 在 Plan Mode 下始终允许
- InProcess Teammate 在 Agent Swarms 模式下可以使用 `AgentTool`（生成同步子 Agent）和 Task 工具（共享任务列表协调）

工具解析还支持通配符（`'*'` 或 `undefined` = 全部工具）和 Agent-scoped 限制（`AgentTool(worker, researcher)` 语法，`agentToolUtils.ts:165-172`）。

### 4.4.5 Coordinator Mode 四阶段工作流

Coordinator Mode 的核心逻辑在 `coordinatorMode.ts:126-369` 的 `getCoordinatorSystemPrompt()` 中通过 prompt 定义。它将所有任务分解为四个阶段：

**Phase 1: Research**（Worker 并行执行）
- 多个 Worker 同时探索代码库
- 关键 prompt 指令：*"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Phase 2: Synthesis**（Coordinator 自己做）
- 这是最关键的阶段——Coordinator 必须亲自阅读研究结果并理解
- 明确禁止的反模式：*"Never write 'based on your findings'"*
- 要求产出 synthesized spec，包含具体文件路径、行号和修改内容

**Phase 3: Implementation**（Worker 执行）
- Coordinator 决定是 continue（`SendMessageTool`）还是 spawn fresh（`AgentTool`）
- 决策依据是上下文重叠度（prompt 中有完整决策表）

**Phase 4: Verification**（独立 Worker）
- 明确要求独立验证：*"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- 验证标准：*"proving the code works, not confirming it exists"*

### 4.4.6 Team Mode 持久化协作

Team Mode 基于 TeamFile（`.claude/teams/{team_name}/team.json`）实现持久化团队状态。与 Coordinator Mode 的 fire-and-forget Worker 不同，Teammate 是**长驻进程**：

1. **生成**：`spawnTeammate()` 创建 Tmux pane 或 InProcess task
2. **运行**：Teammate 执行 prompt → 完成 → 发送 `idle_notification` → 等待下一个 prompt
3. **通信**：所有消息通过文件邮箱（任何后端都可以用文件系统通信）
4. **关闭**：Leader 发送 `shutdown_request` → Teammate 的 LLM 决定 approve 或 reject

InProcess Runner 的主循环（`inProcessRunner.ts:883-1464`）实现了完整的持久化语义：

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. 运行当前 prompt (调用 runAgent())
  // 2. 标记为 idle
  // 3. 发送 idle_notification 给 Leader
  // 4. waitForNextPromptOrShutdown() — 轮询邮箱
  //    ├─ shutdown_request → 传给 LLM 决定
  //    ├─ new_message → 设为下一轮 prompt
  //    └─ aborted → shouldExit = true
}
```

值得注意的是消息优先级策略（`inProcessRunner.ts:760-804`）：
1. 最高优先级：`shutdown_request`（Leader 的关闭指令不会被淹没）
2. 其次：来自 `team-lead` 的消息（Leader 代表用户意图）
3. 最后：FIFO 队列中的 peer 消息

### 4.4.7 文件邮箱通信协议

文件邮箱是所有后端的通信基座。其设计选择了**简单性**而非性能：

**写入协议**（`teammateMailbox.ts:133-191`）：
```
1. ensureInboxDir() — 确保目录存在
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — 原子创建（如果不存在）
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — 获取文件锁
4. readMailbox() — 锁内重新读取（避免脏读）
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — 写回
7. release() — 释放锁
```

**读取协议**（`teammateMailbox.ts:83-107`）：
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. 返回 TeammateMessage[]
```

注意读取是**无锁**的——这是有意设计。读取端只需要最终一致性，而写入端通过 `lockfile` 保证了原子性。

### 4.4.8 SendMessage 5 路路由

`SendMessageTool.call()` 实现了 5 条独立的消息路由路径（`SendMessageTool.ts`）：

```
input.to 的值
│
├─ [路由1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — 跨机器 Remote Control
│   （需要 safety check：跨机器消息需要用户显式同意）
│
├─ [路由2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — 本地 Unix Domain Socket
│
├─ [路由3] agentNameRegistry 或 toAgentId 匹配
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task stopped/evicted → resumeAgentBackground()
│       （自动从磁盘 transcript 恢复已停止的 Agent）
│
├─ [路由4] to === '*'
│   → handleBroadcast() — 遍历 TeamFile.members 逐一写入邮箱
│
└─ [路由5] 其他
    ├─ 纯文本 → handleMessage() — 写入邮箱
    └─ 结构化消息 → 分发到对应 handler：
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

路由 3 中的**自动恢复**机制尤为精妙：当向一个已停止的 Agent 发送消息时，系统会自动从磁盘 transcript 恢复它并在后台运行。这意味着 Coordinator 可以通过 `SendMessage` 无缝地 continue 一个之前完成的 Worker，而无需关心它是否还在运行。

### 4.4.9 权限委托完整流程

InProcess Teammate 的权限处理是整个系统中最复杂的部分之一（`inProcessRunner.ts:127-449`）。核心挑战是：**后台 Agent 如何请求人类授权？**

解决方案是两层 fallback：

**主路径：Leader UI 队列桥接**
```
Worker 触发需要权限的工具
  → createInProcessCanUseTool() 被调用
  → hasPermissionsToUseTool() 返回 { behavior: 'ask' }
  → 检查 Bash classifier auto-approval（如果可用）
  → getLeaderToolUseConfirmQueue() — 获取 Leader 的 UI 确认队列
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ Worker 身份标识
     └→ Leader terminal 显示带 Worker badge 的权限对话框
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Fallback 路径：邮箱权限请求**
```
Worker 触发需要权限的工具
  → Leader UI 队列不可用
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → 轮询自己的邮箱（500ms 间隔）
  → 等待 Leader 写回 permission_response
  → processMailboxPermissionResponse()
```

权限更新的传播也很重要：当 Leader approve 一个权限并选择 "Always allow" 时，`persistPermissionUpdates()` 写入磁盘，同时 `getLeaderSetToolPermissionContext()` 将更新写回 Leader 的共享上下文——但会 `preserveMode: true`，防止 Worker 的 `acceptEdits` 模式泄漏回 Coordinator（`inProcessRunner.ts:275-277`）。

### 4.4.10 Worker 完整生命周期

```
诞生
  │
  ├─ Sync Agent 路径:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → return AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ Async Agent 路径:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ InProcess Teammate 路径:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — 持久循环:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → 新消息/shutdown/abort → 决定是否继续
          }

执行中
  │
  ├─ query() 循环 → API 调用 → tool_use → canUseTool 检查
  │   ├─ allow → 执行工具
  │   ├─ deny → 工具被拒绝
  │   └─ ask → 权限对话框（sync）或 邮箱权限（async/teammate）
  │
  ├─ 进度追踪:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (SDK 事件)
  │
  └─ 自动后台化（Sync Agent only）:
      backgroundPromise race → 如果用户按 Ctrl+Z
      → wasBackgrounded = true → 继续在后台跑

通信
  │
  ├─ Sync Agent: yield message → 父级直接收集
  ├─ Async Agent: <task-notification> 注入父级 user messages
  └─ Teammate: writeToMailbox() → Leader 轮询读取

终止
  │
  ├─ 正常完成: finalizeAgentTool() → 提取最终文本 → 标记 completed
  ├─ 用户 Kill: AbortError → killAsyncAgent() → 提取 partialResult → 通知
  ├─ 错误: catch → failAsyncAgent() → 通知错误
  └─ 清理: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Worktree 隔离的创建与清理

Git Worktree 为 Agent 提供了文件系统级别的隔离（`worktree.ts`）。核心流程：

**创建**（`worktree.ts:234-374`）：
```
1. validateWorktreeSlug(slug) — 防止路径遍历攻击
2. 快速恢复检查: readWorktreeHeadSha() — 如果 worktree 已存在，跳过 fetch
3. 如果不存在:
   a. 尝试读取本地 origin/<default> ref（避免 `git fetch` 的 6-8s 开销）
   b. 如果本地不存在 → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. 可选: sparse-checkout（仅检出指定路径）
4. performPostCreationSetup():
   - 复制 settings.local.json
   - 配置 git hooks（处理 husky 的 core.hooksPath 问题）
   - 符号链接 node_modules 等大目录
   - 复制 .worktreeinclude 指定的 gitignored 文件
```

**清理决策**（`AgentTool.tsx:644-685`）：
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // Hook-based 始终保留
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // 无变更，删除 worktree
    }
  }
  return { worktreePath, worktreeBranch }; // 有变更，保留
};
```

关键安全措施：
- `validateWorktreeSlug()` 验证每个 `/` 分隔的段都匹配 `[a-zA-Z0-9._-]+`，防止 `../../../` 路径遍历
- `flattenSlug()` 将嵌套 slug 展平（`user/feature` → `user+feature`），避免 git ref D/F 冲突和目录嵌套问题
- `GIT_NO_PROMPT_ENV` 禁用所有 git 凭据提示，防止 CLI hang

## 4.5 设计决策分析

### 4.5.1 为什么选择文件邮箱而非 IPC

文件邮箱看起来是一个"原始"的选择——为什么不用 Unix Domain Socket、Named Pipe、或 gRPC？

**核心原因：后端无关性**。文件系统是所有三种后端（Tmux、InProcess、Remote）的最大公约数：
- Tmux Pane 是独立进程，没有共享内存
- InProcess 在同一进程但使用 AsyncLocalStorage 隔离
- Remote 跨网络，但可以共享网络文件系统

文件邮箱的额外优势：
1. **可观测性**：直接 `cat ~/.claude/teams/*/inboxes/*.json` 就能调试
2. **持久性**：进程崩溃后消息不丢失
3. **简单性**：无需复杂的连接管理、心跳、断线重连
4. **并发安全**：`proper-lockfile` 提供的文件锁足够可靠

代价是**延迟**：500ms 轮询间隔意味着最坏情况下消息传递有 500ms 延迟。但对于 LLM Agent 场景，每个工具调用本身就要数秒，500ms 可以忽略不计。

### 4.5.2 InProcess vs Pane 后端的 tradeoff

| 维度 | InProcess | Tmux Pane |
|------|-----------|-----------|
| **内存** | 共享 V8 堆（低） | 独立进程堆（高） |
| **启动延迟** | ~0ms | ~2-3s（CLI 启动） |
| **隔离** | AsyncLocalStorage（弱） | OS 进程（强） |
| **权限** | Leader UI 桥接（实时） | 邮箱轮询（延迟） |
| **调试** | 共享日志（复杂） | 独立 terminal（直观） |
| **生存期** | 绑定 Leader | 独立 |

InProcess 后端的最大优势是**权限桥接**——通过 `getLeaderToolUseConfirmQueue()`，Worker 的权限对话框直接显示在 Leader 的 terminal 中，带有 Worker badge 标识。这意味着用户不需要切换到 Worker 的 terminal 去 approve 权限。

但 InProcess 有一个根本限制：**Worker 不能生成后台 Agent**（`AgentTool.tsx:277-278`），因为其生命周期绑定在 Leader 进程中，后台 Agent 需要独立的 AbortController。

### 4.5.3 权限始终由人类把控的设计哲学

整个多代理系统的权限设计遵循一个不可妥协的原则：**人类始终是最终权限授予者**。

这个原则在代码中的体现：
1. **子 Agent 不能升级权限**：`runAgent.ts:419` — `bypassPermissions`、`acceptEdits`、`auto` 模式的父级设置始终优先于子 Agent 的 `permissionMode`
2. **Leader 的 permission 不泄漏到 Worker**：`runAgent.ts:467-477` — 当 `allowedTools` 被指定时，清空 session 级别的 allow rules，只保留 CLI 参数级别的 rules
3. **跨机器消息需要显式同意**：`SendMessageTool.ts:checkPermissions` — 发送到 `bridge:` 地址需要 `safetyCheck`，且 `classifierApprovable: false`（安全分类器不能自动 approve）
4. **Plan Mode 审批**：Teammate 可以被设置为 `plan_mode_required`，此时必须先提交 plan 给 Leader approve 才能执行

### 4.5.4 query() 循环复用的递归设计

`runAgent()` 的核心是调用 `query()` 函数——而 `query()` 正是主 REPL 循环使用的同一个函数。这意味着**子 Agent 和主 Agent 使用完全相同的 API 调用和工具执行管线**。

```typescript
// runAgent.ts:748-757 — Agent 的 query() 调用
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```

这个设计的深远影响：
- **工具一致性**：Agent 使用的工具与用户使用的完全相同（只是经过过滤）
- **递归能力**：Agent 的工具池中可以包含 `AgentTool`，因此 Agent 可以生成子 Agent（InProcess Teammate 被允许生成同步子 Agent）
- **Prompt Cache 复用**：Fork 路径通过 `useExactTools` 确保子 Agent 的 API 请求前缀与父 Agent 字节一致，最大化 prompt cache 命中率

但递归也带来了风险——无限递归 fork。解决方案是双重检查（`AgentTool.tsx:331-333`）：
1. `querySource === 'agent:builtin:fork'` — 编译时耐久（context.options 不受 autocompact 影响）
2. `isInForkChild(messages)` — 消息扫描回退

### 4.5.5 与 LangGraph / AutoGen / CrewAI 的对比

| 维度 | Claude Code | LangGraph | AutoGen | CrewAI |
|------|------------|-----------|---------|--------|
| **编排模型** | Leader-Worker (prompt-programmed) | DAG/StateGraph | Agent Chat | Sequential/Hierarchical |
| **通信** | 文件邮箱 + 共享 AppState | State channels | Python function calls | Shared memory |
| **隔离** | 3 级（InProcess/Pane/Remote） | 无 | 无 | 无 |
| **权限** | Human-in-the-loop，始终 | 可选 | 可选 | 无 |
| **持久性** | 磁盘 transcript + TeamFile | 可选 checkpointing | 无 | 无 |
| **工具共享** | 统一 Tool 池 + 过滤 | 每节点独立绑定 | 每 Agent 独立 | 每 Agent 独立 |
| **模型异构** | `model` 参数逐 Agent 指定 | 支持 | 支持 | 支持 |

Claude Code 最大的差异化在于两点：

1. **Coordinator 逻辑是 prompt-programmed 的**——其他框架的编排逻辑是代码硬编码的 DAG 或状态机。Claude Code 的 Coordinator 通过自然语言 prompt 编程，这意味着编排策略可以通过修改 prompt 来调整，不需要改代码。
2. **文件系统作为通信基座**——这看似原始，但提供了跨进程、跨机器的统一通信能力和完全的可观测性。其他框架依赖进程内的 Python function call，在多机场景下需要额外的 RPC 层。

## 4.6 可迁移模式

### 4.6.1 Agent 编排的通用模式

从 Claude Code 的实现中可以提取 5 个通用的 Agent 编排模式：

**模式 1：AsyncGenerator 作为 Agent 接口**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGenerator 提供了拉取式（pull-based）的消息流语义——调用者决定何时消费下一条消息，自然地支持了 background 切换（在 yield 点插入 race）和 progress 追踪。

**模式 2：Foreground → Background 无缝切换**

Claude Code 的 Sync Agent 可以在执行过程中被 background——通过 `Promise.race([nextMessage, backgroundSignal])`。这个模式适用于任何需要"长任务可以中途后台化"的场景。关键是要有一个稳定的 taskId 在 foreground 和 background 之间传递。

**模式 3：文件系统作为 Agent 间通信的"最小公倍数"**

当多种后端（进程内/跨进程/跨机器）需要统一通信时，文件系统是最简单的选择。JSON 文件 + 文件锁提供了足够的一致性保证。

**模式 4：Prompt-Programmed Coordination**

将编排逻辑写在 system prompt 而非代码中，使得协调策略变成了"配置"而非"实现"。这在 Agent 编排快速迭代的阶段特别有价值——改 prompt 的成本远低于改代码。

**模式 5：安全状态转换优先于通知装饰**

gh-20236 的修复模式：在异步流程中，先完成核心状态转换（`completeAsyncAgent`），再执行可能挂起的装饰性操作（classifier check、worktree cleanup）。任何可能阻塞的操作都不应该 gate 关键状态变更。

### 4.6.2 Doramagic FlowController 可借鉴之处

Claude Code 的 Agent 架构与 Doramagic 的 FlowController（lease 系统 + staging/delivery 隔离 + 12 状态机）有几个值得对照的点：

1. **状态机 vs Prompt-Programmed**：Doramagic 用 12 状态机硬编码流程控制，Claude Code 用 prompt 编程 Coordinator。两者各有适用场景——确定性流程用状态机，需要灵活判断的流程用 prompt 编程。

2. **文件邮箱的直接可借鉴性**：Doramagic 的 staging/delivery 目录隔离与 Claude Code 的 `.claude/teams/*/inboxes/` 结构异曲同工。Doramagic 的 FlowController 可以直接采用文件邮箱模式来实现 skill 之间的松耦合通信。

3. **权限模型的借鉴**：Claude Code 的 "子 Agent 不能升级权限" 原则可以映射到 Doramagic 的 skill 权限——被调用的 skill 不应该获得比调用者更高的系统访问权限。

4. **Worktree 隔离思路**：对于 Doramagic 的并行 skill 执行（如多个 soul extractor 并行提取不同项目），可以借鉴 Worktree 的文件系统隔离模式，为每个并行执行创建独立的工作目录。

## 4.7 源码索引

| 文件 | 路径 | 关键导出 |
|------|------|---------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool`（buildTool 定义）、`inputSchema`、`outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | `runAgent()` AsyncGenerator、`filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition` 类型联合、`getAgentDefinitionsWithOverrides()`、`parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`、`resolveAgentTools()`、`finalizeAgentTool()`、`runAsyncAgentLifecycle()`、`classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`、`renderToolResultMessage()`、`renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`、`getCoordinatorSystemPrompt()`、`getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool`（5 路路由）、`handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`、`handleSpawnSplitPane()`、`resolveTeammateModel()`、`buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`、`createInProcessCanUseTool()`、`waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`、`writeToMailbox()`、`markMessageAsReadByIndex()`、所有结构化消息类型 |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`、`createAgentWorktree()`、`removeAgentWorktree()`、`validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | `TaskState` 联合（7 种 task 类型）、`isBackgroundTask()` |

**TaskState 联合类型**（`tasks/types.ts`）：
```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

---

*本章基于 Claude Code TypeScript 源码快照（2026-03-31，~512K LOC）分析完成。所有代码引用均标注具体文件名和行号范围。*
