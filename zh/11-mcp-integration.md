# 第 11 章：MCP 集成

## 11.1 概述与定位

### MCP 是什么

MCP（Model Context Protocol）是 Anthropic 主导设计的开放协议，定义了 AI 应用程序与外部工具服务之间的标准化通信格式。本质上是一个 JSON-RPC 2.0 协议，运行在多种传输层（stdio、SSE、HTTP Streamable、WebSocket）之上，规定了工具发现（`tools/list`）、工具调用（`tools/call`）、资源管理（`resources/list`/`resources/read`）、Prompt 模板（`prompts/list`/`prompts/get`）等标准消息格式。

### Claude Code 中 MCP 的角色

Claude Code 的内置工具集（Bash、Read、Edit 等）覆盖的是文件系统和本地开发场景。MCP 的设计定位是**开放的工具扩展接口**：任何第三方服务（Slack、GitHub、Jira、数据库、浏览器自动化等）均可实现一个 MCP 服务器，Claude Code 通过标准协议连接后即可调用这些外部能力，无需修改核心代码。

在架构上，Claude Code 是一个纯粹的 **MCP 客户端**，不实现任何 MCP 服务器能力（除了响应 `roots/list` 请求以告知服务器工作目录）。每个连接的 MCP 服务器的工具会被动态注册为 `mcp__<serverName>__<toolName>` 格式的 Tool 对象，与内置工具共享同一套执行框架。

### 代码规模

MCP 集成涉及约 12,310 行 TypeScript 代码，分布在以下文件中：

| 文件 | 行数 | 职责 |
|------|------|------|
| `services/mcp/client.ts` | 3,348 | 连接管理、工具发现、执行核心 |
| `services/mcp/config.ts` | 1,578 | 配置管理（多来源合并、策略过滤） |
| `services/mcp/auth.ts` | 2,465 | OAuth 2.0 认证（含 XAA 跨应用访问） |
| `services/mcp/utils.ts` | 575 | 工具过滤、名称哈希、Stale 检测 |
| `services/mcp/types.ts` | 258 | 类型定义（Transport、ServerConfig、连接状态） |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | UI 折叠分类（Search/Read 工具识别） |
| `tools/MCPTool/UI.tsx` | 402 | 工具执行结果渲染 |
| `services/mcp/channelPermissions.ts` | 240 | Channel 权限中继 |
| `services/mcp/channelNotification.ts` | 316 | Channel 消息推入机制 |
| `services/mcp/elicitationHandler.ts` | 313 | Elicitation（表单/URL 交互）处理 |
| `skills/mcpSkillBuilders.ts` | 44 | Skill 构建器注册表（依赖图解耦） |

---

## 11.2 理论基础

### 协议驱动的工具扩展模式

传统的插件系统通常依赖宿主应用提供的 SDK，插件开发者必须了解宿主的内部接口。MCP 采用的是**协议驱动（protocol-driven）**模式：宿主（Claude Code）和插件（MCP 服务器）之间的所有交互通过标准 JSON-RPC 消息完成，双方可以独立演进。

这与 LSP（Language Server Protocol）的设计思路高度一致：

| 维度 | LSP | MCP |
|------|-----|-----|
| 核心模式 | 编辑器 ↔ 语言服务器 | AI Agent ↔ 工具服务器 |
| 发现机制 | `initialize` 中交换 capabilities | `tools/list`、`resources/list`、`prompts/list` |
| 传输层 | stdio、LSP over TCP | stdio、SSE、HTTP Streamable、WebSocket |
| 双向通信 | 支持 | 支持（notifications、elicitation） |
| 版本协商 | 支持 | 支持（`protocolVersion`） |

LSP 解决了"每个编辑器需要对接每种语言"的 M×N 爆炸问题；MCP 解决了"每个 AI 工具需要对接每种外部服务"的同类问题。

### 客户端-服务器协议的设计原则

MCP 的两个关键设计选择对 Claude Code 的实现有深远影响：

