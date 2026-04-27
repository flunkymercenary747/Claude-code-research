# Claude Code 架构全景分析报告

> **源码基线**：Claude Code TypeScript 源码快照（2026-03-31，~512K LOC，~1,900 文件）
> **分析日期**：2026-04-02
> **报告规模**：14 章，428KB

---

## 项目背景

### 源码来源

本报告基于 2026-03-31 泄露的 Claude Code 完整 TypeScript 源码快照。该快照包含 512,664 行 TypeScript 代码（`.ts` + `.tsx`），分布在 1,884 个文件中，涵盖 35 个顶层目录。源码存放于 mini 服务器 `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`。

### 分析方法论

采用 **14 子代理并行分析** 架构：5 个 Opus 代理负责核心复杂度最高的章节（架构总览、Query Engine、Agent 编排、安全模型、上下文管理），9 个 Sonnet 代理负责其余章节。每个代理独立阅读源码、提取架构模式、验证竞品分析结论，最终由总编辑统一审校。

这一方法论本身就是对 Claude Code 多代理架构（第 4 章）的实践验证——用 Claude Code 分析 Claude Code。

### 与 win4r/cc-notebook 的对比

win4r/cc-notebook 是此前社区对同一源码的分析笔记，本报告在以下维度做了显著增强：

- **独立工具系统章节**（第 3 章）：cc-notebook 未单独分析工具系统，本报告填补了这一关键空白
- **源码级验证**：每个架构声称都附带文件名、行号和代码片段，而非二手转述
- **理论锚定**：每章以学术理论基础开篇（信息论、缓存理论、认知科学等），将工程实现置于更宏观的知识框架中

---

## 全景架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Claude Code 分层架构                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     用户交互层 (User Interaction)                │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │ Terminal  │  │ Command  │  │  Slash   │  │  Vim Mode     │  │    │
│  │  │ UI (Ink)  │  │ Parser   │  │ Commands │  │  (状态机)      │  │    │
│  │  │  Ch.12    │  │  Ch.7    │  │  Ch.7    │  │  Ch.12        │  │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────────┘  │    │
│  └───────┼──────────────┼──────────────┼──────────────────────────┘    │
│          │              │              │                                 │
│  ┌───────▼──────────────▼──────────────▼──────────────────────────┐    │
│  │                     会话编排层 (Session Orchestration)           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │    │
│  │  │ Query Engine │  │    Agent     │  │  Prompt Engineering  │ │    │
│  │  │  (主循环)     │  │ Orchestrator │  │  (系统提示词编排)     │ │    │
│  │  │  Ch.2        │  │  Ch.4        │  │  Ch.5               │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │    │
│  └─────────┼────────────────┼──────────────────────┼─────────────┘    │
│            │                │                      │                    │
│  ┌─────────▼────────────────▼──────────────────────▼─────────────┐    │
│  │                     能力执行层 (Capability Execution)           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │ Tool     │  │  Skill   │  │  MCP     │  │  Bash/Read/  │  │    │
│  │  │ System   │  │  System  │  │ Client   │  │  Edit/Grep   │  │    │
│  │  │  Ch.3    │  │  Ch.6    │  │  Ch.11   │  │  (内置工具)    │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     状态持久层 (State Persistence)              │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │    │
│  │  │ Context Mgmt │  │   Memory     │  │  Session Resume  │    │    │
│  │  │ (上下文压缩)  │  │  System     │  │  (会话恢复)       │    │    │
│  │  │  Ch.8        │  │  Ch.9        │  │  Ch.9            │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘    │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     基础设施层 (Infrastructure)                 │    │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ Security │  │ Cost Tracker │  │ Feature  │  │  OTel    │ │    │
│  │  │ & Perms  │  │ & Model Sel. │  │  Flags   │  │ Telemetry│ │    │
│  │  │  Ch.10   │  │  Ch.13       │  │  Ch.14   │  │  Ch.14   │ │    │
│  │  └──────────┘  └──────────────┘  └──────────┘  └──────────┘ │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 子系统间的核心依赖关系

