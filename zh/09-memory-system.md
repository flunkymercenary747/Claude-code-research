# 第 9 章：记忆系统

## 9.1 概述与定位

Claude Code 的记忆系统是整个工具链中设计最精密、工程投入最深的子系统之一。它解决了 LLM 最根本的局限：上下文窗口在会话结束后归零。用户每次新开会话，Claude 都面临一张白纸——不知道用户是谁、偏好什么、上次犯了什么错误、团队有哪些规范。

记忆系统的设计目标是：**让 Claude 跨会话保持连续性，像一个真正的长期协作者一样行事。**

从源码体量来看，这是一个规模可观的系统：
- `memdir/` 目录：7 个文件，1736 行
- `services/SessionMemory/`：3 个文件，1026 行
- `services/extractMemories/`：2 个文件，769 行
- `services/teamMemorySync/`：5 个文件，2167 行

合计约 5700 行，占整个代码库的约 1.1%，但其复杂度和设计思考密度远超这个比例。

---

## 9.2 理论基础

### 人类记忆模型的对应

系统架构明确对应了认知科学中的三种记忆：

| 人类记忆 | Claude Code 对应 | 技术实现 |
|---------|-----------------|---------|
| 工作记忆（Working Memory）| 当前上下文窗口 | 会话消息列表，随会话结束清空 |
| 情节记忆（Episodic Memory）| Session Memory | `~/.claude/projects/<slug>/session-memory.md`，会话内持续更新 |
| 语义记忆（Semantic Memory）| Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`，跨会话长期保存 |

Session Memory 对应"当下的回忆"——它记录这次会话正在做什么、到了哪一步；Persistent Memory 对应"积累的知识"——用户偏好、反馈教训、项目背景。

### 知识图谱 vs 文档记忆的选择

系统选择了**文件系统上的 Markdown 文档**而非数据库或向量索引。这一选择在 `memoryTypes.ts` 的注释中有明确说明：

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

这揭示了一个第一性原则：**可以实时查询的信息不应该记忆。** 记忆只存储"不可派生"的上下文——用户的偏好、团队的历史教训、项目背后的动机。这与知识图谱的设计截然不同，后者倾向于将一切可以结构化的信息都放进去。

### 最终一致性在记忆中的应用

Team Memory 的同步设计明确采用了最终一致性语义：

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

删除不传播这一设计是有意为之——团队记忆是"追加型"资产，误删不应成为永久损失。这是对分布式系统最终一致性原则的一种保守实现。

---

## 9.3 三层记忆架构

系统由三层构成，从生命周期最短到最长依次排列：

### 第一层：Session Memory（会话级）

**文件路径**：`~/.claude/projects/<sanitized-cwd>/session-memory.md`（通过 `getSessionMemoryPath()` 获取）

Session Memory 是一个在**当前会话内持续维护**的 Markdown 文件，内容结构固定：

```markdown
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

（`services/SessionMemory/prompts.ts:14-36`，`DEFAULT_SESSION_MEMORY_TEMPLATE`）

它不会随会话结束清除，而是被 Auto Compact 机制在压缩上下文时读取，作为"前情提要"注入新的上下文窗口。

**数据结构约束**：
- 每个 section 上限 2000 tokens（`MAX_SECTION_LENGTH = 2000`）
- 全文上限 12000 tokens（`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`）
- 超出限制时系统会在 prompt 中添加警告并要求 Agent 压缩

**生命周期**：与当前项目 session 绑定，Auto Compact 触发时读取

### 第二层：Persistent Memory（跨会话持久记忆）

**文件路径**：`~/.claude/projects/<sanitized-git-root>/memory/`

这是核心的长期记忆层。每条记忆独立存为一个 `.md` 文件，带有 YAML frontmatter：

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

（`memdir/memoryTypes.ts:230-240`，`MEMORY_FRONTMATTER_EXAMPLE`）

