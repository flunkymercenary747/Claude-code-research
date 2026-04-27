# 第 14 章：Feature Flags 与可观测性

## 14.1 概述与定位

Claude Code 的可观测性体系是一个多层次、多目标的系统，覆盖从编译期特性裁剪到运行期行为追踪的全过程。整个体系由三大支柱构成：

1. **Feature Flag 系统**：双轨制设计——编译时 `feature()` 调用（通过 `bun:bundle` 实现 Dead Code Elimination）与运行时 GrowthBook 动态配置，前者控制发布给不同用户群的功能边界，后者支持不重新发布即可调整功能开关。

2. **可观测性管道**：基于 OpenTelemetry 标准，支持 gRPC/HTTP/Protobuf 三种导出协议，统一采集 Metrics、Logs、Traces 三类信号，并通过 Perfetto 追踪格式提供内部调试层。

3. **Analytics 采集**：双路路由——Datadog（外部监控）+ First-Party 1P 事件日志（内部 BigQuery/proto），通过事件名称前缀 `tengu_*` 标识所有业务事件，利用 GrowthBook 动态采样配置控制数据量。

这套体系的核心设计原则是**分层隔离**：用户隐私优先（默认不记录代码内容和文件路径）、内外部构建差异（ant-only vs external）、graceful degradation（每一层都有 kill-switch）。

---

## 14.2 理论基础

### Feature Flag 驱动开发

Feature Flag（功能标志）使团队能够在同一代码库中并行开发不同阶段的功能，并按需启用。Claude Code 采用了两层 flag 机制：

- **编译时 flag**：通过 `bun:bundle` 提供的 `feature()` 调用，在打包时进行 Dead Code Elimination。外部版本中不存在的整块代码会被完整删除，不仅减少包体积，还防止内部逻辑被逆向。
- **运行时 flag**：通过 GrowthBook SDK 从服务端动态获取，支持 A/B 测试、灰度发布、紧急 kill-switch 等场景。

### 可观测性的三大支柱

OpenTelemetry 社区将可观测性定义为三大信号（Three Pillars of Observability）：

- **Metrics（指标）**：时序数值型数据，如 API 延迟、token 消耗量。Claude Code 使用 `@opentelemetry/sdk-metrics` 通过 PeriodicExportingMetricReader 每 60 秒导出一次。
- **Logs（日志）**：结构化事件记录。所有 `logEvent()` 调用最终通过 OTel `LoggerProvider` + `BatchLogRecordProcessor` 批量导出。
- **Traces（追踪）**：分布式调用链。Claude Code 通过 `sessionTracing.ts` 建立 Interaction → LLM Request → Tool Call 的层次化 Span 树，支持多 Agent 场景下的父子关系追踪。

### A/B 测试在 CLI 工具中的应用

与 Web 产品不同，CLI 工具的 A/B 测试面临独特挑战：无浏览器指纹、多平台多发行渠道、离线运行场景。Claude Code 的应对策略：

- 用户维度靶向：`GrowthBookUserAttributes` 携带 `platform`、`subscriptionType`、`rateLimitTier` 等属性，支持分层实验。
- 本地磁盘缓存：每次成功从服务端获取特性值后写入 `~/.claude/config.json` 中的 `cachedGrowthBookFeatures`，确保离线时也能使用上次已知值。
- 曝光去重：同一 session 内每个 feature 的实验曝光事件只记录一次（`loggedExposures` Set）。

---

## 14.3 Feature Flag 系统

### GrowthBook 集成

GrowthBook 是一个开源的 Feature Flag 和 A/B 测试平台。Claude Code 通过官方 `@growthbook/growthbook` SDK 集成，文件位于 `src/services/analytics/growthbook.ts`（1155 行）。

**初始化流程**：

```typescript
// growthbook.ts:529-600（简化）
export const initializeGrowthBook = memoize(
  async (): Promise<GrowthBook | null> => {
    let clientWrapper = getGrowthBookClient()
    // ...
    await clientWrapper.initialized
    setupPeriodicGrowthBookRefresh()
    return clientWrapper.client
  },
)
```

关键设计：`memoize` 确保整个进程生命周期内只初始化一次 GrowthBook 客户端；auth 变化（登录/登出）时通过 `refreshGrowthBookAfterAuthChange()` 销毁并重建客户端，而不是尝试更新 `apiHostRequestHeaders`（SDK 不支持初始化后更新）。

