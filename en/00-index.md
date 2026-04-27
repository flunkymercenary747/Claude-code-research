# Claude Code Architecture Panoramic Analysis Report

> **Source Code Baseline**: Claude Code TypeScript source snapshot (2026-03-31, ~512K LOC, ~1,900 files)
> **Analysis Date**: 2026-04-02
> **Report Size**: 14 chapters, 428KB

---

## Project Background

### Source Code Origin

This report is based on the complete Claude Code TypeScript source snapshot leaked on 2026-03-31. The snapshot contains 512,664 lines of TypeScript code (`.ts` + `.tsx`) distributed across 1,884 files in 35 top-level directories. The source is stored on the mini server at `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`.

### Analysis Methodology

A **14-subagent parallel analysis** architecture was used: 5 Opus agents handled the most complex core chapters (architecture overview, Query Engine, Agent orchestration, security model, context management), and 9 Sonnet agents handled the remaining chapters. Each agent independently read the source code, extracted architectural patterns, and validated competitive analysis conclusions, with a chief editor doing a final unified review.

This methodology itself is a practical validation of Claude Code's multi-agent architecture (Chapter 4) — using Claude Code to analyze Claude Code.

### Comparison with win4r/cc-notebook

win4r/cc-notebook is the community's previous analysis notes on the same source code. This report makes significant enhancements in the following dimensions:

- **Independent Tool System chapter** (Chapter 3): cc-notebook did not analyze the tool system separately; this report fills that critical gap
- **Source-level verification**: Every architectural claim is accompanied by file name, line number, and code snippets, rather than second-hand paraphrase
- **Theory anchoring**: Each chapter opens with academic theoretical foundations (information theory, cache theory, cognitive science, etc.), placing engineering implementations in a broader knowledge framework

---

## Panoramic Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Claude Code Layered Architecture                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     User Interaction Layer                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │ Terminal  │  │ Command  │  │  Slash   │  │  Vim Mode     │  │    │
│  │  │ UI (Ink)  │  │ Parser   │  │ Commands │  │  (State Machine)│  │    │
│  │  │  Ch.12    │  │  Ch.7    │  │  Ch.7    │  │  Ch.12        │  │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────────┘  │    │
│  └───────┼──────────────┼──────────────┼──────────────────────────┘    │
│          │              │              │                                 │
│  ┌───────▼──────────────▼──────────────▼──────────────────────────┐    │
│  │                     Session Orchestration Layer                  │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │    │
│  │  │ Query Engine │  │    Agent     │  │  Prompt Engineering  │ │    │
│  │  │  (Main Loop) │  │ Orchestrator │  │  (System Prompt)     │ │    │
│  │  │  Ch.2        │  │  Ch.4        │  │  Ch.5               │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │    │
│  └─────────┼────────────────┼──────────────────────┼─────────────┘    │
│            │                │                      │                    │
│  ┌─────────▼────────────────▼──────────────────────▼─────────────┐    │
│  │                     Capability Execution Layer                  │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │ Tool     │  │  Skill   │  │  MCP     │  │  Bash/Read/  │  │    │
│  │  │ System   │  │  System  │  │ Client   │  │  Edit/Grep   │  │    │
│  │  │  Ch.3    │  │  Ch.6    │  │  Ch.11   │  │  (Built-in)  │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     State Persistence Layer                     │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │    │
│  │  │ Context Mgmt │  │   Memory     │  │  Session Resume  │    │    │
│  │  │ (Compaction) │  │  System      │  │                  │    │    │
│  │  │  Ch.8        │  │  Ch.9        │  │  Ch.9            │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘    │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     Infrastructure Layer                        │    │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ Security │  │ Cost Tracker │  │ Feature  │  │  OTel    │ │    │
│  │  │ & Perms  │  │ & Model Sel. │  │  Flags   │  │ Telemetry│ │    │
│  │  │  Ch.10   │  │  Ch.13       │  │  Ch.14   │  │  Ch.14   │ │    │
│  │  └──────────┘  └──────────────┘  └──────────┘  └──────────┘ │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Core Dependency Relationships Between Subsystems