路径解析逻辑由 `getAutoMemPath()` 负责（`memdir/paths.ts:173-190`），解析优先级为：

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 多用户场景使用）
2. `settings.json` 中的 `autoMemoryDirectory`（仅信任 policy/local/user 来源，**不信任** projectSettings 以防恶意 repo 劫持写入路径）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`（默认）

Git 工作树统一到 canonical git root（`findCanonicalGitRoot`），确保同一仓库的不同 worktree 共享同一份记忆。

**生命周期**：永久，直到用户显式删除或 Agent 主动更新/删除

### 第三层：Team Memory（团队共享记忆）

**文件路径**：`~/.claude/projects/<sanitized-git-root>/memory/team/`（`getTeamMemPath()` 返回值）

Team Memory 是 Persistent Memory 的子目录，通过 REST API 在同一 GitHub 仓库的所有认证成员间同步。它是 Auto Memory 之上的扩展，`isTeamMemoryEnabled()` 会先检查 `isAutoMemoryEnabled()` 确保父系统开启。

**生命周期**：由 Anthropic 服务端维护，跨用户、跨机器持久化

---

## 9.4 MEMORY.md 索引机制

MEMORY.md 是 Persistent Memory 层的**索引文件**，而非内容文件。系统在多处明确区分了这两者：

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### 格式规范

MEMORY.md 的每一行是一个指向具体记忆文件的链接：

```
- [用户偏好：简洁响应](feedback_terse_responses.md) — 用户不喜欢在回复末尾总结
- [项目背景：Auth 中间件重写](project_auth_rewrite.md) — 法务合规要求，非技术债
```

MEMORY.md 在每次会话开始时被加载进系统 prompt，因此其大小直接影响每次请求的 token 消耗。

### 200 行 / 25KB 双重限制

系统在 `memdir/memdir.ts` 中定义了严格的双重上限：

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

（`memdir/memdir.ts:30-33`）

截断逻辑在 `truncateEntrypointContent()` 中实现（`memdir/memdir.ts:55-102`）：先按行数截断，再按字节数截断（在最近一个换行符处切割避免截断中间行）。截断后追加警告信息提示用户索引过长。

**设计意图**：每行约 125 字符 × 200 行 ≈ 25KB，这是经过实测（p97 分位）的合理上限。字节限制是针对"200 行以内但每行极长"的边界情况（实测 p100：197KB 未超行数限制）。

### 与记忆文件的关系

写入记忆是**两步操作**：
1. 写内容文件（`user_role.md`、`feedback_testing.md` 等）
2. 在 MEMORY.md 中添加指向条目

读取时只有被 findRelevantMemories 选中的文件才会被读取（详见 9.7），MEMORY.md 本身则常驻 system prompt。

---

## 9.5 四种记忆类型

系统将所有记忆约束在四种类型内，这是设计上最重要的决策之一。类型定义在 `memdir/memoryTypes.ts` 中（`MEMORY_TYPES` 常量）：

### user 类型

**适用场景**：用户的角色、目标、责任、知识背景

**触发时机**：任何时候获知用户的角色、偏好、职责或知识水平

**用途**：调整回复方式以适应具体用户的认知水平和需求

**范围**：始终是 private（个人私有），在 Team Memory 模式下也如此

**反面案例（不应保存的内容）**：
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### feedback 类型

**适用场景**：用户对工作方式的纠正和确认——既包括"不要这样做"，也包括"继续保持这样做"

**结构要求**：
- 规则本身
- `**Why:**` 行（给出原因，以便在边缘情况下判断是否适用）
- `**How to apply:**` 行（何时何地生效）

**独特设计**：明确要求同时记录**失败教训和成功确认**：

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**触发时机**：用户说"不要这样"（显式纠正）或"正是这样"/"完美"（隐式确认，更难识别）

**范围**：默认 private；只有当指导方针明确是项目级规范时（如测试策略、构建约束）才保存为 team

### project 类型

**适用场景**：关于正在进行的工作、目标、计划、Bug 或事件的信息，这些信息**无法从代码或 git 历史中派生**

**结构要求**：
- 事实/决策本身
- `**Why:**` 行（动机——通常是约束条件、截止日期或干系人需求）
- `**How to apply:**` 行（如何影响建议）

**重要规则**：保存时必须将相对日期转换为绝对日期（"下周四" → "2026-04-08"），确保记忆在时间流逝后仍可解释。

**范围**：默认 team（项目上下文本质上是共享的）

**衰减特性**：project 类型记忆衰减最快，Why 字段有助于判断记忆是否仍然有效。

### reference 类型

**适用场景**：指向外部系统中信息位置的指针（Linear 项目、Slack 频道、Grafana 看板等）

**触发时机**：获知外部资源位置及其用途

**范围**：通常是 team

**典型示例**：

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### 不应保存的内容（明确排除）

`WHAT_NOT_TO_SAVE_SECTION` 明确列出了不该保存的六类内容（`memdir/memoryTypes.ts:196-207`）：

1. 代码模式、约定、架构、文件路径——可从项目当前状态派生
2. Git 历史、最近变更——`git log`/`git blame` 是权威来源
3. Debug 解决方案或修复方法——修复在代码里，上下文在 commit message 里
4. 已在 CLAUDE.md 中记录的内容
5. 临时任务细节：进行中的工作、临时状态、当前会话上下文
6. **即使用户明确要求保存的上述内容**——如果用户要求保存 PR 列表，应该问"有什么出人意料的或不显而易见的内容？那才值得保存"

---

## 9.6 自动记忆提取

### Fork Agent 自动提取机制

记忆提取使用了"Fork Agent"模式——创建一个与主会话完全相同的 Agent 上下文，在后台异步运行，不阻塞主对话流。

这一机制的核心是 `runForkedAgent()`，提取 Agent 共享父会话的 prompt cache，实现最大化的缓存命中率：

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // 不写入主会话记录，避免竞态
  maxTurns: 5,            // 硬上限防止验证死循环
})
```