**用户属性模型**（`growthbook.ts:31-46`）：

```typescript
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

**刷新策略**：
- 外部用户：每 6 小时刷新一次（`6 * 60 * 60 * 1000`）
- 内部员工（ant）：每 20 分钟刷新一次

**缓存架构**（三级优先级）：
1. 内存中的 `remoteEvalFeatureValues` Map（进程内最新值）
2. 磁盘缓存 `~/.claude/config.json` 中的 `cachedGrowthBookFeatures`（跨进程持久化）
3. 旧版 `cachedStatsigGates`（迁移兼容层，逐步淘汰中）

**API 兼容 Workaround**（`growthbook.ts:320-390`）：服务端返回的 remoteEval 响应使用 `value` 字段，而 SDK 期望 `defaultValue`，代码中有显式的格式转换逻辑，并附有 TODO 注释等待服务端修复。

**环境变量覆盖**（仅限 ant 内部用户）：
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### 编译时 Feature Flag 完整清单（80+ 个）

通过 `bun:bundle` 的 `feature()` 调用实现死代码消除，以下是从源码中提取的全部编译时 flag：

| Flag 名称 | 所在位置 | 控制功能 |
|-----------|---------|---------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Perfetto 追踪（ant-only） |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | 增强遥测 beta |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | 自动模式/定时任务系统 |
| `KAIROS_BRIEF` | `commands.ts` | KAIROS 精简模式 |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | KAIROS 频道支持 |
| `KAIROS_DREAM` | `commands.ts` | KAIROS 梦境模式 |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | GitHub webhook 订阅 |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | KAIROS 推送通知 |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Agent 触发器（定时任务） |
| `AGENT_TRIGGERS_REMOTE` | — | 远程 Agent 触发器 |
| `AGENT_MEMORY_SNAPSHOT` | — | Agent 内存快照 |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | 对话分类器 |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | 验证代理 |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | 内置探索/计划代理 |
| `COORDINATOR_MODE` | `builtInAgents.ts` | 协调器模式 |
| `FORK_SUBAGENT` | `commands.ts` | Fork 子代理 |
| `BUDDY` | `commands.ts` | Buddy 功能 |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Unix Domain Socket 收件箱 |
| `BRIDGE_MODE` | `commands.ts` | 桥接模式（CCR） |
| `DAEMON` | `commands.ts` | 守护进程模式 |
| `VOICE_MODE` | `commands.ts` | 语音模式 |
| `ULTRAPLAN` | `commands.ts` | UltraPlan 命令 |
| `ULTRATHINK` | — | UltraThink 功能 |
| `TORCH` | `commands.ts` | TORCH 命令（动态加载） |
| `MCP_SKILLS` | `commands.ts` | MCP 技能支持 |
| `CHICAGO_MCP` | `metadata.ts` | Chicago MCP 内置服务器（computer-use） |
| `WORKFLOW_SCRIPTS` | `commands.ts` | 工作流脚本 |
| `CCR_REMOTE_SETUP` | `commands.ts` | CCR 远程设置命令 |
| `CCR_AUTO_CONNECT` | — | CCR 自动连接 |
| `CCR_MIRROR` | — | CCR 镜像模式 |
| `PROACTIVE` | `commands.ts` | 主动模式 |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | 实验性技能搜索 |
| `HISTORY_SNIP` | `commands.ts` | 历史片段功能 |
| `HISTORY_PICKER` | — | 历史选择器 |
| `WEB_BROWSER_TOOL` | — | Web 浏览器工具 |
| `QUICK_SEARCH` | — | 快速搜索 |
| `MONITOR_TOOL` | — | 监控工具 |
| `OVERFLOW_TEST_TOOL` | — | 溢出测试工具 |
| `BREAK_CACHE_COMMAND` | — | 强制缓存断点命令 |
| `TREE_SITTER_BASH` | — | Tree-sitter Bash 解析 |
| `TREE_SITTER_BASH_SHADOW` | — | Tree-sitter 影子对比 |
| `BASH_CLASSIFIER` | — | Bash 安全分类器 |
| `TERMINAL_PANEL` | — | 终端面板 |
| `NATIVE_CLIPBOARD_IMAGE` | — | 原生剪贴板图片支持 |
| `NATIVE_CLIENT_ATTESTATION` | — | 原生客户端证明 |
| `AUTO_THEME` | — | 自动主题 |
| `POWERSHELL_AUTO_MODE` | — | PowerShell 自动模式 |
| `TOKEN_BUDGET` | — | Token 预算显示 |
| `STREAMLINED_OUTPUT` | — | 精简输出模式 |
| `CONNECTOR_TEXT` | — | 连接器文本 |
| `CONTEXT_COLLAPSE` | — | 上下文折叠 |
| `COMPACTION_REMINDERS` | — | 压缩提醒 |
| `CACHED_MICROCOMPACT` | — | 缓存微压缩 |
| `REACTIVE_COMPACT` | — | 响应式压缩 |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Prompt Cache 断点检测 |
| `EXTRACT_MEMORIES` | — | 自动提取记忆 |
| `LODESTONE` | — | Lodestone 功能 |
| `TEAMMEM` | — | 团队记忆 |
| `TEMPLATES` | — | 模板功能 |
| `FILE_PERSISTENCE` | — | 文件持久化 |
| `BG_SESSIONS` | — | 后台会话 |
| `DOWNLOAD_USER_SETTINGS` | — | 用户设置下载 |
| `UPLOAD_USER_SETTINGS` | — | 用户设置上传 |
| `NEW_INIT` | — | 新版初始化流程 |
| `HARD_FAIL` | — | 硬失败模式 |
| `SLOW_OPERATION_LOGGING` | — | 慢操作日志 |
| `SHOT_STATS` | — | 请求统计 |
| `MEMORY_SHAPE_TELEMETRY` | — | 内存形状遥测 |
| `COWORKER_TYPE_TELEMETRY` | — | 协作者类型遥测 |
| `ANTI_DISTILLATION_CC` | — | 反蒸馏保护 |
| `RUN_SKILL_GENERATOR` | — | 技能生成器 |
| `SKILL_IMPROVEMENT` | — | 技能改进 |
| `REVIEW_ARTIFACT` | — | 代码审查制品 |
| `MESSAGE_ACTIONS` | — | 消息操作 |
| `AWAY_SUMMARY` | — | 离开摘要 |
| `COMMIT_ATTRIBUTION` | — | 提交归属 |
| `UNATTENDED_RETRY` | — | 无人值守重试 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | libc 类型检测（构建时注入） |

### 运行时 Feature Flag vs 编译时 Feature Flag

| 维度 | 编译时（`feature()`） | 运行时（GrowthBook） |
|------|----------------------|---------------------|
| 执行时机 | 打包阶段 | 进程启动后异步加载 |
| 代码保留 | 被删除的分支不存在于产物中 | 代码存在但逻辑受 flag 值控制 |
| 更新方式 | 发布新版本 | 服务端推送，最快 20 分钟生效 |
| 典型用途 | ant-only 功能、实验性工具、平台差异代码 | A/B 测试、灰度、kill-switch、动态配置 |
| 覆盖方式 | 构建变量 | `CLAUDE_INTERNAL_FC_OVERRIDES` 环境变量（仅 ant） |

### Dead Code Elimination 机制

`bun:bundle` 的 `feature()` 是 Bun 打包器的特殊内置函数，在构建阶段根据 build-time 定义直接将 `feature('X')` 替换为 `true` 或 `false`，再由常量折叠和死代码消除裁剪掉永假分支。

示例（`perfettoTracing.ts:216-220`）：
```typescript
// 在外部构建中，这整个 if 块被完全删除
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... 所有 Perfetto 初始化代码
}
```

这种机制不仅缩小包体积，还防止内部工具的代码暴露在外部产物中。

### 已知重要运行时 Feature Flags

以下为部分已知的 `tengu_*` GrowthBook flag 及其功能：

| Flag 名称 | 类型 | 功能说明 |
|-----------|------|---------|
| `tengu_auto_mode_config` | Object | 自动模式配置（enabled/opt-in） |
| `tengu_1p_event_batch_config` | Object | 1P 事件批量导出配置 |
| `tengu_event_sampling_config` | Object | 事件采样率配置字典 |
| `tengu_log_datadog_events` | Boolean | Datadog 事件上报开关 |
| `tengu_frond_boric` | Object | Analytics sink kill-switch（按 sink 名称禁用） |
| `tengu_quartz_lantern` | Boolean | FileWriteTool 原子写入行为控制 |
| `tengu_hive_evidence` | Boolean | 任务更新/Todo 写入行为控制 |
| `tengu_plum_vx3` | Boolean | WebSearchTool 使用 Haiku 模型的开关 |
| `tengu_kairos_cron` | Object | KAIROS 定时任务配置 |
| `tengu_kairos_cron_durable` | Boolean | 持久化定时任务支持 |
| `tengu_agent_list_attach` | Boolean | AgentTool 列表附加行为 |
| `tengu_amber_stoat` | Boolean | 内置代理可用性控制 |
| `tengu_slim_subagent_claudemd` | Boolean | 精简子代理 CLAUDE.md 加载 |
| `tengu_glacier_2xr` | Boolean | ToolSearch 模式决策控制 |
| `tengu_max_version_config` | Object | 最大版本限制（强制升级 kill-switch） |
| `tengu_prompt_cache_1h_config` | Object | Prompt Cache 1 小时配置 |
| `tengu_sm_compact_config` | Object | Session Memory 压缩配置 |
| `tengu_ant_model_override` | String | ant 专用模型覆盖 |
| `enhanced_telemetry_beta` | Boolean | 增强遥测 beta 开关 |

---

## 14.4 可观测性系统

### OpenTelemetry 集成

Claude Code 完整实现了 OpenTelemetry 三信号支持，核心入口在 `src/utils/telemetry/instrumentation.ts`（825 行）。

**初始化引导**（`instrumentation.ts:bootstrapTelemetry()`）：
在 ant 构建中，从 `ANT_OTEL_*` 前缀的变量读取配置并映射到标准 `OTEL_*` 变量。对外部用户则遵循标准 OTel 环境变量配置规范，默认 temporality 设为 `delta`（增量而非累计）。

**三信号导出器配置**（懒加载设计）：

```typescript
// instrumentation.ts:169-190（简化）
// OTLP/Prometheus 导出器使用动态 import 懒加载
// 避免 @grpc/grpc-js (~700KB) 在不需要时被加载
case 'grpc': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-grpc'
  )
  exporters.push(new OTLPMetricExporter())
  break
}
case 'http/protobuf': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-proto'
  )
  exporters.push(new OTLPMetricExporter(httpConfig))
  break
}
```

三种传输协议均支持：`grpc`、`http/json`、`http/protobuf`，通过 `OTEL_EXPORTER_OTLP_PROTOCOL` 环境变量选择。

**资源属性**：服务名为 `claude-code`，携带平台架构、WSL 版本、订阅类型、服务版本等属性，通过 `envDetector`、`hostDetector`、`osDetector` 自动填充。

### gRPC 数据传输

gRPC 是企业场景下的推荐传输协议，提供双向流式传输和强类型 protobuf 编码。在 Claude Code 中：

- gRPC 导出器(`@opentelemetry/exporter-metrics-otlp-grpc`)作为懒加载依赖，避免影响启动时间
- mTLS 配置通过 `getMTLSConfig()` 支持，企业内网场景可用自签证书
- 代理支持通过 `getProxyUrl()` + `HttpsProxyAgent` 透明处理

子进程不继承 OTEL 相关环境变量（`subprocessEnv.ts`）：
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Perfetto 追踪

Perfetto 是 Google 开发的高性能系统级追踪框架，Claude Code 实现了其 Chrome Trace Event 格式的兼容层（`src/utils/telemetry/perfettoTracing.ts`，1120 行，ant-only）。

**启用方式**：
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # 写入 ~/.claude/traces/trace-<session-id>.json
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # 写入指定路径
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # 每 60 秒定期写入
```

