# 第 1 章：架构总览与启动流程

> 数据源：Claude Code TypeScript 源码快照（2026-03-31）
> 源码路径（mini）：`~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 概述与定位

**Claude Code 是什么：** Claude Code 是一个运行在终端中的 AI 编程助手，通过 React/Ink 渲染交互式 TUI（Terminal User Interface），以 REPL 循环驱动 Claude API 完成代码编辑、命令执行、文件操作等开发任务。

### 技术栈概览

| 层次 | 技术 | 用途 |
|------|------|------|
| 运行时 | Bun（主要）/ Node.js 18+（兼容） | JavaScript 运行环境 |
| 语言 | TypeScript | 全项目严格类型 |
| UI 框架 | React + Ink | 终端 TUI 渲染 |
| CLI 框架 | Commander.js（`@commander-js/extra-typings`） | 命令行参数解析 |
| API 客户端 | `@anthropic-ai/sdk` | Claude API 调用 |
| MCP 集成 | `@modelcontextprotocol/sdk` | MCP 服务器协议 |
| 功能开关 | GrowthBook + `bun:bundle` feature flags | A/B 测试与 DCE |
| 遥测 | OpenTelemetry（懒加载 ~400KB） | 指标/日志/追踪 |
| 校验 | Zod v4 | 运行时 schema 验证 |

### 代码规模统计

- **总行数**：512,664 行（`.ts` + `.tsx` 文件）
- **文件数**：1,884 个 TypeScript 文件
- **顶层目录数**：35 个

各主要目录 LOC 占比：

```
utils/       180,472 行  (35.2%)  — 工具函数、权限、认证、设置等
components/   81,546 行  (15.9%)  — React UI 组件
services/     53,680 行  (10.5%)  — API、MCP、分析、内存等服务
tools/        50,828 行  (9.9%)   — 30 个工具实现（Bash/File/Agent 等）
commands/     26,428 行  (5.2%)   — slash 命令实现
screens/       5,977 行  (1.2%)   — REPL 等顶层屏幕
bootstrap/     ~5,000 行  (1.0%)  — 全局状态（state.ts 1,758 行）
entrypoints/   ~3,000 行  (0.6%)  — CLI/SDK/MCP 入口
main.tsx       4,683 行  (0.9%)   — 主入口协调器
setup.ts         477 行  (0.1%)   — 初始化设置
```

---

## 1.2 理论基础

### 命令行应用的架构模式

Claude Code 融合了两种经典 CLI 架构模式：

**REPL Loop（Read-Eval-Print Loop）**
传统 REPL 在一个同步循环中读取输入、求值、打印输出。Claude Code 将其升级为异步事件驱动 REPL：输入由 React 组件捕获，"求值"是一次 Claude API round-trip（包含多轮工具调用），输出通过 React/Ink reconciler 渲染到终端。

**Event-Driven Architecture**
启动时不阻塞等待所有初始化完成——MDM 读取、Keychain 预取、MCP 连接、plugin hook 加载全部以 fire-and-forget 方式并行触发（详见 1.4 节）。这使得 TTFR（Time To First Render）最小化，与 Web 应用的 Critical Rendering Path 优化思路一致。

### 终端 UI 框架的设计哲学：React in Terminal

Ink 将 React 的组件模型、声明式状态、reconciliation 机制移植到终端。核心思路：

- **虚拟 DOM → 虚拟终端缓冲区**：每次 state 变化触发 diff，只重绘变化的字符行，避免闪烁
- **Flexbox → 终端布局**：用 CSS Yoga 引擎计算列宽、换行，使得终端 UI 可以用 JSX 声明式描述
- **组件复用**：Loading spinner、确认弹窗、Diff 显示等 UI 逻辑封装为可测试的 React 组件

这使得 Claude Code 的 UI 代码与 Web 前端代码共享认知框架，`components/` 目录下 81,546 行代码可以用熟悉的 React 模式理解。

### 插件化架构的理论基础

Claude Code 的插件系统基于"能力注册"模型（Capability Registration Pattern）：

- 工具（Tools）、命令（Commands）、Hooks 均在启动时注册到全局注册表
- 插件通过文件系统约定（`~/.claude/plugins/`）扩展工具/命令列表
- `bun:bundle` 的 `feature()` 函数在编译时做 Dead Code Elimination（DCE），实验性功能不会出现在外部构建产物中

---

## 1.3 整体架构图

### 分层架构（ASCII）

```
┌─────────────────────────────────────────────────────────┐
│                    入口层 (Entry Layer)                   │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (CLI交互)    (Commander.js路由)     (MCP服务器模式)       │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  引导层 (Bootstrap Layer)                 │
│    setup.ts      │    entrypoints/init.ts                 │
│  (会话初始化)      │    bootstrap/state.ts                 │
│  (worktree/tmux)  │    (全局状态单例)                      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  UI 层 (Ink/React TUI)                    │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (主交互界面)       │  components/ (81K LOC)               │
│  replLauncher.tsx  │  (输入/输出/弹窗/等待动画)              │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  引擎层 (Engine Layer)                    │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (会话生命周期管理)  │  (API调用)  │  (React状态树)          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  工具层 (Tool Layer)                      │
│  tools/ (30个工具，50K LOC)                               │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  服务层 (Service Layer)                   │
│  services/ (53K LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (MCP客户端)     (GrowthBook/OTel)          │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (语言服务器)   (会话记忆)       (企业托管配置)               │
└─────────────────────────────────────────────────────────┘
```

### 模块依赖关系概览

```
main.tsx
  ├── entrypoints/init.ts       (memoized，只初始化一次)
  ├── entrypoints/cli.tsx       (Commander 子命令路由)
  ├── bootstrap/state.ts        (全局状态，严禁循环依赖)
  ├── setup.ts                  (每次会话调用)
  ├── QueryEngine.ts            (headless/SDK 路径)
  ├── replLauncher.tsx          (interactive 路径)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (MCP 工具/资源加载)
```

**bootstrap/state.ts 的特殊地位**：代码中有显式注释 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`，且有 ESLint 规则 `custom-rules/bootstrap-isolation` 阻止该文件被非叶子模块导入，防止循环依赖。

### 三种入口点对比

| 入口 | 文件 | 触发方式 | 特点 |
|------|------|----------|------|
| CLI 交互 | `entrypoints/cli.tsx` | `claude` 命令 | 完整 REPL + React TUI |
| SDK 无头 | `QueryEngine.ts` | `-p` flag / SDK API | 无 UI，单次或流式输出 |
| MCP 服务器 | `entrypoints/mcp.ts` | `claude --mcp` | 暴露工具集为 MCP server |

---

## 1.4 启动流程详解

### main.tsx 完整启动序列

`main.tsx` 的 4,683 行并非顺序执行——文件顶部的 import 副作用是精心编排的并行预热序列。

**阶段 0：模块加载期（import 副作用，~135ms）**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. 性能基准起点

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. 并行：MDM 子进程（plutil/reg query）

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. 并行：macOS Keychain 预读（OAuth + API key）

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // 所有 import 完成
```

注释精确说明了这三个并行操作的收益：MDM 读取节省 ~135ms 模块求值时间，Keychain 预读节省 ~65ms 的顺序 sync spawn。这是 Claude Code 启动优化的核心技巧：**利用 ES module 的静态分析特性，在模块图求值期间偷跑 I/O 密集型操作**。

**阶段 1：Commander 路由（同步）**

`entrypoints/cli.tsx` 中 Commander.js 解析 argv，根据子命令（`chat`、`api`、`mcp`、`resume` 等）或 flag 分发到不同执行路径：

```typescript
// entrypoints/cli.tsx（简化结构）
async function main(): Promise<void> {
  // 快速路径：--version 零 import
  // 常规路径：await init() → setup() → 分支执行
}
```

**阶段 2：init() 初始化（memoized，只执行一次）**

`entrypoints/init.ts` 中的 `init` 函数用 `memoize` 包裹，确保多次调用只初始化一次：

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // 配置系统激活
  applySafeConfigEnvironmentVariables()  // 信任对话前的安全 env vars
  applyExtraCACertsFromConfig()     // 早于任何 TLS 连接设置 CA 证书
  setupGracefulShutdown()           // 注册退出清理钩子
  // 懒加载：OpenTelemetry（~400KB）+ gRPC（~700KB）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // 异步缓存
  detectCurrentRepository()          // GitHub repo 检测
  preconnectAnthropicApi()           // TCP+TLS 预连接（~100-200ms overlap）
  configureGlobalMTLS()
  configureGlobalAgents()            // proxy 配置
})
```

**阶段 3：setup() 会话初始化（每次会话调用）**

```typescript
// setup.ts — 关键步骤序列
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. UDS messaging server（swarm/ant 模式）
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Terminal 备份检查（iTerm2/Terminal.app）
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — 必须在所有依赖 cwd 的代码之前
  setCwd(cwd)
  // 4. Hooks 配置快照（必须在 setCwd 之后）
  captureHooksConfigSnapshot()
  // 5. Worktree 创建（如果 --worktree）
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. 后台任务注册（SessionMemory、context collapse）
  if (!isBareMode()) initSessionMemory()
  // 7. Plugin 预取（并行，非阻塞）
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. 分析 sink 激活 + 第一个遥测事件
  initSinks()
  logEvent('tengu_started', {})
  // 9. Release notes 检查（交互模式）
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**阶段 4：REPL 渲染**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // 懒加载 UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

最终 Ink 接管终端，React 组件树开始渲染，REPL ready。

### 并行预取策略

Claude Code 的启动优化遵循"**越早触发，越晚等待**"原则：

| 操作 | 触发时机 | 等待时机 |
|------|----------|----------|
| MDM 子进程 (`plutil/reg query`) | `main.tsx` 第1行 import 副作用 | `applySafeConfigEnvironmentVariables()` 调用前 |
| Keychain 预读 (OAuth + API key) | `main.tsx` 第3行 import 副作用 | `ensureKeychainPrefetchCompleted()` |
| Claude API TCP 预连接 | `init()` 内 `preconnectAnthropicApi()` | 第一次 API 请求时自动复用连接 |
| Plugin hooks 加载 | `setup()` 内 fire-and-forget | `processSessionStartHooks()` 渲染前 |
| MCP configs 读取 | `getClaudeCodeMcpConfigs()` kick-off | 交互模式中 `getMcpToolsCommandsAndResources()` |

### 懒加载机制

Claude Code 对启动关键路径上的大模块做了显式懒加载：

```typescript
// entrypoints/init.ts — OpenTelemetry 懒加载注释
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

此外，`replLauncher.tsx` 在最后一刻才 `import` App 和 REPL 组件，避免 React 树在 Commander 路由完成前就被求值。

`bun:bundle` 的 `feature()` 函数实现编译时 DCE：

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

外部构建中这些代码被完全剔除，减少包体积。

### setup.ts 初始化步骤详解

`setup.ts` 的 477 行围绕以下关键约束展开：

1. **`setCwd()` 必须最先调用**：所有后续操作（hooks、settings、plugin 加载）都依赖正确的 cwd
2. **Hooks 快照必须在 `setCwd()` 之后**：确保从正确目录读取 `.claude/settings.json`
3. **Worktree 创建在 `getCommands()` 之前**：否则 `/eject` 命令不可用
4. **`initSinks()` 在所有后台任务注册之后**：确保分析事件队列已就绪

`--bare` 模式（scripted/SDK 无头调用）跳过了大量互动功能：terminal 备份检查、plugin hook 预取、commit attribution、team memory watcher 等，使脚本调用的启动开销最小化。

### bootstrap/state.ts 状态构建

`state.ts`（1,758 行）维护着整个会话的全局单例状态。核心 `State` 类型覆盖：

```typescript
// bootstrap/state.ts（State 类型定义，部分）
type State = {
  originalCwd: string
  projectRoot: string          // 稳定的项目根目录，worktree 不改变它
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // 遥测计数器
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // 日志/追踪提供器
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... 共约60个字段
}
```

**设计约束**：注释 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` 是一个架构警卫。ESLint 规则 `custom-rules/bootstrap-isolation` 防止 state.ts 被会导致循环依赖的模块导入。所有状态通过 setter/getter 函数访问，不直接暴露可变对象。

---

## 1.5 入口点分析

### CLI 入口（interactive mode）

`entrypoints/cli.tsx` 是最复杂的入口，承担所有用户面向的功能路由：

**启动路径**：
1. Commander.js 解析 argv → 识别子命令或 flag
2. `await init()` 初始化（memoized）
3. 处理 MCP configs、enterprise policy、Chrome 集成
4. `await setup(cwd, permissionMode, ...)` 会话初始化
5. 根据模式分支：
   - **交互模式**：`showSetupScreens()` → `launchRepl()` → React TUI
   - **Print 模式（`-p`）**：`runHeadless()` → `QueryEngine` → stdout
   - **Resume 模式**：`loadConversationForResume()` → 恢复历史会话
   - **Teleport 模式**：远程会话接管

**关键 CLI options**（部分）：

| Flag | 功能 |
|------|------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | 动态 MCP 服务器配置 |
| `--worktree` | 创建 git worktree 隔离 |
| `--tmux` | 在 tmux session 中运行 |
| `--model` | 覆盖主循环模型 |
| `--resume` | 恢复历史会话 |

### SDK 入口（programmatic API）

当通过 `-p` flag 或 SDK 编程 API 调用时，绕过 React TUI，直接进入 `QueryEngine.ts`：

- `isNonInteractiveSession = true`
- 跳过所有 UI 渲染（Ink）
- 通过 `SDKMessage` 类型的流式输出到 stdout
- 支持 `SDKStatus`、`SDKPermissionDenial`、`SDKCompactBoundaryMessage` 等结构化输出

SDK 模式还有专属 beta features：`entrypoints/sdk/coreSchemas.ts` 定义了结构化 JSON 输入/输出 schema，`entrypoints/agentSdkTypes.ts` 定义了 `HookEvent`、`ModelUsage` 等 SDK 专属类型。

### MCP 入口（MCP server mode）

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools：将 Claude Code 的所有工具暴露为 MCP tools
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool：代理执行到对应 Tool 实现
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

MCP 模式将 Claude Code 的整个工具集（BashTool、FileReadTool、GrepTool 等）反向暴露给外部 MCP 客户端，实现"Claude Code as MCP server"。

### 三种入口的共享逻辑

无论哪种入口，都共享：
- `bootstrap/state.ts` 全局状态
- `entrypoints/init.ts` 初始化（memoized 保证只执行一次）
- `Tool.ts` 工具注册表
- `services/` 下的所有服务（API 客户端、权限系统等）
- Hooks 生命周期系统

差异在于是否渲染 React TUI 以及输出格式（交互文本 vs. 结构化 JSON）。

---

## 1.6 设计决策分析

### 为什么选择 Bun 而非 Node.js

从代码中可以观察到 Bun 的使用特征：

1. **`bun:bundle` 的 `feature()` 函数**：这是 Bun 特有的编译时 feature flag 机制，支持 Dead Code Elimination。在 `main.tsx` 中大量使用（COORDINATOR_MODE、KAIROS、CHICAGO_MCP、UDS_INBOX 等），外部构建会完全剔除这些实验代码。

2. **Bun 的 WebView API**（条件引用）：`typeof Bun !== 'undefined' && 'WebView' in Bun`，说明部分功能依赖 Bun 特有 API。

3. **Bun 的 single-file executable**：注释中提到 `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`，说明发布产物是 Bun 编译的单文件可执行文件。

4. **性能**：Bun 的启动速度和模块加载速度显著优于 Node.js，对 CLI 工具的 TTFR 至关重要。

同时保留 Node.js 18+ 兼容性（`setup.ts` 中有 Node 版本检查），是为了支持非 Bun 环境（CI、企业管控机器）。

### 为什么用 React/Ink 做终端 UI

`components/` 目录 81,546 行代码说明 UI 复杂度极高。如果用原始 ANSI 控制码手写，维护成本将难以控制。React/Ink 的选择带来：

1. **声明式 UI**：流式输出、工具执行状态、权限确认弹窗等都可以用 React state 驱动，而非命令式光标控制
2. **组件隔离**：`screens/REPL.tsx` 只需关心整体布局，各子功能（输入框、消息列表、工具进度）各自封装
3. **热重载友好**：开发时可以用标准 React DevTools 思路调试
4. **测试性**：React 组件可以用 `@testing-library/react` 写单元测试，不依赖真实终端

### 并行预取的性能优化思路

Claude Code 的启动优化有一个清晰的优先级模型：**TTFR（Time To First Render）最优先，而非"所有初始化完成"**。

具体体现：
- Keychain 读取（~65ms）在第一行 import 副作用就触发，而非等到需要 API key 时
- MCP servers 的连接在后台并行，REPL 渲染不等待（用户看到界面后 MCP 才连接完成）
- Release notes、GrowthBook 配置、plugin hooks 全部 fire-and-forget

代价是需要仔细管理"预取完成前被消费"的 race condition，通过 `ensureKeychainPrefetchCompleted()` 等 await 点精确控制。

### 懒加载 vs. 预加载的 tradeoff

| 策略 | 对象 | 理由 |
|------|------|------|
| 预加载（import 副作用） | MDM 子进程、Keychain | I/O 密集，越早越好 |
| 懒加载（`await import()`） | OpenTelemetry（~400KB）、gRPC（~700KB）、React TUI 组件 | 模块求值昂贵，不在关键路径 |
| 条件加载（`feature()` DCE） | COORDINATOR_MODE、KAIROS、CHICAGO_MCP | 实验功能，外部用户不需要 |
| `setImmediate()` 延迟 | commit attribution hook | 避免在 setup() 微任务窗口阻塞 event loop |

这种分层策略让 Claude Code 在启动时只做"显示界面所需的最小工作"。

---

## 1.7 可迁移模式

### 启动优化的通用模式

Claude Code 的启动序列展示了一个可复用的"**并行预热 + 懒加载 + DCE**"三层优化框架：

**Pattern 1：利用 ES module 副作用做 I/O 预热**
```typescript
// 在 import 语句之间插入 fire-and-forget I/O
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // 立即触发，不 await
import { SomethingElse } from './other.js'  // 并行加载
```
适用于：任何有"必须读但读取慢"的初始化数据（配置文件、凭证、网络预连接）。

**Pattern 2：memoize 单次初始化**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
适用于：多个入口点共享的初始化逻辑，防止重复执行。

**Pattern 3：`--bare` 模式分层**
scripted/API 调用不需要 UI、terminal 检查、analytics 等，用 `isBareMode()` 快速跳过，保持 headless 调用的低开销。

**Pattern 4：状态分离**
`bootstrap/state.ts` 作为严格的叶子模块（无循环依赖），通过 setter/getter 访问，配合 ESLint 规则强制执行。这使得状态模块可以在任意地方安全 import。

### Doramagic CLI 可借鉴之处

基于以上分析，Doramagic CLI 在架构设计上可以采用以下模式：

1. **分离启动关键路径**：将"必须在渲染前完成"和"可以在渲染后完成"的初始化严格分开，用注释标注理由（参考 Claude Code 的 `// ~65ms on every macOS startup` 注释风格）

2. **全局状态单例 + 访问器模式**：参考 `bootstrap/state.ts`，用一个严格的叶子模块维护会话状态，避免状态散落各处

3. **`memoize` 初始化函数**：无论从哪个入口调用，保证初始化只执行一次

4. **三种模式分离**：interactive（React TUI）/ headless（-p flag）/ server（MCP），共享底层工具和服务层

5. **feature flag + DCE**：实验性功能用 feature flag 包裹，发布时自动剔除

---

## 1.8 源码索引

| 文件 | 行数 | 关键内容 |
|------|------|----------|
| `main.tsx` | 4,683 | 主入口、Commander 路由、状态初始化、交互/headless 分支 |
| `setup.ts` | 477 | 会话初始化：cwd、hooks、worktree、plugin 预取 |
| `bootstrap/state.ts` | 1,758 | 全局状态单例，`State` 类型定义，所有 getter/setter |
| `entrypoints/init.ts` | ~400 | memoized 全局初始化：config、mTLS、proxy、OTel 懒加载 |
| `entrypoints/cli.tsx` | ~2,000 | Commander.js 路由，交互/print/resume/teleport 分支 |
| `entrypoints/mcp.ts` | ~200 | MCP server 模式，暴露工具集 |
| `entrypoints/sdk/coreSchemas.ts` | - | SDK 模式结构化输入/输出 schema |
| `entrypoints/agentSdkTypes.ts` | - | SDK 专属类型（HookEvent、ModelUsage 等） |
| `replLauncher.tsx` | ~30 | 懒加载 App + REPL，启动 React TUI |
| `QueryEngine.ts` | ~1,500 | 会话生命周期管理，headless 路径核心 |
| `Tool.ts` | - | 工具接口定义（inputSchema、call、prompt 等） |
| `tools/` | 50,828 | 30 个工具实现（BashTool/FileEditTool/AgentTool 等）|
| `services/api/` | - | Claude API 调用、重试、usage 统计 |
| `services/mcp/client.ts` | - | MCP 客户端连接管理 |
| `utils/startupProfiler.ts` | - | `profileCheckpoint()` 性能埋点 |
| `utils/secureStorage/keychainPrefetch.ts` | - | macOS Keychain 并行预读 |
| `utils/settings/mdm/rawRead.ts` | - | MDM 配置并行读取 |

### 关键代码定位

- **并行预热起点**：`main.tsx:12-20`（3 个 import 副作用）
- **memoized 初始化**：`entrypoints/init.ts:57`（`export const init = memoize(...)`）
- **全局状态类型**：`bootstrap/state.ts:30-200`（`type State = {...}`）
- **MCP server 定义**：`entrypoints/mcp.ts:42`（`startMCPServer`）
- **REPL 渲染入口**：`replLauncher.tsx:14`（`launchRepl`）
- **工具接口**：`Tool.ts:1-30`（`ToolInputJSONSchema`、`ToolUseContext`）
- **setup 关键顺序**：`setup.ts:77-230`（setCwd → captureHooksConfigSnapshot → worktree → background jobs）

---

*章节字数：约 9,800 字符 | 源码快照日期：2026-03-31*