（`services/extractMemories/extractMemories.ts:258-267`）

`maxTurns: 5` 的设计注释说明了意图：

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

提取 Agent 的高效策略被明确设计为"2轮完成"：
- **第1轮**：并行发出所有需要更新的文件的 FileRead 请求
- **第2轮**：并行发出所有 FileWrite/FileEdit 请求

### 触发时机（Stop Hooks）

提取在**每次完整的查询循环结束后**触发——即模型产生没有 tool_use 的最终响应时，通过 `handleStopHooks` 调用 `executeExtractMemories()`。

状态通过闭包管理，关键变量包括：

```typescript
let lastMemoryMessageUuid: string | undefined    // 游标：上次提取到哪
let inProgress = false                           // 防止并发运行
let pendingContext: {...} | undefined            // 运行中到达的调用存入此处
let turnsSinceLastExtraction = 0                // 用于节流控制
```

（`services/extractMemories/extractMemories.ts:225-240`）

**并发控制策略**：如果提取正在进行时又有新调用到来，新调用被"stash"（存入 `pendingContext`）而非丢弃。当前提取完成后，会立即用最新的 context 运行一次"trailing extraction"，确保最后一批消息不被遗漏。

**互斥规则**：若主 Agent 自己已经写了记忆文件（`hasMemoryWritesSince` 检测），Fork Agent 跳过本次提取，只推进游标：

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // 主 Agent 写了，跳过 fork agent，推进游标
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

（`services/extractMemories/extractMemories.ts:198-209`）

### 提取提示词分析