```
Query Engine ──────► Claude API (claude.ts, 3,419 lines)
     │
     ├──► Tool System ──► 40+ built-in tools
     │         │
     │         ├──► MCP Client ──► external MCP servers
     │         │
     │         └──► Skill System ──► Markdown workflow definitions
     │
     ├──► Agent Orchestrator ──► runAgent() shared engine
     │         │
     │         ├──► Subagent (in-process)
     │         ├──► Coordinator Mode (async notifications)
     │         └──► Team Mode (file mailbox / Tmux)
     │
     ├──► Context Management ──► micro-compaction → full compaction (progressive degradation)
     │
     ├──► Security Layer ──► 8-layer defense in depth
     │         │
     │         ├──► AST Analysis (Bash parser)
     │         ├──► Sandbox (sandbox-exec / bwrap)
     │         └──► Permission Rules + Hooks
     │
     └──► Cost Tracker ──► real-time billing by model
```

### Data Flow (Lifecycle of a Single User Interaction)

```
User Input → Command Parser → REPL routing
                               │
                    ┌──────────┴──────────┐
                    ▼                      ▼
              Slash command           Natural language input
              (immediate)                  │
                                          ▼
                                  Prompt Assembly
                                  (system prompt + CLAUDE.md
                                   + tool descriptions + history)
                                          │
                                          ▼
                              ┌── Query Engine Main Loop ──┐
                              │                             │
                              ▼                             │
                        Claude API streaming request        │
                              │                             │
                              ▼                             │
                     ┌── Response type decision ──┐        │
                     │                             │        │
                     ▼                             ▼        │
                  Text output              Tool call        │
                  (render to UI)           │                │
                                          ▼                 │
                                   Permission check         │
                                          │                 │
                              ┌──────────┴──────┐          │
                              ▼                  ▼          │
                           Allow            Deny/Ask        │
                              │                  │          │
                              ▼                  │          │
                         Tool execution          │          │
                              │                  │          │
                              ▼                  ▼          │
                         Inject result into conversation ──►│
                              │                             │
                              ▼                             │
                     Context window check ──► Compact?     │
                              │                             │
                              └─────────────────────────────┘
                                    (loop until model stops)
```

---

## Code Scale Statistics

### Code Volume Distribution by Subsystem

| Subsystem | Core LOC | Core Files | Share |
|-----------|----------|------------|-------|
| Security & Permissions (Ch.10) | ~25,000 | 30+ | 4.9% |
| MCP Integration (Ch.11) | ~12,310 | 10+ | 2.4% |
| Agent Orchestration (Ch.4) | ~8,700 | 12 | 1.7% |
| Query Engine (Ch.2) | ~7,418 | 8 | 1.4% |
| Memory System (Ch.9) | ~5,700 | 17 | 1.1% |
| Context Management (Ch.8) | ~6,000 | 13+ | 1.2% |
| Tool System (Ch.3) | ~4,000+ | 40+ dirs | 0.8%+ |
| API Layer (claude.ts) | 3,419 | 1 | 0.7% |
| **Subtotal above** | **~72,500+** | — | **~14%** |
| Rest (UI/rendering/Skill/commands/config, etc.) | ~440,000 | — | ~86% |
| **Total** | **512,664** | **1,884** | **100%** |

### Top 20 Core Files by Line Count

| Rank | File | LOC | Subsystem |
|------|------|-----|-----------|
| 1 | `services/api/claude.ts` | 3,419 | API calls/streaming/retry/cache |
| 2 | `services/mcp/client.ts` | 3,348 | MCP connection management |
| 3 | `services/mcp/auth.ts` | 2,465 | MCP OAuth authentication |
| 4 | `services/teamMemorySync/` (5 files) | 2,167 | Team memory sync |
| 5 | `query.ts` | 1,729 | Query main loop |
| 6 | `memdir/` (7 files) | 1,736 | Memory directory management |
| 7 | `services/tools/toolExecution.ts` | 1,745 | Tool execution engine |
| 8 | `services/mcp/config.ts` | 1,578 | MCP config management |
| 9 | `inProcessRunner.ts` | 1,552 | Agent InProcess backend |
| 10 | `AgentTool.tsx` | 1,397 | Agent unified entry point |
| 11 | `QueryEngine.ts` | 1,295 | Session-level state management |
| 12 | `teammateMailbox.ts` | 1,183 | File mailbox protocol |
| 13 | `utils/collapseReadSearch.ts` | 1,109 | Read/search result collapsing |
| 14 | `spawnMultiAgent.ts` | 1,093 | Multi-agent spawning |
| 15 | `utils/toolResultStorage.ts` | 1,040 | Tool result cold storage |
| 16 | `runAgent.ts` | 973 | Agent execution engine |
| 17 | `SendMessageTool.ts` | 917 | 5-route message routing |
| 18 | `UI.tsx` (Agent) | 871 | Agent progress rendering |
| 19 | `Tool.ts` | 792 | Tool core abstraction |
| 20 | `extractMemories/` (2 files) | 769 | Memory extraction |