```
Query Engine ──────► Claude API (claude.ts, 3,419 行)
     │
     ├──► Tool System ──► 40+ 内置工具
     │         │
     │         ├──► MCP Client ──► 外部 MCP 服务器
     │         │
     │         └──► Skill System ──► Markdown 工作流定义
     │
     ├──► Agent Orchestrator ──► runAgent() 共享引擎
     │         │
     │         ├──► Subagent (进程内)
     │         ├──► Coordinator Mode (异步通知)
     │         └──► Team Mode (文件邮箱 / Tmux)
     │
     ├──► Context Management ──► 微压缩 → 全量压缩（渐进式降级）
     │
     ├──► Security Layer ──► 8 层纵深防御
     │         │
     │         ├──► AST 分析 (Bash 解析器)
     │         ├──► 沙箱 (sandbox-exec / bwrap)
     │         └──► Permission Rules + Hooks
     │
     └──► Cost Tracker ──► 按模型分类的实时计费
```

### 数据流向（一次用户交互的生命周期）

```
用户输入 → Command Parser → REPL 路由
                               │
                    ┌──────────┴──────────┐
                    ▼                      ▼
              斜杠命令               自然语言输入
              (立即执行)                  │
                                          ▼
                                  Prompt Assembly
                                  (系统提示词 + CLAUDE.md
                                   + 工具描述 + 历史)
                                          │
                                          ▼
                              ┌── Query Engine 主循环 ──┐
                              │                          │
                              ▼                          │
                        Claude API 流式请求               │
                              │                          │
                              ▼                          │
                     ┌── 响应类型判断 ──┐                │
                     │                  │                │
                     ▼                  ▼                │
                  文本输出          工具调用              │
                  (渲染到 UI)        │                   │
                                     ▼                   │
                              权限检查 (Security)        │
                                     │                   │
                              ┌──────┴──────┐            │
                              ▼              ▼            │
                           允许           拒绝/询问       │
                              │              │            │
                              ▼              │            │
                         工具执行            │            │
                              │              │            │
                              ▼              ▼            │
                         结果注入对话 ───────────────────►│
                              │                          │
                              ▼                          │
                     上下文窗口检查 ──► 触发压缩？        │
                              │                          │
                              └──────────────────────────┘
                                    (循环直到模型停止)
```

---

## 代码规模统计

### 按子系统的代码量分布

| 子系统 | 核心代码行数 | 核心文件数 | 占比 |
|--------|-------------|-----------|------|
| 安全与权限 (Ch.10) | ~25,000 | 30+ | 4.9% |
| MCP 集成 (Ch.11) | ~12,310 | 10+ | 2.4% |
| Agent 编排 (Ch.4) | ~8,700 | 12 | 1.7% |
| Query Engine (Ch.2) | ~7,418 | 8 | 1.4% |
| 记忆系统 (Ch.9) | ~5,700 | 17 | 1.1% |
| 上下文管理 (Ch.8) | ~6,000 | 13+ | 1.2% |
| 工具系统 (Ch.3) | ~4,000+ | 40+ 目录 | 0.8%+ |
| API 层 (claude.ts) | 3,419 | 1 | 0.7% |
| **以上合计** | **~72,500+** | — | **~14%** |
| 其余（UI/渲染/Skill/命令/配置等） | ~440,000 | — | ~86% |
| **总计** | **512,664** | **1,884** | **100%** |

### 核心文件 Top 20（按行数）

| 排名 | 文件 | 行数 | 所属子系统 |
|------|------|------|-----------|
| 1 | `services/api/claude.ts` | 3,419 | API 调用/流式/重试/缓存 |
| 2 | `services/mcp/client.ts` | 3,348 | MCP 连接管理 |
| 3 | `services/mcp/auth.ts` | 2,465 | MCP OAuth 认证 |
| 4 | `services/teamMemorySync/` (5 files) | 2,167 | 团队记忆同步 |
| 5 | `query.ts` | 1,729 | 查询主循环 |
| 6 | `memdir/` (7 files) | 1,736 | 记忆目录管理 |
| 7 | `services/tools/toolExecution.ts` | 1,745 | 工具执行引擎 |
| 8 | `services/mcp/config.ts` | 1,578 | MCP 配置管理 |
| 9 | `inProcessRunner.ts` | 1,552 | Agent InProcess 后端 |
| 10 | `AgentTool.tsx` | 1,397 | Agent 统一入口 |
| 11 | `QueryEngine.ts` | 1,295 | 会话级状态管理 |
| 12 | `teammateMailbox.ts` | 1,183 | 文件邮箱协议 |
| 13 | `utils/collapseReadSearch.ts` | 1,109 | 读/搜索结果折叠 |
| 14 | `spawnMultiAgent.ts` | 1,093 | 多 Agent 生成 |
| 15 | `utils/toolResultStorage.ts` | 1,040 | 工具结果冷存储 |
| 16 | `runAgent.ts` | 973 | Agent 执行引擎 |
| 17 | `SendMessageTool.ts` | 917 | 5 路消息路由 |
| 18 | `UI.tsx` (Agent) | 871 | Agent 进度渲染 |
| 19 | `Tool.ts` | 792 | 工具核心抽象 |
| 20 | `extractMemories/` (2 files) | 769 | 记忆提取 |