提取提示词的核心设计哲学是**信息效率**：

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // 预注入现有记忆列表，避免 Agent 花一轮 ls
  ].join('\n')
}
```

（`services/extractMemories/prompts.ts:20-47`）

预注入现有记忆清单（`existingMemories`）是关键优化——避免 Agent 浪费一轮调用来列出目录，直接在 prompt 里给出结构化的文件清单（文件名、类型、时间戳、description）。

### Session Memory 的触发机制

Session Memory 使用不同的触发机制——通过 `postSamplingHooks` 而非 Stop Hooks，在每次模型采样后评估是否需要更新：

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

（`services/SessionMemory/sessionMemory.ts:130-150`）

默认触发阈值（`DEFAULT_SESSION_MEMORY_CONFIG`，`services/SessionMemory/sessionMemoryUtils.ts:29-33`）：

| 参数 | 默认值 | 说明 |
|-----|-------|------|
| `minimumMessageTokensToInit` | 10,000 | 初始化会话记忆所需的最低 token 数 |
| `minimumTokensBetweenUpdate` | 5,000 | 两次更新之间最少需要增长的 token 数 |
| `toolCallsBetweenUpdates` | 3 | 两次更新之间最少需要的 tool 调用次数 |

这些值可以通过 GrowthBook 远程配置（`tengu_sm_config`）动态调整。

---

## 9.7 智能记忆召回

### Sonnet 选择最多 5 条相关记忆

记忆召回不是全量读取，而是**先扫描 frontmatter，再用 Sonnet 选出最相关的最多 5 条**。

核心流程在 `findRelevantMemories()` 中（`memdir/findRelevantMemories.ts:32-66`）：

1. `scanMemoryFiles()` 扫描记忆目录，读取每个文件的前 30 行（frontmatter），返回 `MemoryHeader[]`
2. 过滤掉已经在前几轮展示过的记忆（`alreadySurfaced`），节省 5 个名额给新内容
3. 用 Sonnet 调用 `selectRelevantMemories()`，基于 query 和文件 description 选出最相关的文件名
4. 返回选中记忆的路径和 mtime

### 相关性判断逻辑

Sonnet 的 system prompt 经过精心设计：

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

（`memdir/findRelevantMemories.ts:13-23`）

**关键设计**：最近使用过的工具的参考文档不应被选中（使用中无需参考文档），但同一工具的**坑/已知问题**记忆仍应选中（正在使用时最需要踩坑警告）。

API 调用使用结构化输出（JSON Schema）确保返回格式可解析：

```typescript
output_format: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      selected_memories: { type: 'array', items: { type: 'string' } },
    },
    required: ['selected_memories'],
    additionalProperties: false,
  },
},
```

（`memdir/findRelevantMemories.ts:97-108`）

### 记忆注入到上下文的方式

被选中的记忆以 `<system-reminder>` 标签包裹注入用户消息之前（`wrapMessagesInSystemReminder`）。超过 1 天的记忆会附加新鲜度警告：

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

（`memdir/memoryAge.ts:38-47`）

这个设计解决了一个实际问题：用户报告了"基于过期记忆做出自信断言"的问题——引用的文件路径或函数名已经被修改，但记忆中的 citation 让断言看起来更可信而非更可疑。

**防漂移机制**：`MEMORY_DRIFT_CAVEAT` 被注入 system prompt，要求 Agent 在根据记忆回答之前验证当前状态：

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Team Memory 同步

### REST API 同步机制

Team Memory 通过 `services/teamMemorySync/` 实现服务端同步，API 设计在 `index.ts` 顶部有完整描述：

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → 仅元数据+哈希
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert entries
404  = 尚无数据
```

（`services/teamMemorySync/index.ts:10-13`）

同步依赖 **OAuth 认证**（需要 `CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`），并以 GitHub 仓库（`owner/repo`）作为 scope。

**Watcher 机制**：`watcher.ts` 使用 `fs.watch({recursive: true})` 监听 team 目录变化，防抖 2 秒后触发 push（`DEBOUNCE_MS = 2000`）。刻意选择原生 `fs.watch` 而非 chokidar：

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS 使用 FSEvents（O(1) 文件描述符），Linux 使用 inotify（O(子目录数量)），均优于 chokidar 的 kqueue 方案。

### 乐观锁定（If-Match）

上传使用乐观并发控制，通过 `If-Match` HTTP 头携带 ETag（checksum）：

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

（`services/teamMemorySync/index.ts:uploadTeamMemory`）

服务端返回 412 Precondition Failed 时表示发生冲突（另一个用户在此期间修改了共享记忆）。系统使用 `GET ?view=hashes` 端点（轻量级，仅返回每个 key 的 SHA-256 哈希，无内容体）刷新本地的 `serverChecksums`，然后重算 delta 重试：

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### 冲突解决策略

冲突解决的策略是**服务端赢（server wins per-key）**——Pull 时服务端内容覆盖本地。Delta push 只上传本地与服务端哈希不同的 key，服务端使用 upsert 语义（未在 PUT 中出现的 key 被保留）。