---

## Chapter Guide

### Chapter 1: Architecture Overview and Startup Flow
**File**: [01-architecture-overview.md](01-architecture-overview.md)

**Core Insight**: Claude Code is a terminal TUI application based on React/Ink, using Bun as the preferred runtime, driving the Claude API through a REPL loop to accomplish agentic programming tasks.

**Key Findings**:
- Precise tech stack choices: Bun + TypeScript + React/Ink + Zod v4 + OpenTelemetry, each with a clear engineering rationale
- 512,664 lines of code across 1,884 files in 35 top-level directories — a mature, large-scale engineering project
- Feature flag system uses a dual-track GrowthBook + bun:bundle approach, trimming internal features at compile time

**Recommended Priority**: ★★★★★ — Essential starting point for building a global mental model

---

### Chapter 2: Query Engine — LLM Interaction Core
**File**: [02-query-engine.md](02-query-engine.md)

**Core Insight**: The Query Engine is Claude Code's "heartbeat," driving the product's most critical path with just 7,400 lines of code (1.4%) — every user interaction with Claude passes through here.

**Key Findings**:
- Core architecture is an Async Generator Pipeline (coroutine pipeline), orchestrating streaming responses, tool calls, context compaction, and cost tracking within a single async generator
- Handles at least 12 exceptional branches: context window overflow, token budget exhaustion, API failures, user interruption, permission approval, etc.
- Token budget and auto-continue decision mechanism ensures long tasks are not interrupted when the model stops

**Recommended Priority**: ★★★★★ — Key to understanding how the entire product works

---

### Chapter 3: Tool System
**File**: [03-tool-system.md](03-tool-system.md)

**Core Insight**: The tool system is the only channel between LLM intent and the real world — not an accessory, but a core engineering asset.

**Key Findings**:
- Self-Describing Tools pattern: each tool self-declares its capabilities through `searchHint`, `prompt`, `userFacingSchema`, and other fields for dynamic model selection
- 40+ tool subdirectories, all scheduled by `toolExecution.ts` (1,745 lines)
- cc-notebook did not analyze this system separately; this chapter fills a critical gap

**Recommended Priority**: ★★★★☆ — Essential for Agent developers

---

### Chapter 4: Agent Orchestration and Multi-Agent Architecture
**File**: [04-agent-orchestration.md](04-agent-orchestration.md)

**Core Insight**: Three progressive collaboration modes (Subagent / Coordinator / Team Mode) share the same `runAgent()` core engine, achieving different behaviors through parameter combinations — one of the most elegant design decisions.

**Key Findings**:
- 8,700 lines of code across 12 core modules — the most complex subsystem in the entire product architecture
- Team Mode uses a file mailbox protocol (`teammateMailbox.ts`, 1,183 lines) for persistent parallel collaboration
- Coordinator Mode implements fully asynchronous communication via `<task-notification>` XML, with independent AbortController isolation

**Recommended Priority**: ★★★★★ — A textbook case for multi-agent system design

---

### Chapter 5: Prompt Engineering
**File**: [05-prompt-engineering.md](05-prompt-engineering.md)

**Core Insight**: Prompt engineering is the subsystem with the highest hidden complexity in the entire system — a 3-line `systemPromptSection` adjustment can simultaneously affect model behavior, Cache hit rate, token billing, and cross-session consistency.

**Key Findings**:
- An 8,000+ token system prompt is not a "description" — it's precise behavior programming
- Prompt Cache tiered strategy (`cacheScope: 'org'` / `'global'`) reduces token costs for millions of requests by an order of magnitude
- The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker precisely delineates the shared prefix boundary, maximizing global cache hits

**Recommended Priority**: ★★★★★ — Essential for prompt engineers; a model for commercial-grade Prompt design

---

### Chapter 6: Skill System
**File**: [06-skill-system.md](06-skill-system.md)