**能力协商（Capability Negotiation）**：服务器在连接时通过 `ServerCapabilities` 声明支持的功能子集（`tools`、`prompts`、`resources`、`elicitation`、`experimental`），客户端仅调用服务器已声明的功能。这意味着 Claude Code 不需要为每类服务器写特殊分支，统一通过 `capabilities` 检查决定行为。

**工具注解（Tool Annotations）**：MCP 2025-03 版本引入了 `tool.annotations` 字段，服务器可声明 `readOnlyHint`、`destructiveHint`、`openWorldHint` 等语义标记。Claude Code 直接将这些标记映射到工具的 `isReadOnly()`、`isDestructive()`、`isOpenWorld()` 方法，无需维护工具名称的静态白名单即可做出安全决策。

---

## 11.3 MCP 客户端架构

### MCPClient 类的核心接口

Claude Code 不直接实现 MCP 客户端，而是封装了 `@modelcontextprotocol/sdk` 提供的 `Client` 类。`connectToServer` 是核心入口函数（`client.ts`），使用 `lodash/memoize` 做连接级缓存，缓存键为 `${name}-${jsonStringify(serverRef)}`：

```typescript
// client.ts（约第 540 行）
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... 根据 serverRef.type 初始化 transport
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... 连接、超时、能力协商
  },
  getServerCacheKey,
)
```

### 连接管理（建立、维护、断开）

**建立连接**：`connectToServer` 根据 `serverRef.type` 创建对应 transport，然后发起 `client.connect(transport)`，并设置 30 秒超时（`getConnectionTimeoutMs()`，可通过 `MCP_TIMEOUT` 环境变量覆盖）：

```typescript
// client.ts（约第 1000 行）
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    transport.close().catch(() => {})
    reject(new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
      `MCP server "${name}" connection timed out after ${getConnectionTimeoutMs()}ms`,
      'MCP connection timeout',
    ))
  }, getConnectionTimeoutMs())
  connectPromise.then(() => clearTimeout(timeoutId), () => clearTimeout(timeoutId))
})
await Promise.race([connectPromise, timeoutPromise])
```

**维护连接**：通过覆写 `client.onerror` 和 `client.onclose` 实现错误检测和自动重连。对于远端传输（SSE/HTTP），维护 `consecutiveConnectionErrors` 计数器，连续 3 次终端错误（`ECONNRESET`/`ETIMEDOUT`/`EPIPE` 等）后触发 `closeTransportAndRejectPending`，这会调用 `client.close()` 使所有挂起的 `callTool()` 拒绝，并清除 memoize 缓存，下次请求时自动重连：

```typescript
// client.ts（约第 1250 行）
client.onclose = () => {
  // 清除所有相关缓存，下次调用时触发重连
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Session 过期处理**：HTTP 传输的 MCP 服务器可能返回 HTTP 404 + JSON-RPC 错误码 `-32001`（Session Not Found）。Claude Code 检测这一特定错误模式，触发重连并在 `fetchToolsForClient.call()` 中透明重试（最多 1 次）：

```typescript
// client.ts（约第 150 行）
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**断开连接**：stdio 传输使用三阶段信号升级：先 `SIGINT`（100ms 等待），再 `SIGTERM`（400ms 等待），最后 `SIGKILL`，总断开时间上限 600ms，避免阻塞 CLI 退出。

### 工具动态发现与注册

`fetchToolsForClient`（带 LRU 缓存，容量 20）向服务器发送 `tools/list`，将每个工具包装为符合内部 `Tool` 接口的对象：

- **命名规则**：`mcp__${normalizeNameForMCP(serverName)}__${toolName}`（串联下划线格式）
- **描述截断**：超过 `MAX_MCP_DESCRIPTION_LENGTH = 2048` 字符的 description 被截断并加上 `… [truncated]`，防止 OpenAPI 生成服务器的超长文档污染上下文
- **权限映射**：`tool.annotations.readOnlyHint` → `isReadOnly()`，`tool.annotations.destructiveHint` → `isDestructive()`
- **折叠分类**：调用 `classifyMcpToolForCollapse(serverName, toolName)` 判断是否为 Search/Read 类工具