---

## 章节导读

### 第 1 章：架构总览与启动流程
**文件**：[01-architecture-overview.md](01-architecture-overview.md)

**核心洞察**：Claude Code 是一个基于 React/Ink 的终端 TUI 应用，以 Bun 为首选运行时，通过 REPL 循环驱动 Claude API 完成 agentic 编程任务。

**关键发现**：
- 技术栈选择精准：Bun + TypeScript + React/Ink + Zod v4 + OpenTelemetry，每个选择都有明确的工程理由
- 512,664 行代码分布在 1,884 个文件、35 个顶层目录中，是一个成熟的大型工程项目
- 功能开关系统采用 GrowthBook + bun:bundle 双轨制，编译期裁剪内部功能

**推荐优先级**：★★★★★ — 必读起点，建立全局心智模型

---

### 第 2 章：Query Engine — LLM 交互核心
**文件**：[02-query-engine.md](02-query-engine.md)

**核心洞察**：Query Engine 是 Claude Code 的"心跳"，用 7,400 行代码（仅占 1.4%）驱动了产品最关键的路径——每一次用户与 Claude 的交互都必经此处。

**关键发现**：
- 核心架构是 Async Generator Pipeline（协程管道），将流式响应、工具调用、上下文压缩、成本追踪编排在一个 async generator 中
- 处理至少 12 种异常分支：context window 溢出、token 预算耗尽、API 故障、用户中断、权限审批等
- Token 预算与 auto-continue 决策机制确保长任务不会因为模型 stop 而中断

**推荐优先级**：★★★★★ — 理解整个产品运作的关键

---

### 第 3 章：工具系统
**文件**：[03-tool-system.md](03-tool-system.md)

**核心洞察**：工具系统是 LLM 意图与真实世界之间的唯一通道——不是配件，而是核心工程资产。

**关键发现**：
- 自描述工具（Self-Describing Tools）模式：每个工具通过 `searchHint`、`prompt`、`userFacingSchema` 等字段自我声明能力，供模型动态选择
- 40+ 个工具子目录，统一由 `toolExecution.ts`（1,745 行）调度执行
- cc-notebook 未单独分析此系统，本章填补了关键空白

**推荐优先级**：★★★★☆ — Agent 开发者必读

---

### 第 4 章：Agent 编排与多代理架构
**文件**：[04-agent-orchestration.md](04-agent-orchestration.md)

**核心洞察**：三种递进的协作模式（Subagent / Coordinator / Team Mode）共享同一个 `runAgent()` 核心引擎，通过参数组合实现不同行为——这是最优雅的设计决策之一。

**关键发现**：
- 8,700 行代码，12 个核心模块，是整个产品架构中最复杂的子系统
- Team Mode 使用文件邮箱协议（`teammateMailbox.ts`，1,183 行）实现持久化并行协作
- Coordinator Mode 通过 `<task-notification>` XML 实现全异步通信，具备独立 AbortController 隔离

**推荐优先级**：★★★★★ — 多代理系统设计的教科书案例

---

### 第 5 章：Prompt 工程
**文件**：[05-prompt-engineering.md](05-prompt-engineering.md)

**核心洞察**：Prompt 工程是整个系统隐性复杂度最高的子系统——一个 3 行的 `systemPromptSection` 调整可能同时影响模型行为、Cache 命中率、token 计费和跨会话一致性。

**关键发现**：
- 8,000+ token 的系统提示词不是"描述"，而是精确的行为编程
- Prompt Cache 分层策略（`cacheScope: 'org'` / `'global'`）将数百万请求的 token 成本降低一个数量级
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记精确划定共享前缀边界，最大化全局缓存命中

**推荐优先级**：★★★★★ — Prompt 工程师必读，商业级 Prompt 设计的范本

---

### 第 6 章：Skill 系统
**文件**：[06-skill-system.md](06-skill-system.md)

**核心洞察**：Skill 是"给 AI 的 SOP"——将人类专家的工作流以 Markdown 格式沉淀，赋予 AI 可复现的专业执行能力。