**Core Insight**: Skills are "SOPs for AI" — condensing human expert workflows in Markdown format to give AI reproducible professional execution capabilities.

**Key Findings**:
- Declarative + imperative fusion: Frontmatter declares metadata (permissions, model, triggers), body is execution instructions
- Multi-source loading with priority merging: bundled / user-level / project-level / Plugin-level / MCP sources
- Conditional activation mechanism: automatically activate via `paths` frontmatter by file path, supports dynamic discovery

**Recommended Priority**: ★★★★☆ — Directly applicable to Doramagic Skill design

---

### Chapter 7: Command System
**File**: [07-command-system.md](07-command-system.md)

**Core Insight**: The command system embodies clear separation of concerns — commands handle "triggering," tools handle "execution," LLM handles "decision-making."

**Key Findings**:
- Three command types: PromptCommand (inject into LLM conversation), LocalCommand (local execution), LocalJSXCommand (with UI rendering)
- Lazy loading design: commands are lazily loaded via `load(): Promise<Module>`, distributing startup overhead across first invocations
- Two key extensions to the classic GoF Command pattern: lazy loading + typed return values

**Recommended Priority**: ★★☆☆☆ — Conventional design pattern application; read as needed

---

### Chapter 8: Context Management
**File**: [08-context-management.md](08-context-management.md)

**Core Insight**: Context management is fundamentally an information compression problem — Claude Code designed a progressive degradation system from lossless to lossy, maintaining hour-long session continuity within a 200K token window.

**Key Findings**:
- Three-layer compression strategy: micro-compaction (cache_edits, lossless) → tool result collapsing (collapseReadSearch) → full Fork Agent compaction (lossy)
- The summary prompt's 9 sections define an implicit distortion metric — which information is least tolerable to lose
- Safety invariants: tool_use/tool_result pairs cannot be broken; recursive protection; Circuit Breaker mechanism

**Recommended Priority**: ★★★★☆ — Critical reference for long-session Agent development

---

### Chapter 9: Memory System
**File**: [09-memory-system.md](09-memory-system.md)

**Core Insight**: The memory system enables Claude to maintain cross-session continuity — mapping the cognitive science trichotomy of working memory / episodic memory / semantic memory to the three-layer architecture of context window / Session Memory / Persistent Memory.

**Key Findings**:
- 5,700 lines of code implementing a complete cross-session memory lifecycle: extraction, storage, sync, loading
- Team memory sync (`teamMemorySync/`, 2,167 lines) is the largest single module, supporting multi-person collaboration scenarios
- `memdir/` directory implements a filesystem-like memory organization structure

**Recommended Priority**: ★★★☆☆ — Reference value for Agents requiring long-term memory

---

### Chapter 10: Security and Permission Model
**File**: [10-security-permission.md](10-security-permission.md)

**Core Insight**: The security system is the largest subsystem by code volume (~25,000 lines), implementing 8-layer defense in depth — from AST parsing to OS sandboxing, each layer independently usable and cumulatively stacked.

**Key Findings**:
- Unique threat model: the AI model itself may be led by prompt injection to perform dangerous operations
- Pure TypeScript Bash AST parser to understand command structure (regex cannot distinguish `echo "rm -rf /"` from `rm -rf /`)
- Dual sandbox: application-level permission checking + OS-level isolation (macOS sandbox-exec / Linux bwrap), OS provides a safety net even if the application layer fails

**Recommended Priority**: ★★★★★ — Essential for security engineers; a benchmark for AI Agent security design

---

### Chapter 11: MCP Integration
**File**: [11-mcp-integration.md](11-mcp-integration.md)

**Core Insight**: Claude Code is a pure MCP client, with 12,310 lines of code implementing complete connection management, OAuth authentication, tool discovery, and dynamic registration.

**Key Findings**:
- MCP tools are dynamically registered in the format `mcp__<serverName>__<toolName>`, sharing the same execution framework as built-in tools
- OAuth 2.0 authentication includes XAA (Cross-Application Access) support
- Supports four transport layers: stdio/SSE/HTTP Streamable/WebSocket

**Recommended Priority**: ★★★☆☆ — Essential for MCP ecosystem developers

---

### Chapter 12: Terminal UI and Rendering Engine
**File**: [12-terminal-ui.md](12-terminal-ui.md)