批量上传限制（`MAX_PUT_BODY_BYTES = 200_000`）防止请求体过大被 API Gateway 拒绝（观测到 gateway 在约 256-512KB 时返回 HTML 格式的 413，与应用层的结构化 413 不同）。超出限制时自动拆分为多个顺序 PUT，upsert 语义保证安全：

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // 贪心装箱：按字节数分批，每批不超过 MAX_PUT_BODY_BYTES
  ...
}
```

（`services/teamMemorySync/index.ts:batchDeltaByBytes`）

**永久失败抑制**：某些错误（no_oauth、no_repo、4xx 非 409/429）是无法通过重试自愈的。系统检测到这类错误后会设置 `pushSuppressedReason`，阻止 watcher 触发的 push 陷入无限重试循环（曾观测到一台无 OAuth 的设备在 2.5 天内发出 167K push 事件）。

---

## 9.9 设计决策分析

### 为什么用文件系统而非数据库

文件系统 + Markdown 的设计有几个关键优势：

1. **Agent 可直接操作**：FileRead/FileWrite/FileEdit 工具是 Claude 的原生工具，无需额外 API 层。Agent 写记忆和写代码使用同一套工具，认知负担最低。

2. **用户可检查**：`~/.claude/projects/.../memory/` 是普通文件夹，用户可以直接 `ls`、`cat`、`vim`，完全透明。

3. **Git 友好**：Markdown 文件天然支持 diff、grep、git history，方便 Team Memory 的 delta 计算。

4. **避免不必要的抽象**：数据库需要 schema 迁移、备份策略、查询层——对于"几百个 KB 的 Markdown 文件"来说是过度工程。

### 为什么限制 MEMORY.md 大小

200 行 / 25KB 的限制背后有实测数据支撑（p97/p100 观测值）。核心原因：

- MEMORY.md 在**每次请求**时都会加载进 system prompt，大小直接影响 token 消耗
- 过大的索引会挤占真正有用的上下文空间
- 强制限制促使用户和 Agent 保持索引精炼，每行只写"hook"而非内容

这是"用约束促进质量"的典型设计——不是因为技术上无法容纳更多，而是通过约束引导正确使用方式。

### 记忆安全的设计考量

系统有多层安全设计：

**路径遍历防护**：`teamMemPaths.ts` 实现了三层检查——先字符串级别检查 `..`、URL 编码遍历、Unicode 规范化攻击，再通过 `realpath` 解析符号链接验证实际文件系统路径：

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

（`memdir/teamMemPaths.ts:130-133`）

**Secret 扫描**：写入 Team Memory 时，`scanForSecrets()` 扫描 30 种高置信度凭证模式（来自 gitleaks 规则库），包括 AWS、GCP、GitHub、Anthropic、OpenAI 等主要平台的 token 格式。扫描在**上传前**和**写入前**双重执行：

- `teamMemSecretGuard.ts` 的 `checkTeamMemSecrets()` 在 FileWriteTool/FileEditTool 的 `validateInput` 阶段拦截写入
- `readLocalTeamMemory()` 在 push 前再次扫描，跳过含敏感信息的文件

**最小权限工具控制**：提取 Agent 的 `canUseTool` 函数只允许：
- FileRead/Grep/Glob（只读）
- 只读 Bash 命令（ls/find/cat/stat/wc/head/tail）
- FileEdit/FileWrite 且路径必须在 memory 目录内

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

（`services/extractMemories/extractMemories.ts:171-176`）

**ProjectSettings 安全豁免**：`autoMemoryDirectory` 的设置只信任 policy/local/user 来源，明确排除 projectSettings（`.claude/settings.json`）：

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 可迁移模式

以下是 Doramagic 记忆系统设计可直接借鉴的模式：

### 模式一：不可派生原则

**什么应该记忆**：凡是可以通过查询当前状态（代码、文件、git）得到的信息，都不值得记忆。记忆只应存储"历史上下文"——为什么做了这个决定、踩过什么坑、用户的隐式偏好。

**Doramagic 应用**：Soul Extractor 提取的"UNSAID"和"WHY"层，天然符合这一原则。OpenClaw 规则文档是可查询的，不需要记忆；但"这个 OpenClaw 规则曾经导致发布失败"这类教训，才是值得记忆的内容。

### 模式二：双步写入 + 轻量索引

文件 + 索引的两步写入模式，确保索引永远精炼（强制约束每行 150 字以内），内容文件可以详细展开。索引的 token 消耗是固定的，内容的读取是按需的。

**Doramagic 应用**：记忆系统的 `MEMORY.md` 类似于 Doramagic 的"积木目录"——轻量可加载的索引，指向可按需展开的详细文件。

### 模式三：Fork Agent 后台提取

不阻塞主对话、共享 prompt cache、最大化缓存命中，是后台后处理任务的标准模式。关键实现细节：
- `skipTranscript: true` 避免写入主会话记录
- `maxTurns: N` 防止 Agent 陷入验证循环
- 游标机制（`lastMemoryMessageUuid`）确保每次只处理增量
- Stash + trailing run 确保在 Agent 繁忙时不丢失最新消息

### 模式四：新鲜度感知

记忆不是永久有效的事实，而是有时效性的观察。系统通过：
1. 在召回时附加"N 天前"的年龄提示
2. 在 system prompt 里植入防漂移指令（先验证再引用）
3. 要求 Agent 在发现过时记忆时主动更新而非保留

这对 Doramagic 的"知识提取"场景尤其相关——提取出的 WHY/UNSAID 会随项目演化而过时，需要类似机制维护新鲜度。

### 模式五：密钥扫描前置

在任何"跨边界"的写入（写入共享空间、网络上传）前，都应该扫描密钥。gitleaks 规则库提供了高置信度的模式集合，可以直接复用。关键设计：扫描在写入工具的 `validateInput` 阶段执行（而非事后），确保密钥不会触碰任何持久化路径。

---

## 9.11 源码索引

| 文件 | 行数 | 核心职责 |
|-----|------|---------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Session Memory 主逻辑：触发条件判断、Fork Agent 调用、手动触发 API |
| `services/SessionMemory/prompts.ts` | 324 | Session Memory 模板、更新 prompt 构建、section 大小分析 |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Session Memory 状态管理：配置、阈值判断、等待/同步工具函数 |
| `services/extractMemories/extractMemories.ts` | 615 | Persistent Memory 提取：Fork Agent 调用、闭包状态、并发控制 |
| `services/extractMemories/prompts.ts` | 154 | 提取 prompt 构建：auto-only 和 combined（含 Team Memory）两种变体 |
| `memdir/memdir.ts` | 507 | MEMORY.md 截断逻辑、记忆 prompt 构建、Directory 保证创建 |
| `memdir/paths.ts` | 278 | Auto Memory 路径解析、启用/禁用判断、路径安全校验 |
| `memdir/memoryTypes.ts` | 271 | 四种记忆类型定义、frontmatter 格式、召回/防漂移/不可派生原则 |
| `memdir/findRelevantMemories.ts` | 141 | Sonnet 召回选择：扫描 frontmatter → 5 条相关记忆 |
| `memdir/memoryScan.ts` | 94 | 目录扫描原语：读取 frontmatter、格式化清单 |
| `memdir/memoryAge.ts` | 53 | 新鲜度计算：天数、human-readable 文本、staleness 警告 |
| `memdir/teamMemPaths.ts` | 292 | Team Memory 路径、路径遍历防护（三层验证）、符号链接解析 |
| `memdir/teamMemPrompts.ts` | 100 | Team Memory + Auto Memory 合并 prompt 构建 |
| `services/teamMemorySync/index.ts` | 1256 | 同步核心：fetch/push 逻辑、乐观锁、批量分片、冲突重试 |
| `services/teamMemorySync/watcher.ts` | 387 | 文件监听：防抖 push、永久失败抑制、启动/停止生命周期 |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 种密钥扫描规则（gitleaks 子集）、redact 工具函数 |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema：TeamMemoryData、同步结果类型、SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | 写入前密钥拦截：FileWriteTool/FileEditTool validateInput 集成 |

**关键常量速查**：

| 常量 | 值 | 位置 |
|-----|---|------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH`（Session Memory 每节）| 2,000 tokens | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 tokens | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10,000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5,000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| 召回上限 | 5 条 | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| 记忆文件数上限 | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Frontmatter 读取行数 | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Team Memory 超时 | 30,000ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Push 防抖延迟 | 2,000ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| 单文件大小上限 | 250,000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| PUT 请求体上限 | 200,000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
| 密钥扫描规则数 | 30 | `secretScanner.ts:SECRET_RULES` |