**关键发现**：
- 声明式 + 执行式融合：Frontmatter 声明元数据（权限、模型、触发条件），正文是执行指令
- 多来源加载与优先级合并：bundled / 用户级 / 项目级 / Plugin 级 / MCP 来源
- 条件激活机制：通过 `paths` frontmatter 按文件路径自动激活，支持动态发现

**推荐优先级**：★★★★☆ — 对 Doramagic Skill 设计有直接参考价值

---

### 第 7 章：命令系统
**文件**：[07-command-system.md](07-command-system.md)

**核心洞察**：命令系统体现了清晰的关注点分离——命令负责"触发"，工具负责"执行"，LLM 负责"决策"。

**关键发现**：
- 三种命令类型：PromptCommand（注入 LLM 对话）、LocalCommand（本地执行）、LocalJSXCommand（带 UI 渲染）
- 惰性加载设计：命令通过 `load(): Promise<Module>` 延迟加载，将启动开销分摊到首次调用
- 经典 GoF 命令模式的两处关键扩展：惰性加载 + 类型化返回值

**推荐优先级**：★★☆☆☆ — 常规设计模式应用，按需阅读

---

### 第 8 章：上下文管理
**文件**：[08-context-management.md](08-context-management.md)

**核心洞察**：上下文管理本质上是信息压缩问题——Claude Code 设计了从无损到有损的渐进式降级体系，在 200K token 窗口内维持小时级会话连续性。

**关键发现**：
- 三层压缩策略：微压缩（cache_edits，无损）→ 工具结果折叠（collapseReadSearch）→ 全量 Fork Agent 压缩（有损）
- 摘要 Prompt 的 9 个章节定义了隐含的失真度量函数——哪些信息最不能容忍丢失
- 安全不变量：tool_use/tool_result 配对不可切断，递归保护，断路器机制

**推荐优先级**：★★★★☆ — 长会话 Agent 开发的关键参考

---

### 第 9 章：记忆系统
**文件**：[09-memory-system.md](09-memory-system.md)

**核心洞察**：记忆系统让 Claude 跨会话保持连续性——从认知科学的工作记忆/情节记忆/语义记忆三分法，映射到上下文窗口/Session Memory/Persistent Memory 三层架构。

**关键发现**：
- 5,700 行代码实现了完整的跨会话记忆生命周期：提取、存储、同步、加载
- 团队记忆同步（`teamMemorySync/`，2,167 行）是最大的单一模块，支撑多人协作场景
- `memdir/` 目录实现了类文件系统的记忆组织结构

**推荐优先级**：★★★☆☆ — 对需要长期记忆的 Agent 有参考价值

---

### 第 10 章：安全与权限模型
**文件**：[10-security-permission.md](10-security-permission.md)

**核心洞察**：安全系统是代码量最大的子系统（~25,000 行），实现了 8 层纵深防御——从 AST 解析到 OS 沙箱，每层独立可用，层层叠加。

**关键发现**：
- 独特威胁模型：AI 模型本身可能被 prompt injection 引导执行危险操作
- 纯 TypeScript 实现的 Bash AST 解析器，用于理解命令结构（正则无法区分 `echo "rm -rf /"` 和 `rm -rf /`）
- 双重沙箱：应用层权限检查 + OS 层隔离（macOS sandbox-exec / Linux bwrap），即使应用层失误也有 OS 兜底

**推荐优先级**：★★★★★ — 安全工程师必读，AI Agent 安全设计的标杆

---

### 第 11 章：MCP 集成
**文件**：[11-mcp-integration.md](11-mcp-integration.md)

**核心洞察**：Claude Code 是纯粹的 MCP 客户端，12,310 行代码实现了完整的连接管理、OAuth 认证、工具发现与动态注册。

**关键发现**：
- MCP 工具以 `mcp__<serverName>__<toolName>` 格式动态注册，与内置工具共享同一套执行框架
- OAuth 2.0 认证含 XAA（Cross-Application Access）跨应用访问支持
- 支持 stdio/SSE/HTTP Streamable/WebSocket 四种传输层

**推荐优先级**：★★★☆☆ — MCP 生态开发者必读

---

### 第 12 章：终端 UI 与渲染引擎
**文件**：[12-terminal-ui.md](12-terminal-ui.md)

**核心洞察**：Claude Code 对 React/Ink 框架进行深度改造——双缓冲渲染、ANSI diff 引擎、硬件滚动加速、IME 感知光标定位，在纯文本终端中实现了接近 GUI 的交互体验。