**追踪的 Span 类型**：

| Span 名称 | 分类 | 携带信息 |
|-----------|------|---------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | attempt 编号 |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**内存管理**（事件上限 100,000 条）：
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// 达到上限时删除最旧的一半，摊销 O(1) 成本
// 插入 trace_truncated 标记使缺口在 ui.perfetto.dev 可见
```

**多 Agent 层次追踪**：每个 Agent（包括子 Agent）映射到独立的 process ID，通过 `parent_agent` metadata 事件记录层次关系，在 Perfetto UI 中表现为独立的泳道。

**写入策略**（三重保障）：
1. `cleanup registry` 异步回调（正常退出）
2. `process.on('beforeExit')` 处理器（备用）
3. `process.on('exit')` 同步写入（最后防线，此时不能用 async）

### OpenTelemetry Session Tracing

`src/utils/telemetry/sessionTracing.ts`（927 行）是面向外部用户的增强遥测入口，基于标准 OTel Span 而非 Perfetto 格式。

**启用条件**（`sessionTracing.ts:170-185`）：
```typescript
export function isEnhancedTelemetryEnabled(): boolean {
  if (feature('ENHANCED_TELEMETRY_BETA')) {
    const env = process.env.CLAUDE_CODE_ENHANCED_TELEMETRY_BETA
      ?? process.env.ENABLE_ENHANCED_TELEMETRY_BETA
    if (isEnvTruthy(env)) return true
    if (isEnvDefinedFalsy(env)) return false
    return (
      process.env.USER_TYPE === 'ant' ||
      getFeatureValue_CACHED_MAY_BE_STALE('enhanced_telemetry_beta', false)
    )
  }
  return false
}
```

**AsyncLocalStorage 上下文传播**：每个 Interaction 和 Tool Call 使用独立的 ALS 存储 SpanContext，确保多 Agent 并发场景下 Span 不混淆。弱引用 WeakRef 防止长期存活的 span 内存泄漏，60 秒间隔清理超过 30 分钟的孤儿 Span。

**logEvent 事件体系**

所有业务事件通过 `src/services/analytics/index.ts` 的 `logEvent()` 函数统一派发：

```typescript
// index.ts（简化）
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // 仅允许 boolean | number | undefined
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