同样，`fetchCommandsForClient` 发送 `prompts/list`，将 MCP Prompt 转换为 `/命令` 对象；`fetchResourcesForClient` 发送 `resources/list`，为支持资源的服务器注入 `ListMcpResourcesTool` 和 `ReadMcpResourceTool` 工具。

### 消息传输层

Claude Code 支持 6 种传输类型：

| 类型 | 适用场景 | Transport 类 |
|------|----------|-------------|
| `stdio` | 本地子进程（大多数社区服务器） | `StdioClientTransport` |
| `sse` | 远端 SSE 服务器（带 OAuth） | `SSEClientTransport` |
| `sse-ide` | IDE 扩展内部 SSE（无 OAuth） | `SSEClientTransport`（简化配置） |
| `http` | MCP Streamable HTTP（最新规范） | `StreamableHTTPClientTransport` |
| `ws` | WebSocket 传输 | 自定义 `WebSocketTransport` |
| `ws-ide` | IDE 扩展内部 WebSocket | `WebSocketTransport`（带 `X-Claude-Code-Ide-Authorization`） |

特殊场景下，Chrome Extension MCP 服务器和 Computer Use MCP 服务器以**进程内模式（In-Process）**运行，通过 `createLinkedTransportPair()` 建立内存管道，避免 ~325 MB 的子进程开销。

HTTP 传输有一个重要的工程细节：每次 POST 请求都需要携带 `Accept: application/json, text/event-stream` 头（MCP Streamable HTTP 规范要求），Claude Code 通过 `wrapFetchWithTimeout` 统一注入这一头，防止某些运行时环境丢失：

```typescript
// client.ts（约第 460 行）
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// wrapFetchWithTimeout 中：
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 MCP 配置管理

### 服务器配置格式

`types.ts` 使用 Zod 定义了 7 种服务器配置 Schema，通过 `z.union([...])` 聚合为 `McpServerConfigSchema`：

```typescript
// types.ts（第 28-115 行，概要）
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // 向后兼容：无 type 字段 = stdio
  command: z.string().min(1),
  args: z.array(z.string()).default([]),
  env: z.record(z.string(), z.string()).optional(),
}))

export const McpSSEServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('sse'),
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: McpOAuthConfigSchema().optional(),
}))
// + HTTP / WebSocket / SDK / claudeai-proxy ...
```

`ScopedMcpServerConfig` 在基础配置之上增加 `scope`（配置来源）和 `pluginSource`（提供插件的来源标识）字段，供 Channel 权限验证使用。

### 多来源配置合并（enterprise > local > project > user > dynamic）

`getClaudeCodeMcpConfigs`（`config.ts`）实现了多层配置合并，优先级从高到低：

1. **enterprise**（`managed-mcp.json`）：企业独占模式，当此文件存在时屏蔽所有其他来源
2. **local**（项目私有，存储在用户全局配置内，绑定到 CWD）
3. **project**（`.mcp.json`，向上遍历目录树，近端优先）
4. **user**（全局 `~/.claude/config.json` 中的 `mcpServers` 字段）
5. **dynamic**（CLI `--mcp-config` 参数运行时注入）

Project 配置需要额外的**用户审批门控**：首次遇到 `.mcp.json` 中的服务器时，会弹出批准对话框。`getProjectMcpServerStatus()` 读取 `enabledMcpjsonServers`/`disabledMcpjsonServers` 设置，返回 `approved`/`rejected`/`pending`。非交互模式（`-p` 参数、SDK 调用）且 `isSettingSourceEnabled('projectSettings')` 时自动批准。

配置合并后还执行**去重**：Plugin 服务器按"签名"（stdio 服务器用命令数组，远端服务器用 URL）去重，防止同一底层服务器被连接两次；claude.ai Connector 也通过相同机制避免与手动配置重复。

### 环境变量扩展

配置文件中可使用 `${ENV_VAR}` 语法，`expandEnvVarsInString`（`config.ts`/`envExpansion.ts`）在读取配置时展开。未定义的变量会被收集到 `missingVars` 列表并向用户报告。

---

## 11.5 MCP 认证系统

### OAuth 2.0 集成

`ClaudeAuthProvider`（`auth.ts`）实现了 MCP SDK 的 `OAuthClientProvider` 接口，负责整个 OAuth 生命周期。认证流程遵循 RFC 6749 授权码流程 + PKCE（Proof Key for Code Exchange），并通过本地 HTTP 服务器接收回调：

1. **元数据发现**：先探测 RFC 9728（`/.well-known/oauth-protected-resource`），失败后回退到 RFC 8414（`/.well-known/oauth-authorization-server`），最终再尝试路径感知的发现（保持向后兼容）
2. **DCR（动态客户端注册）**：首次认证自动注册 OAuth 客户端，`clientId`/`clientSecret` 存入系统 Keychain
3. **Token 交换**：本地随机端口接收授权码，交换 access_token + refresh_token
4. **Token 刷新**：通过 `checkAndRefreshOAuthTokenIfNeeded()` 在调用前检测过期并刷新，失败时进行智能重试

**Slack 兼容层**：某些 OAuth 服务器（显著如 Slack）对 token 端点返回 HTTP 200 但携带错误体，违反 RFC 6749 预期。Claude Code 通过 `normalizeOAuthErrorBody` 将这类响应重写为 HTTP 400，使 SDK 的错误分类逻辑正常工作：

```typescript
// auth.ts（约第 250 行）
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // 检测是否是 OAuthErrorResponse 伪装成 200
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // 将 Slack 非标准错误码 'invalid_refresh_token' 标准化为 'invalid_grant'
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### 多认证方式支持