**关键发现**：
- 渲染管线帧间隔 `FRAME_INTERVAL_MS = 16`（理论 62.5fps），与显示器刷新率对齐
- 完整的 Vim 模式实现：独立状态机，覆盖 NORMAL/INSERT 双模式、operators/motions/textObjects 全集
- 自定义 Ink 渲染器（`src/ink/`）替换了原生 Ink 的渲染管线

**推荐优先级**：★★☆☆☆ — TUI 框架开发者的深度参考，其他读者可跳过

---

### 第 13 章：模型选择与成本控制
**文件**：[13-model-cost.md](13-model-cost.md)

**核心洞察**：三条设计哲学——用户意图优先（多层覆盖链）、成本完全透明（强制打印费用）、无秘密降级（Overload Fallback 必须告警）。

**关键发现**：
- 模型选择优先级链：`/model` 命令 → `--model` 标志 → 环境变量 → 配置文件，上层覆盖下层
- 内建精确价格表（`modelCost.ts`），支持按模型分类的实时成本追踪
- Opus → Sonnet 的 Overload Fallback 绝不静默切换，必须向用户显示警告

**推荐优先级**：★★★☆☆ — 多模型系统路由设计的实用参考

---

### 第 14 章：Feature Flags 与可观测性
**文件**：[14-feature-flags-observability.md](14-feature-flags-observability.md)

**核心洞察**：双轨制 Feature Flag（编译时 bun:bundle DCE + 运行时 GrowthBook）配合 OpenTelemetry 三大支柱，构建了从编译裁剪到运行追踪的全链路可观测体系。

**关键发现**：
- 编译时 flag 通过 Dead Code Elimination 完整删除内部代码，防止逆向
- 双路 Analytics：Datadog（外部监控）+ 1P 事件日志（内部 BigQuery），事件名前缀 `tengu_*`
- 隐私优先设计：默认不记录代码内容和文件路径

**推荐优先级**：★★☆☆☆ — 大规模 SaaS 可观测性设计的参考

---

## 阅读建议

### 如果你是 AI Agent 开发者

**推荐路线**：Ch.1 → Ch.2 → Ch.4 → Ch.3 → Ch.8 → Ch.10

你最需要理解的是 Query Engine 的 async generator 编排模式（Ch.2）和三层 Agent 协作架构（Ch.4）。工具系统（Ch.3）展示了如何设计自描述、可扩展的工具接口。上下文管理（Ch.8）解决了长会话场景下的核心工程挑战。安全模型（Ch.10）是 AI Agent 绕不开的信任边界问题。

**核心收获**：Agent 循环的完整工程实现，从"demo 能跑"到"生产可用"的差距有多大。

---

### 如果你是 Prompt 工程师

**推荐路线**：Ch.5 → Ch.6 → Ch.8 → Ch.9 → Ch.2

系统提示词工程（Ch.5）是核心——8,000+ token 的行为编程、Prompt Cache 分层、动态边界标记。Skill 系统（Ch.6）展示了如何将工作流知识结构化沉淀。上下文管理（Ch.8）揭示了 Prompt 在压缩/摘要场景下的信息保留策略。记忆系统（Ch.9）是跨会话 Prompt 上下文的持久化方案。

**核心收获**：商业级 Prompt 工程远超"写一段描述"——它是行为编程、缓存优化、成本控制的交叉学科。

---

### 如果你是安全工程师

**推荐路线**：Ch.10 → Ch.3 → Ch.11 → Ch.2 → Ch.14

安全模型（Ch.10）是重中之重——25,000 行代码、8 层纵深防御、Bash AST 分析、OS 沙箱。工具系统（Ch.3）展示了权限检查如何嵌入工具执行管线。MCP 集成（Ch.11）涉及第三方扩展的信任边界。Query Engine（Ch.2）的 12 种异常分支处理揭示了攻击面。可观测性（Ch.14）展示了安全事件的追踪基础设施。

**核心收获**：AI Agent 安全是一个全新的威胁模型——模型本身可能成为攻击向量，传统的输入校验远远不够。

---

### 如果你是 Doramagic 开发者

**推荐路线**：Ch.6 → Ch.3 → Ch.5 → Ch.4 → Ch.9

