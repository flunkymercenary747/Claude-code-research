# Claude Code Architecture Deep Analysis

🌍 **Select Language / 选择语言:**

| Language | Link |
|----------|------|
| 🇨🇳 中文 (Chinese) | [阅读中文版](zh/00-index.md) |
| 🇬🇧 English | [Read in English](en/00-index.md) |
| 🇰🇷 한국어 (Korean) | [한국어로 읽기](ko/00-index.md) |
| 🇯🇵 日本語 (Japanese) | [日本語で読む](ja/00-index.md) |
| 🇪🇸 Español (Spanish) | [Leer en español](es/00-index.md) |
| 🇧🇷 Português (Portuguese) | [Ler em português](pt/00-index.md) |
| 🇷🇺 Русский (Russian) | [Читать на русском](ru/00-index.md) |

---

## About This Project

A **14-chapter deep architecture analysis** of Claude Code, based on the TypeScript source snapshot (~512K LOC, ~1,900 files) that became publicly accessible on March 31, 2026.

**Analysis Method**: 14 parallel AI sub-agents (5 Opus + 9 Sonnet) each analyzed a subsystem directly from source files, producing source-level insights with code citations.

## Report Structure

| # | Chapter | Key Topic |
|---|---------|-----------|
| 00 | Index & Overview | Architecture panorama, reading guide, glossary |
| 01 | Architecture Overview | Entry points, startup sequence, parallel prefetch |
| 02 | Query Engine | LLM interaction core, streaming, retry mechanism |
| 03 | Tool System | 40+ tools unified abstraction, registration, execution |
| 04 | Agent Orchestration | Leader-Worker, Coordinator, Team, Worktree |
| 05 | Prompt Engineering | System prompt structure, Cache optimization, dynamic injection |
| 06 | Skill System | Markdown-as-Code, loading mechanism, bundled skills |
| 07 | Command System | 50+ slash commands, registration and execution |
| 08 | Context Management | 4-layer compression, circuit breaker, cache economics |
| 09 | Memory System | Three-tier memory, auto-extraction, intelligent recall |
| 10 | Security & Permissions | Bash AST analysis, sandbox, Parser Differential defense |
| 11 | MCP Integration | MCP protocol, OAuth, dynamic tool discovery |
| 12 | Terminal UI & Rendering | Ink dual-buffer, Int32Array buffer, Vim mode |
| 13 | Model Selection & Cost | Model routing, Fallback, Effort Level |
| 14 | Feature Flags & Observability | 80+ compile-time flags, 400+ runtime flags, Perfetto |

## Key Findings

- **Query Engine** is not a generic LLM wrapper but an inference-aware coroutine state machine
- **Agent orchestration logic lives in the system prompt**, not in code — Coordinator's 4-phase workflow is defined entirely by ~5000 characters of prompt
- **Prompt Engineering** uses a 3-layer cache architecture (global/org/null) where each conditional branch must evaluate its 2^N impact on cache variants
- **Context Management** is fundamentally a cache economics optimization — cache read ($0.30/M) is 10x cheaper than cache miss ($3/M)
- **Security model** core innovation is Parser Differential defense — systematically protecting against semantic gaps between AST parsers and real shells
- **Feature Flags** actually number 80+ compile-time and 400+ runtime flags, far exceeding previously published analyses

## Comparison with Similar Projects

| Dimension | [cc-notebook](https://github.com/win4r/cc-notebook) | This Project |
|-----------|------------|--------------|
| Total Size | ~150 KB / 13 files | **520 KB / 16 files** |
| Topics Covered | 7 | **14** |
| Languages | Chinese only | **7 languages** |
| New Coverage | — | Prompt Engineering, Skill, MCP, Tool System, Commands, Architecture, Terminal UI |
| Theoretical Foundation | None | CS theory anchors per chapter |
| Source Citations | Yes | Yes, with file:line numbers |

## Each Chapter Structure

1. **Overview & Positioning** — Role within the overall architecture
2. **Theoretical Foundation** — Corresponding CS theories and design patterns
3. **Architecture & Data Structures** — Core type definitions and architecture diagrams
4. **Core Algorithms & Flows** — Decision trees, state machines, code snippets
5. **Design Decision Analysis** — Tradeoff analysis and industry comparison
6. **Transferable Patterns** — Reusable engineering patterns
7. **Source Code Index** — File list with responsibilities

## Disclaimer

This project is for educational and technical research purposes. The intellectual property of Claude Code source code belongs to Anthropic. This project does not contain original source files, only architecture analysis and code snippet citations.