关键设计：metadata 类型故意排除 `string`，强制开发者使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型转换，在类型层面防止意外记录代码内容或文件路径。

---

## 14.5 Analytics 采集

### 双路路由架构

所有事件经 `sink.ts` 路由到两个后端：

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog（若 tengu_log_datadog_events gate 开启）
    │     仅传输 DATADOG_ALLOWED_EVENTS 白名单内的事件
    │     去除 _PROTO_* 键（PII 标记字段）
    └─→ 1P First-Party Logger（OpenTelemetry BatchLogRecordProcessor）
          发送到 /api/event_logging/batch
          保留 _PROTO_* 键（路由到 BigQuery 受保护列）
```

**Datadog 集成**（`datadog.ts`）：
- Endpoint：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- 批量发送：100 条/批，15 秒刷新间隔
- 网络超时：5 秒
- 白名单机制：约 50 个核心事件（`DATADOG_ALLOWED_EVENTS` Set）
- 禁用条件：Bedrock/Vertex/Foundry 第三方云、测试环境、用户选择 no-telemetry

**1P Event Logging（FirstPartyEventLoggingExporter）**：
- 使用 OpenTelemetry 标准 `LogRecordExporter` 接口
- 批量导出：默认 200 条/批，5 秒调度延迟
- 失败重试：指数退避（基础 500ms，最大 30s，最多 8 次）
- 持久化失败队列：失败事件写入 `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl`，下次启动时重试
- Proto 序列化：使用生成的 `ClaudeCodeInternalEvent` protobuf 类型

### 用户行为追踪

约 400+ 个 `tengu_*` 事件名称覆盖了完整的用户交互生命周期。核心事件类别：

**会话生命周期**：`tengu_started`、`tengu_init`、`tengu_exit`、`tengu_cancel`

**API 调用**：`tengu_api_query`、`tengu_api_success`、`tengu_api_error`、`tengu_api_retry`

**工具使用**：`tengu_tool_use_success`、`tengu_tool_use_error`、`tengu_tool_use_granted_in_prompt_permanent`

**权限请求**：`tengu_internal_bash_tool_use_permission_request`、`tengu_tool_use_show_permission_request`、`tengu_tool_use_granted_by_classifier`

**OAuth 认证**：`tengu_oauth_flow_start`、`tengu_oauth_success`、`tengu_oauth_token_refresh_*`（完整的锁状态机追踪）

**MCP 服务器**：`tengu_mcp_server_connection_succeeded`、`tengu_mcp_server_connection_failed`、`tengu_mcp_oauth_flow_*`

**更新机制**：`tengu_binary_download_attempt`、`tengu_native_update_complete`、`tengu_binary_download_failure`

### 性能指标采集

`sessionTracing.ts` 中的 API Call Span 会计算以下派生指标：

```typescript
// perfettoTracing.ts（endLLMRequestPerfettoSpan 简化）
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second（输入 token 处理速率）

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second（采样速率）

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // Cache 命中率（百分比）
```

### 事件采样控制

通过 GrowthBook 动态配置 `tengu_event_sampling_config` 控制各事件的采样率：

```typescript
// firstPartyEventLogger.ts（shouldSampleEvent 简化）
// 返回 null = 100% 采样（无配置）
// 返回 0 = 完全丢弃
// 返回 rate (0-1) = 随机采样，并将 sample_rate 写入 metadata
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // 示例：10% 采样
}
```

### 错误报告

多层错误事件体系：
- `tengu_uncaught_exception`、`tengu_unhandled_rejection`：进程级未捕获错误
- `tengu_api_error`、`tengu_query_error`：API 调用错误
- `tengu_streaming_error`：流式响应错误
- `tengu_atomic_write_error`：文件写入错误
- `tengu_compact_failed`：会话压缩失败

---

## 14.6 诊断与调试

### /doctor 命令

`src/commands/doctor/index.ts` 注册了 `/doctor` 命令：

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

该命令以 `local-jsx` 类型执行（在 REPL 内直接渲染 React 组件），检查内容包括：安装完整性、MCP 服务器连接状态、键绑定配置有效性、环境依赖（ripgrep 等）。

### 诊断追踪系统

IDE 集成场景下，Claude Code 通过 Language Server Protocol 接收代码诊断信息。当文件保存后（`didSave` 事件），TypeScript Server 发送新的诊断消息，系统将其注入为 `<new-diagnostics>` XML 标签传递给模型：

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### 堆内存诊断

`src/utils/heapDumpService.ts` 提供进程级内存诊断能力，在触发堆转储时同步采集内存使用快照，输出到 `~/Desktop/<session-id>-diagnostics.json`，包含 `heapUsed`、`external`、`rss` 以及分析建议。对应 analytics 事件：`tengu_heap_dump`。

### 错误恢复日志

`src/utils/telemetry/bigqueryExporter.ts` 实现了 BigQuery 指标导出器，与 OTEL Metrics 管道集成，用于 ant 内部的长期性能监控和容量规划。`1p_failed_events` 持久化队列确保即使网络故障也不丢失关键事件。

---

## 14.7 设计决策分析

### 编译时 Flag 的优缺点

**优点**：
1. **零运行时开销**：被删除的代码分支不存在于产物中，无任何条件判断开销
2. **安全隔离**：ant-only 功能代码对外部用户完全不可见，无法逆向
3. **包体积优化**：大型模块（如 `@grpc/grpc-js` ~700KB）只在需要的构建中存在
4. **类型安全**：TypeScript 的类型检查作用于打包前，不影响运行时

**缺点**：
1. **发布依赖**：变更 flag 状态必须发布新版本，无法热更新
2. **测试矩阵爆炸**：N 个编译时 flag 理论上需要 2^N 种构建组合测试
3. **调试复杂性**：外部用户报告问题时，某些代码路径在其构建中根本不存在

### 隐私与可观测性的平衡

Claude Code 在隐私保护上采用了多道防线：

1. **类型系统防护**：`LogEventMetadata` 仅允许 `boolean | number | undefined`，禁止字符串直接上报。要记录字符串必须显式声明 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`，这是一个 `never` 类型，无法真正持有值——它只是强制开发者写出类型注释，表明已手动验证该字符串不含代码或路径。