Skill 系统（Ch.6）与 Doramagic 的 Skill 形态直接对标——声明式 Frontmatter + 执行式正文、多来源加载、条件激活，这些设计模式可直接借鉴。工具系统（Ch.3）的自描述模式对 OpenClaw 工具设计有参考价值。Prompt 工程（Ch.5）展示了大规模系统提示词的管理策略。Agent 编排（Ch.4）对未来多 Agent 提取管线有启发。记忆系统（Ch.9）对 Doramagic 的经验积累系统有参考价值。

**核心收获**：Claude Code 的 Skill 系统验证了"给 AI 的 SOP"这条路线是对的——Doramagic 的 Skill 设计可以站在巨人肩膀上。

---

## 术语表

| 术语 | 定义 |
|------|------|
| **Agentic Loop** | LLM 在循环中交替执行"推理→工具调用→结果注入→再推理"，直到任务完成。Claude Code 的核心运行模式。 |
| **Async Generator Pipeline** | TypeScript 的 `async function*` 协程，Query Engine 用它编排流式响应的逐步产出。 |
| **Auto-Continue** | 当模型因 token 限制停止（`end_turn` 但有未完成工具调用）时，Query Engine 自动注入继续指令，不中断用户体验。 |
| **Cache Editing** | Claude API 的缓存编辑能力，允许在不失效整个缓存的前提下删除旧消息的缓存副本。微压缩的基础。 |
| **CLAUDE.md** | 用户/项目级的指令文件，类似 `.editorconfig` 但面向 AI。在系统提示词中注入，指导模型行为。 |
| **Coordinator Mode** | Agent 编排的第二层模式：全异步通信，通过 XML 通知协调多个 Agent，各自拥有独立 AbortController。 |
| **DCE (Dead Code Elimination)** | 编译时移除不可达代码。bun:bundle 的 `feature()` 调用使内部功能在外部构建中被完整删除。 |
| **Defense in Depth** | 纵深防御策略。Claude Code 的 8 层安全体系：权限模式→规则匹配→AST 分析→安全验证器→路径验证→分类器→Hook→沙箱。 |
| **Fork Agent** | 上下文全量压缩时启动的独立 Agent，负责阅读完整对话历史并生成结构化摘要，不污染当前会话。 |
| **Frontmatter** | Markdown 文件头部的 YAML 元数据块（`---` 包裹）。Skill 文件用它声明权限、模型、触发条件等。 |
| **GrowthBook** | 运行时 Feature Flag 平台，支持 A/B 测试和灰度发布。Claude Code 用它动态控制功能开关和采样率。 |
| **Ink** | 基于 React 的终端 UI 框架。Claude Code 深度改造了其渲染管线（双缓冲、ANSI diff、硬件滚动）。 |
| **MCP (Model Context Protocol)** | Anthropic 主导的开放协议，定义 AI 应用与外部工具服务的标准化通信。基于 JSON-RPC 2.0。 |
| **Micro-Compaction (微压缩)** | 上下文管理的第一级策略：通过 cache editing 清除旧工具结果的服务端缓存，本地保留占位符。无损。 |
| **Overload Fallback** | 主模型（如 Opus）过载时透明降级到次级模型（如 Sonnet），必须向用户显示警告，绝不静默切换。 |
| **Prompt Cache** | Claude API 的提示词缓存机制。相同前缀的请求共享缓存，大幅降低 token 成本。分 `org` 和 `global` 两种范围。 |
| **REPL** | Read-Eval-Print Loop，Claude Code 的核心交互模式：读取用户输入→LLM 处理→输出结果→等待下一轮。 |
| **Skill** | Markdown 格式的工作流定义，通过斜杠命令或 AI 主动调用触发。本质是"给 AI 的 SOP"。 |
| **Subagent** | Agent 编排的第一层模式：在主进程内同步/异步执行的子 Agent，通过函数返回值通信。 |
| **System Reminder** | 注入对话历史中的系统级消息，用于动态更新模型的上下文信息（如当前日期、可用 Skill 列表）。 |
| **Team Mode** | Agent 编排的第三层模式：通过文件邮箱协议实现持久化并行协作，支持 Tmux Pane / InProcess / Remote 三种后端。 |
| **Tool Result Storage** | 工具执行结果的磁盘冷存储（`toolResultStorage.ts`），微压缩后的工具结果从上下文移入此处，按需恢复。 |
| **TUI (Terminal User Interface)** | 终端用户界面，Claude Code 在纯文本终端中实现的交互式界面，基于 React/Ink 声明式渲染。 |

---

*本报告由 14 个分析代理并行产出，总编辑统一审校。如有疏漏，以源码为准。*