除标准 OAuth 外，Claude Code 还支持：

- **Step-Up Auth**：部分操作需要提升权限范围，服务器返回 HTTP 403 并附带新的 scope 要求，Claude Code 检测后重新触发 OAuth 流程
- **XAA（Cross-App Access / SEP-990）**：企业场景下通过统一 IdP（支持 OIDC）一次登录授权多个 MCP 服务器，采用 RFC 8693（Token Exchange）+ RFC 7523（JWT Bearer）复合流程，无需为每个服务器单独弹出浏览器窗口
- **Static Headers**：通过配置文件或 `headersHelper` 脚本注入静态认证头（适用于 API Key 认证）

### Token 管理

Token 数据结构存储在系统安全存储（macOS Keychain / Linux Secret Service）中，键为 `${serverName}|${SHA256(config)[:16]}`，确保同名不同配置的服务器使用独立的 token 槽位。

`auth-cache`（`mcp-needs-auth-cache.json`）记录近期返回 401 的服务器，TTL 15 分钟，避免在每次启动时重复探测必然失败的服务器。缓存读取通过 Promise 共享（`authCachePromise`），防止批量连接时 N 次并发读同一文件。

---

## 11.6 MCP 工具执行

### MCPTool 的执行流程

当 LLM 决定调用 `mcp__slack__send_message` 时，执行流程如下：

1. **路由**：`fetchToolsForClient` 注册的 `call()` 函数被调用，参数为 LLM 生成的 JSON input
2. **重连检查**：`ensureConnectedClient(client)` 检查连接是否仍然有效，必要时重连
3. **进度通知**：通过 `onProgress` 回调发出 `mcp_progress: started` 事件
4. **工具调用**：`callMCPToolWithUrlElicitationRetry`（封装了 `callMCPTool`）向服务器发送 `tools/call` 请求
5. **结果处理**：对图片、大型二进制内容做特殊处理（持久化到磁盘，传递引用），对超大文本内容进行截断
6. **进度通知**：发出 `mcp_progress: completed` 事件（含耗时）

Session 过期的透明重试逻辑：

```typescript
// client.ts（约第 2100 行）
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // 自动重试一次
    }
    throw error
  }
}
```

### classifyForCollapse — 工具结果的上下文折叠分类

`classifyForCollapse.ts` 维护了两个静态 Set：`SEARCH_TOOLS`（约 100 个工具名）和 `READ_TOOLS`（约 300 个工具名），覆盖了 40+ 主流 MCP 服务器（Slack、GitHub、Linear、Datadog、Sentry、Jira、Asana、Gmail、Grafana、PagerDuty 等）。