2. **MCP 工具名脱敏**：MCP 工具名格式 `mcp__<server>__<tool>` 可能泄露用户私有服务配置，默认脱敏为 `mcp_tool`。仅在 `cowork` 入口、官方 MCP 注册表内的服务器、或明确声明为内置的服务器名称才保留原始名称。

3. **PII 标记字段**：`_PROTO_*` 前缀的 metadata key 表示 PII 敏感字段，仅路由到 1P 受保护的 BigQuery 列，`sink.ts` 在转发 Datadog 之前会 strip 这些字段。

4. **第三方云禁用**：使用 Bedrock/Vertex/Foundry 的企业客户，所有 Anthropic 侧 analytics（含 Datadog 和 1P）默认关闭。

### 懒加载 Telemetry 的原因

OTLP 相关包（gRPC 约 700KB，proto 约 300KB）使用动态 `import()` 懒加载，原因是：

1. **启动时间敏感**：CLI 工具的首要性能指标是 Time-to-First-Output，任何不必要的初始化都应推迟
2. **协议互斥**：一个进程只会使用一种传输协议，静态 import 所有变体（6 个包）纯属浪费
3. **Bun 优化兼容**：懒加载符合 Bun 的模块解析优化模式，静态分析后按需打包

---

## 14.8 可迁移模式