**Core Insight**: Claude Code deeply transforms the React/Ink framework — double-buffered rendering, ANSI diff engine, hardware scroll acceleration, IME-aware cursor positioning — achieving a near-GUI interactive experience in a pure text terminal.

**Key Findings**:
- Rendering pipeline frame interval `FRAME_INTERVAL_MS = 16` (theoretical 62.5fps), aligned with monitor refresh rate
- Complete Vim mode implementation: independent state machine, covering NORMAL/INSERT dual mode, full set of operators/motions/textObjects
- Custom Ink renderer (`src/ink/`) replaces the native Ink rendering pipeline

**Recommended Priority**: ★★☆☆☆ — Deep reference for TUI framework developers; other readers may skip

---

### Chapter 13: Model Selection and Cost Control
**File**: [13-model-cost.md](13-model-cost.md)

**Core Insight**: Three design philosophies — user intent first (multi-layer override chain), fully transparent costs (force-print fees), no silent downgrade (Overload Fallback must alert).

**Key Findings**:
- Model selection priority chain: `/model` command → `--model` flag → environment variable → config file; upper layers override lower
- Built-in precise price table (`modelCost.ts`), supporting real-time cost tracking by model
- Opus → Sonnet Overload Fallback never silently switches; must display warning to user

**Recommended Priority**: ★★★☆☆ — Practical reference for multi-model system routing design

---

### Chapter 14: Feature Flags and Observability
**File**: [14-feature-flags-observability.md](14-feature-flags-observability.md)

**Core Insight**: Dual-track Feature Flags (compile-time bun:bundle DCE + runtime GrowthBook) combined with the three pillars of OpenTelemetry, building a full-chain observability system from compile-time trimming to runtime tracing.

**Key Findings**:
- Compile-time flags completely delete internal code via Dead Code Elimination, preventing reverse engineering
- Dual analytics: Datadog (external monitoring) + 1P event logging (internal BigQuery), event names prefixed `tengu_*`
- Privacy-first design: code content and file paths are not recorded by default

**Recommended Priority**: ★★☆☆☆ — Reference for large-scale SaaS observability design

---

## Reading Guide

### For AI Agent Developers

**Recommended route**: Ch.1 → Ch.2 → Ch.4 → Ch.3 → Ch.8 → Ch.10

What you most need to understand is the Query Engine's async generator orchestration pattern (Ch.2) and the three-layer Agent collaboration architecture (Ch.4). The tool system (Ch.3) shows how to design self-describing, extensible tool interfaces. Context management (Ch.8) solves the core engineering challenge of long-session scenarios. The security model (Ch.10) addresses the unavoidable trust boundary problem in AI Agents.

**Core takeaway**: The complete engineering implementation of the Agent loop — how large the gap is between "demo works" and "production ready."

---

### For Prompt Engineers

**Recommended route**: Ch.5 → Ch.6 → Ch.8 → Ch.9 → Ch.2

System prompt engineering (Ch.5) is the core — 8,000+ token behavior programming, Prompt Cache tiering, dynamic boundary markers. The Skill system (Ch.6) shows how to structure and preserve workflow knowledge. Context management (Ch.8) reveals information retention strategies in compression/summary scenarios. The memory system (Ch.9) is the cross-session persistence solution for Prompt context.

**Core takeaway**: Commercial-grade Prompt engineering is far more than "writing a description" — it's an interdisciplinary field of behavior programming, cache optimization, and cost control.

---

### For Security Engineers

**Recommended route**: Ch.10 → Ch.3 → Ch.11 → Ch.2 → Ch.14

The security model (Ch.10) is the top priority — 25,000 lines of code, 8-layer defense in depth, Bash AST analysis, OS sandboxing. The tool system (Ch.3) shows how permission checking is embedded in the tool execution pipeline. MCP integration (Ch.11) involves trust boundaries for third-party extensions. The Query Engine (Ch.2) reveals the attack surface through its 12 exceptional branch handlers. Observability (Ch.14) shows the infrastructure for tracking security events.

**Core takeaway**: AI Agent security is a fundamentally new threat model — the model itself may become an attack vector; traditional input validation is far from sufficient.

---

### For Doramagic Developers

**Recommended route**: Ch.6 → Ch.3 → Ch.5 → Ch.4 → Ch.9