分类规则：工具名先经过 `normalize()`（camelCase/kebab-case 统一转换为 snake_case），再查找是否在两个 Set 中：

```typescript
// classifyForCollapse.ts（第 587-598 行）
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')
    .replace(/-/g, '_')
    .toLowerCase()
}

export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean } {
  const normalized = normalize(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
  }
}
```

**设计意图**：Search/Read 类工具的结果通常较长，但对后续 LLM 推理的价值有限（检索中间态）。标记后，UI 层可以在对话历史中折叠这些结果，节省可视空间和 context window。注意分类是**保守的**（unknown tools 不折叠），且**仅基于工具名**，不区分服务器名，因为主流服务器的工具名是稳定的跨实例标识符。

### 权限与沙箱控制

MCP 工具执行前调用 `checkPermissions()`，该方法返回 `passthrough` 状态（即始终需要显示权限提示），提示信息中包含建议用户将该工具名添加到 `allow` 规则列表的快捷操作。

工具调用超时由 `MCP_TOOL_TIMEOUT` 环境变量控制，默认 `100_000_000` 毫秒（约 27.8 小时，接近"无限"），允许耗时操作的 MCP 服务器正常完成。

---

## 11.7 MCP Channel 系统

Channel 系统是 MCP 的一个扩展用途：让外部消息平台（Telegram、Discord、iMessage、Slack 等）向进行中的 Claude Code 会话推送消息（feature flag: `KAIROS`/`KAIROS_CHANNELS`，runtime gate: `tengu_harbor`）。

### Channel 权限管理

`channelPermissions.ts` 实现了**权限委托**机制：当 Claude Code 遇到需要用户批准的操作时，可以同时通过 Channel 服务器向用户手机发送提示，用户回复 `yes <5字母ID>` 后，服务器解析并通过 `notifications/claude/channel/permission` 事件通知 Claude Code 批准。

5 字母 ID 使用 25 字符字母表（去掉 `l` 防止与 `1`/`I` 混淆），通过 FNV-1a 哈希生成，包含脏词过滤（`ID_AVOID_SUBSTRINGS` 列表，约 24 个词），确保不在工作消息中出现不当内容：

```typescript
// channelPermissions.ts（第 86-110 行）
export function shortRequestId(toolUseID: string): string {
  let candidate = hashToId(toolUseID)
  for (let salt = 0; salt < 10; salt++) {
    if (!ID_AVOID_SUBSTRINGS.some(bad => candidate.includes(bad))) {
      return candidate
    }
    candidate = hashToId(`${toolUseID}:${salt}`)
  }
  return candidate
}
```

Channel 服务器必须同时声明 `capabilities.experimental['claude/channel']` 和 `capabilities.experimental['claude/channel/permission']` 两个能力，才能成为权限中继方，防止意外开放安全边界。

### Channel 通知机制

`channelNotification.ts` 定义了接收入站消息的完整门控逻辑（`gateChannelServer`），依次检查：

1. 服务器能力声明（`claude/channel`）
2. Runtime 开关（`tengu_harbor`）
3. OAuth 认证（仅支持 claude.ai 账号登录，不支持 API Key）
4. 团队/企业策略（`channelsEnabled: true`）
5. 会话 `--channels` 参数（用户显式声明信任的 Channel）
6. Marketplace 来源验证（防止 `slack@evil` 仿冒 `slack@anthropic`）

入站消息被包装为 `<channel source="serverName" meta_key="value">content</channel>` 格式注入会话队列，`SleepTool` 轮询（约 1 秒间隔）唤醒后模型可决定如何回应。

### Elicitation 处理

`elicitationHandler.ts` 处理服务器主动发起的交互请求（MCP Elicitation 规范）。支持两种模式：

- **form 模式**：服务器请求用户填写表单（`requestedSchema` 字段定义 JSON Schema）
- **url 模式**：服务器请求用户访问 URL 完成操作（如 OAuth 授权）

处理流程：先运行 Hook 系统（可编程式响应），若 Hook 无响应则将请求入队至 `AppState.elicitation.queue`，等待 UI 渲染表单或打开浏览器，用户操作后 `respond()` 回调触发响应：