以下设计模式对其他项目有较高参考价值：

### 1. 类型系统防止 PII 泄漏

通过 `never` 类型的 marker type，在编译期强制开发者显式确认不含敏感信息。代价为零（runtime 无开销），防护效果100%（绕过需要显式类型断言）。适用于任何有数据上报需求的系统。

### 2. 双级 Feature Flag 架构

编译时（代码分层）+ 运行时（行为控制）双轨制，对应不同的发布速度需求：
- 结构性功能（整个模块的有无）→ 编译时
- 行为调优（参数、比例、算法选择）→ 运行时

### 3. Sink Kill-Switch 模式

`tengu_frond_boric` GrowthBook 配置允许按名称（`datadog`、`firstParty`）独立关闭任意 analytics 后端，无需发布新版本。这是一个通用的紧急断路器模式，适合所有有多个下游 sink 的事件系统。

### 4. 失败事件持久化重试

1P 事件导出失败时写入本地 JSONL 文件，下次启动时重试。这保证了关键遥测数据在网络故障时不丢失，特别适合离线或不稳定网络环境下运行的工具。

### 5. 实验曝光去重

GrowthBook 实验曝光事件（用于 A/B 测试结果分析）通过 session 级 Set 去重，确保同一 feature 的曝光在分析侧只记录一次，防止多次调用同一 flag 导致曝光计数虚高。