The Skill system (Ch.6) directly maps to Doramagic's Skill form — declarative Frontmatter + imperative body, multi-source loading, conditional activation; these design patterns can be borrowed directly. The tool system (Ch.3) has reference value for OpenClaw tool design with its self-describing pattern. Prompt engineering (Ch.5) shows strategies for managing large-scale system prompts. Agent orchestration (Ch.4) is inspirational for future multi-Agent extraction pipelines. The memory system (Ch.9) has reference value for Doramagic's experience accumulation system.

**Core takeaway**: Claude Code's Skill system validates that the "SOP for AI" approach is correct — Doramagic's Skill design can stand on the shoulders of giants.

---

## Glossary

| Term | Definition |
|------|------------|
| **Agentic Loop** | The LLM alternates in a loop between "reasoning → tool call → result injection → re-reasoning" until the task is complete. Claude Code's core operating mode. |
| **Async Generator Pipeline** | TypeScript's `async function*` coroutine; the Query Engine uses it to orchestrate the incremental output of streaming responses. |
| **Auto-Continue** | When the model stops due to token limits (`end_turn` but with unfinished tool calls), the Query Engine automatically injects a continue instruction without interrupting the user experience. |
| **Cache Editing** | Claude API's cache editing capability, allowing deletion of cached copies of old messages without invalidating the entire cache. The foundation of micro-compaction. |
| **CLAUDE.md** | User/project-level instruction files, similar to `.editorconfig` but for AI. Injected into the system prompt to guide model behavior. |
| **Coordinator Mode** | The second-level Agent orchestration mode: fully asynchronous communication, coordinating multiple Agents via XML notifications, each with an independent AbortController. |
| **DCE (Dead Code Elimination)** | Removing unreachable code at compile time. bun:bundle's `feature()` calls completely delete internal features from external builds. |
| **Defense in Depth** | Layered defense strategy. Claude Code's 8-layer security system: permission mode → rule matching → AST analysis → safety validator → path validation → classifier → Hook → sandbox. |
| **Fork Agent** | An independent Agent launched during full context compaction, responsible for reading the complete conversation history and generating a structured summary without polluting the current session. |
| **Frontmatter** | YAML metadata block at the top of a Markdown file (wrapped in `---`). Skill files use it to declare permissions, models, triggers, etc. |
| **GrowthBook** | Runtime Feature Flag platform, supporting A/B testing and canary releases. Claude Code uses it to dynamically control feature switches and sampling rates. |
| **Ink** | React-based terminal UI framework. Claude Code deeply transforms its rendering pipeline (double-buffering, ANSI diff, hardware scrolling). |
| **MCP (Model Context Protocol)** | An open protocol led by Anthropic, defining standardized communication between AI applications and external tool services. Based on JSON-RPC 2.0. |
| **Micro-Compaction** | The first-level context management strategy: clear server-side cache for old tool results via cache editing, keeping local placeholders. Lossless. |
| **Overload Fallback** | When the primary model (e.g., Opus) is overloaded, transparently downgrade to a secondary model (e.g., Sonnet); must display a warning to the user, never silently switch. |
| **Prompt Cache** | Claude API's prompt caching mechanism. Requests with the same prefix share a cache, significantly reducing token costs. Available in `org` and `global` scopes. |
| **REPL** | Read-Eval-Print Loop, Claude Code's core interaction mode: read user input → LLM processing → output result → wait for next round. |
| **Skill** | A Markdown-format workflow definition, triggered via slash commands or proactive AI invocation. Essentially "SOPs for AI." |
| **Subagent** | The first-level Agent orchestration mode: a sub-Agent running synchronously/asynchronously within the main process, communicating via function return values. |
| **System Reminder** | System-level messages injected into conversation history, used to dynamically update the model's context information (e.g., current date, available Skill list). |
| **Team Mode** | The third-level Agent orchestration mode: persistent parallel collaboration via file mailbox protocol, supporting Tmux Pane / InProcess / Remote three backends. |
| **Tool Result Storage** | Disk cold storage for tool execution results (`toolResultStorage.ts`); results from micro-compacted tools are moved here from context and restored on demand. |
| **TUI (Terminal User Interface)** | Terminal user interface; the interactive interface Claude Code implements in a pure text terminal, based on declarative React/Ink rendering. |

---

*This report was produced in parallel by 14 analysis agents with unified review by a chief editor. For any discrepancies, the source code takes precedence.*