```typescript
// elicitationHandler.ts（第 69-90 行）
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. 先尝试 Hook（程序化响应）
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. 显示 UI，等待用户响应
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

URL 模式还支持 `ElicitationCompleteNotificationSchema`：服务器完成操作后主动通知 Claude Code，对应队列项标记 `completed: true`，UI 据此更新显示状态。

---

## 11.8 MCP Skill 构建

`skills/mcpSkillBuilders.ts` 是一个极简的依赖图解耦模块（44 行），解决了循环依赖问题：

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts  (循环)
```

解决方案是引入写一次的注册表（Write-Once Registry）：

```typescript
// mcpSkillBuilders.ts（全文）
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) throw new Error('MCP skill builders not registered')
  return builders
}
```

`loadSkillsDir.ts` 模块初始化时调用 `registerMCPSkillBuilders()`，这在应用启动时（通过 `commands.ts` 的静态 import）提前完成，保证 MCP 服务器连接时注册表已就绪。

Skill 发现机制（Feature Flag: `MCP_SKILLS`）：`fetchMcpSkillsForClient` 读取服务器的 `skill://` 资源，将 Markdown 格式的 skill 文件（含 frontmatter 元数据）解析为 Skill Command，注册为 `/serverName:skillName` 格式的 slash 命令，实现 MCP 服务器自动提供可复用的 AI 工作流。

---

## 11.9 设计决策分析

### 为什么采用 MCP 而非自定义插件协议

**问题**：Claude Code 需要支持大量第三方工具集成，自定义插件 API 会产生生态锁定。

**决策**：采用 MCP 开放标准，与 MCP 社区生态共享。

**理由**：
- 2025 年初 MCP 生态已有数百个开源服务器可以直接复用
- 插件开发者可以用任何语言实现 MCP 服务器（Python、Go、Java 等），不受 TypeScript 生态约束
- Anthropic 既是 Claude Code 的开发者，也是 MCP 规范的主导者，可以在规范层面解决需求而非在客户端打补丁

**代价**：协议复杂性（认证、传输层差异、版本兼容）全部由客户端承担，`auth.ts` 的 2,465 行很大程度上是处理 OAuth 规范缺陷和各厂商不合规实现的结果。

### MCP 认证的复杂性处理

**问题**：MCP 规范对认证的描述是"建议性"的，实际服务器实现千差万别（Slack 用 200 返回错误、某些服务器不实现 Token Revocation 等）。

**决策**：在 `auth.ts` 建立完整的兼容层，处理已知的不合规行为。

关键策略：
- `normalizeOAuthErrorBody`：处理 200 伪装成错误的响应
- `NONSTANDARD_INVALID_GRANT_ALIASES`：标准化 Slack 等的非标准错误码
- RFC 7009 revocation 的双重尝试（先标准方式，收到 401 再用 Bearer 重试）
- 两套 auth 发现路径（RFC 9728 → RFC 8414 → 路径感知回退）

### 工具折叠分类的设计考量

**问题**：MCP 工具结果可能极长（检索结果、日志输出），但大量内联在对话历史中会降低可读性并浪费 context window。

**决策**：采用显式的工具名白名单而非启发式分类，对已知的 Search/Read 类工具结果在 UI 层默认折叠。

**权衡**：
- 优点：确定性强，无误判，已知工具的行为一致
- 缺点：需要维护静态列表（`classifyForCollapse.ts` 当前覆盖 40+ 服务器），新增服务器需要手动更新
- 保守策略（未知工具不折叠）确保新服务器不会因错误折叠而丢失信息

---

## 11.10 可迁移模式

以下模式来自 Claude Code MCP 集成的工程实践，适用于其他需要集成外部工具或服务的系统：

**1. 能力协商优先于类型判断**
在实现协议客户端时，始终通过 `capabilities` 检查决定行为，而非通过服务器类型或名称做 if-else 分支。这使得新能力的支持可以增量添加，不影响现有逻辑。

**2. Memoize + Cache Invalidation 模式**
连接、工具发现等昂贵操作使用 memoize 缓存，但缓存必须在连接断开时立即失效（`client.onclose` 中清除所有相关缓存条目）。使用 LRU 缓存（容量 20）防止内存泄漏。