---

## 14.9 源码索引

| 文件路径（相对 `src/`） | 行数 | 核心职责 |
|------------------------|------|---------|
| `services/analytics/growthbook.ts` | 1155 | GrowthBook SDK 集成，Feature Flag 读取，A/B 曝光记录 |
| `services/analytics/index.ts` | 173 | logEvent 公开 API，事件队列，Sink 接口定义 |
| `services/analytics/sink.ts` | 114 | 双路路由实现（Datadog + 1P），初始化 |
| `services/analytics/datadog.ts` | 307 | Datadog 批量日志发送，白名单过滤 |
| `services/analytics/firstPartyEventLogger.ts` | 449 | OpenTelemetry LoggerProvider 初始化，采样控制 |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | 1P 事件 HTTP 导出，持久化重试，proto 序列化 |
| `services/analytics/metadata.ts` | 973 | 事件元数据富化，MCP 工具名脱敏，PII 处理 |
| `services/analytics/config.ts` | 38 | isAnalyticsDisabled() 共享逻辑 |
| `services/analytics/sinkKillswitch.ts` | 25 | Sink 级 Kill-Switch（tengu_frond_boric） |
| `utils/telemetry/instrumentation.ts` | 825 | OTel SDK 初始化，三信号（Metrics/Logs/Traces）配置 |
| `utils/telemetry/sessionTracing.ts` | 927 | OTel Span 管理，AsyncLocalStorage 上下文传播 |
| `utils/telemetry/perfettoTracing.ts` | 1120 | Perfetto Chrome Trace 格式追踪（ant-only） |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Beta 追踪扩展属性 |
| `utils/telemetry/bigqueryExporter.ts` | 252 | BigQuery 指标导出器 |
| `utils/telemetry/pluginTelemetry.ts` | 289 | 插件遥测封装 |
| `utils/telemetry/events.ts` | 75 | OTel 事件类型定义 |
| `commands/doctor/index.ts` | 12 | /doctor 命令注册 |
| `commands.ts` | — | 编译时 feature() 集中调用处 |