**3. 写一次注册表解决循环依赖**
当模块 A 需要依赖模块 B 的函数，而模块 B 又间接依赖模块 A 时，引入一个零外部依赖的注册表模块，在应用初始化时由模块 B 向注册表注入实现，模块 A 从注册表读取。`mcpSkillBuilders.ts` 是最小可复制的模板。

**4. 协议兼容层集中管理**
OAuth/HTTP 规范的不合规实现应集中在一个位置（如 `auth.ts`）处理，而非散布在调用处。`normalizeOAuthErrorBody` 是这一模式的典型示例：一个纯函数，在传输层统一处理后，调用处无需关心服务器是否规范。

**5. 并发连接的分层限速**
不同类型的操作对资源的消耗不同。stdio 服务器需要 fork 子进程（CPU + 内存密集），网络服务器只需建立 TCP 连接（I/O 密集）。对两类操作使用不同的并发限制（local: 3，remote: 20）可以在保护系统的同时最大化吞吐。

**6. needs-auth 状态的双重检查**
对于需要认证的远端服务，结合**基于时间的缓存**（15 分钟 TTL）和**基于状态的检查**（有 discovery 状态但无 token）双重判断跳过不可能成功的连接，避免每次启动时的无效探测延迟。

---

## 11.11 源码索引

| 关键实现 | 文件:行号 |
|---------|---------|
| `MCPServerConnection` 联合类型定义 | `services/mcp/types.ts:170-200` |
| `ConfigScope` 枚举（7个来源） | `services/mcp/types.ts:10-22` |
| `connectToServer` 主函数（memoized） | `services/mcp/client.ts:540` |
| Transport 初始化分支（6种类型） | `services/mcp/client.ts:570-930` |
| 连接超时处理 | `services/mcp/client.ts:1000-1040` |
| 断连错误检测与重连触发 | `services/mcp/client.ts:1200-1320` |
| stdio 三阶段关闭（SIGINT/SIGTERM/SIGKILL） | `services/mcp/client.ts:1370-1490` |
| `fetchToolsForClient`（工具注册） | `services/mcp/client.ts:1830-2050` |
| `getMcpToolsCommandsAndResources`（批量连接入口） | `services/mcp/client.ts:2580` |
| `isMcpSessionExpiredError` | `services/mcp/client.ts:150-165` |
| `wrapFetchWithTimeout`（HTTP Accept 注入） | `services/mcp/client.ts:450-510` |
| 配置来源合并 `getClaudeCodeMcpConfigs` | `services/mcp/config.ts:1050` |
| 企业独占模式判断 | `services/mcp/config.ts:1080-1090` |
| Project 服务器审批门控 | `services/mcp/utils.ts:210-250` |
| 插件服务器去重（签名哈希） | `services/mcp/config.ts:215-270` |
| `ClaudeAuthProvider`（OAuth 核心） | `services/mcp/auth.ts:500+` |
| `normalizeOAuthErrorBody`（Slack 兼容） | `services/mcp/auth.ts:250-290` |
| `performMCPXaaAuth`（跨应用认证） | `services/mcp/auth.ts:700+` |
| `getServerKey`（Token 存储键生成） | `services/mcp/auth.ts:390-405` |
| `hasMcpDiscoveryButNoToken`（快速失败） | `services/mcp/auth.ts:420-435` |
| `classifyMcpToolForCollapse`（折叠分类） | `tools/MCPTool/classifyForCollapse.ts:587-598` |
| `SEARCH_TOOLS` / `READ_TOOLS` 白名单 | `tools/MCPTool/classifyForCollapse.ts:20-585` |
| `shortRequestId`（Channel 权限 ID）| `services/mcp/channelPermissions.ts:140-160` |
| `gateChannelServer`（6层门控）| `services/mcp/channelNotification.ts:190-310` |
| `registerElicitationHandler` | `services/mcp/elicitationHandler.ts:65-150` |
| `registerMCPSkillBuilders`（循环依赖解耦） | `skills/mcpSkillBuilders.ts:30-44` |
