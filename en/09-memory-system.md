# Chapter 9: Memory System

## 9.1 Overview and Purpose

Claude Code's memory system is one of the most sophisticated and deeply engineered subsystems in the entire toolchain. It addresses the most fundamental limitation of LLMs: the context window resets to zero at the end of each session. Every time a user starts a new session, Claude faces a blank slate — unaware of who the user is, their preferences, past mistakes, or team conventions.

The memory system's design goal is: **to let Claude maintain continuity across sessions and act like a genuine long-term collaborator.**

In terms of source code volume, this is a substantial system:
- `memdir/` directory: 7 files, 1,736 lines
- `services/SessionMemory/`: 3 files, 1,026 lines
- `services/extractMemories/`: 2 files, 769 lines
- `services/teamMemorySync/`: 5 files, 2,167 lines

Total approximately 5,700 lines — about 1.1% of the entire codebase — yet its complexity and design thinking density far exceed that proportion.

---

## 9.2 Theoretical Foundation

### Mapping to the Human Memory Model

The system architecture explicitly maps to three types of memory from cognitive science:

| Human Memory | Claude Code Equivalent | Technical Implementation |
|-------------|----------------------|------------------------|
| Working Memory | Current context window | Session message list, cleared when session ends |
| Episodic Memory | Session Memory | `~/.claude/projects/<slug>/session-memory.md`, continuously updated within the session |
| Semantic Memory | Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`, preserved long-term across sessions |

Session Memory corresponds to "in-the-moment recall" — recording what this session is working on and where it stands; Persistent Memory corresponds to "accumulated knowledge" — user preferences, lessons learned, project background.

### Knowledge Graph vs. Document Memory

The system chose **Markdown documents on the filesystem** over a database or vector index. This choice is explicitly stated in comments in `memoryTypes.ts`:

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

This reveals a first-principles decision: **information that can be queried in real time should not be memorized.** Memory only stores "non-derivable" context — user preferences, team historical lessons, the motivation behind project decisions. This is fundamentally different from a knowledge graph design, which tends to structure everything it can.

### Eventual Consistency in Memory

The Team Memory sync design explicitly adopts eventual consistency semantics:

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

The decision that deletions do not propagate is intentional — team memory is an "append-only" asset, and accidental deletion should not become a permanent loss. This is a conservative implementation of eventual consistency principles from distributed systems.

---

## 9.3 Three-Layer Memory Architecture

The system consists of three layers, ordered from shortest to longest lifecycle:

### Layer 1: Session Memory (Session-Level)

**File path**: `~/.claude/projects/<sanitized-cwd>/session-memory.md` (accessed via `getSessionMemoryPath()`)

Session Memory is a Markdown file **continuously maintained within the current session**, with a fixed content structure:

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

(`services/SessionMemory/prompts.ts:14-36`, `DEFAULT_SESSION_MEMORY_TEMPLATE`)

It is not cleared when the session ends; instead, it is read by the Auto Compact mechanism when compressing the context, and injected as a "previously on" summary into the new context window.

**Data Structure Constraints**:
- Each section is limited to 2,000 tokens (`MAX_SECTION_LENGTH = 2000`)
- Full document is limited to 12,000 tokens (`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`)
- When limits are exceeded, the system adds a warning in the prompt requiring the Agent to compress

**Lifecycle**: Bound to the current project session, read when Auto Compact is triggered

### Layer 2: Persistent Memory (Cross-Session Persistent Memory)

**File path**: `~/.claude/projects/<sanitized-git-root>/memory/`

This is the core long-term memory layer. Each memory is stored independently as a `.md` file with YAML frontmatter:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

(`memdir/memoryTypes.ts:230-240`, `MEMORY_FRONTMATTER_EXAMPLE`)

Path resolution logic is handled by `getAutoMemPath()` (`memdir/paths.ts:173-190`), with the following priority:

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` environment variable (used in Cowork multi-user scenarios)
2. `autoMemoryDirectory` in `settings.json` (only trusted from policy/local/user sources — **not** projectSettings to prevent malicious repo hijacking write paths)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` (default)

Git worktrees are unified to the canonical git root (`findCanonicalGitRoot`), ensuring different worktrees of the same repository share the same memory.

**Lifecycle**: Permanent, until the user explicitly deletes it or the Agent actively updates/deletes it

### Layer 3: Team Memory (Shared Team Memory)

**File path**: `~/.claude/projects/<sanitized-git-root>/memory/team/` (returned by `getTeamMemPath()`)

Team Memory is a subdirectory of Persistent Memory, synchronized via REST API among all authenticated members of the same GitHub repository. It is an extension built on top of Auto Memory; `isTeamMemoryEnabled()` first checks `isAutoMemoryEnabled()` to ensure the parent system is enabled.

**Lifecycle**: Maintained by Anthropic's servers, persistent across users and machines

---

## 9.4 MEMORY.md Index Mechanism

MEMORY.md is the **index file** of the Persistent Memory layer, not a content file. The system explicitly distinguishes between the two in multiple places:

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### Format Specification

Each line in MEMORY.md is a link pointing to a specific memory file:

```
- [User Preference: Concise Responses](feedback_terse_responses.md) — user dislikes summaries at the end of replies
- [Project Background: Auth Middleware Rewrite](project_auth_rewrite.md) — legal compliance requirement, not technical debt
```

MEMORY.md is loaded into the system prompt at the start of each session, so its size directly affects the token consumption of every request.

### Dual Limit of 200 Lines / 25KB

The system defines strict dual limits in `memdir/memdir.ts`:

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

(`memdir/memdir.ts:30-33`)

Truncation logic is implemented in `truncateEntrypointContent()` (`memdir/memdir.ts:55-102`): first truncates by line count, then by byte count (cutting at the most recent newline to avoid truncating in the middle of a line). After truncation, a warning message is appended notifying the user that the index is too long.

**Design Intent**: ~125 chars/line × 200 lines ≈ 25KB, a reasonable upper bound validated by real-world measurements (p97). The byte limit handles edge cases of "fewer than 200 lines but each line extremely long" (observed p100: 197KB without exceeding the line limit).

### Relationship with Memory Files

Writing a memory is a **two-step operation**:
1. Write the content file (`user_role.md`, `feedback_testing.md`, etc.)
2. Add a pointer entry in MEMORY.md

When reading, only files selected by `findRelevantMemories` are read (see 9.7); MEMORY.md itself is always present in the system prompt.

---

## 9.5 Four Memory Types

The system constrains all memories to four types — one of the most important design decisions. Types are defined in `memdir/memoryTypes.ts` (the `MEMORY_TYPES` constant):

### user Type

**Applicable scenarios**: The user's role, goals, responsibilities, and knowledge background

**Trigger timing**: Whenever the user's role, preferences, responsibilities, or knowledge level is learned

**Purpose**: Adapting response style to fit the specific user's cognitive level and needs

**Scope**: Always private (personal), even in Team Memory mode

**Negative examples (what should not be saved)**:
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### feedback Type

**Applicable scenarios**: User corrections and confirmations of working style — both "don't do this" and "keep doing this"

**Structure requirements**:
- The rule itself
- A `**Why:**` line (providing reasoning, to help judge whether it applies in edge cases)
- A `**How to apply:**` line (when and where it takes effect)

**Unique design**: Explicitly requires recording both **failure lessons and success confirmations**:

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**Trigger timing**: When the user says "don't do this" (explicit correction) or "exactly" / "perfect" (implicit confirmation, harder to recognize)

**Scope**: Default private; only save as team when the guidance is clearly a project-level standard (e.g., testing strategy, build constraints)

### project Type

**Applicable scenarios**: Information about ongoing work, goals, plans, bugs, or events that **cannot be derived from code or git history**

**Structure requirements**:
- The fact/decision itself
- A `**Why:**` line (motivation — typically constraints, deadlines, or stakeholder needs)
- A `**How to apply:**` line (how it influences recommendations)

**Important rule**: When saving, relative dates must be converted to absolute dates ("next Thursday" → "2026-04-08"), ensuring memories remain interpretable as time passes.

**Scope**: Default team (project context is inherently shared)

**Decay characteristic**: project type memories decay fastest; the Why field helps judge whether a memory is still valid.

### reference Type

**Applicable scenarios**: Pointers to the location of information in external systems (Linear projects, Slack channels, Grafana dashboards, etc.)

**Trigger timing**: When learning the location and purpose of external resources

**Scope**: Usually team

**Typical example**:

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### What Should Not Be Saved (Explicitly Excluded)

`WHAT_NOT_TO_SAVE_SECTION` explicitly lists six categories of content that should not be saved (`memdir/memoryTypes.ts:196-207`):

1. Code patterns, conventions, architecture, file paths — derivable from the current project state
2. Git history, recent changes — `git log`/`git blame` are authoritative sources
3. Debug solutions or fixes — the fix is in the code, context is in the commit message
4. Content already documented in CLAUDE.md
5. Temporary task details: work in progress, temporary state, current session context
6. **Even if the user explicitly asks to save the above content** — if the user asks to save a PR list, ask "is there something surprising or non-obvious? That's what's worth saving"

---

## 9.6 Automatic Memory Extraction

### Fork Agent Automatic Extraction Mechanism

Memory extraction uses the "Fork Agent" pattern — creating an Agent context identical to the main session, running asynchronously in the background without blocking the main conversation flow.

The core of this mechanism is `runForkedAgent()`. The extraction Agent shares the parent session's prompt cache, achieving maximum cache hit rate:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // not written to main session record, avoids race conditions
  maxTurns: 5,            // hard cap prevents verification infinite loops
})
```

(`services/extractMemories/extractMemories.ts:258-267`)

The `maxTurns: 5` design comment explains the intent:

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

The extraction Agent's efficient strategy is explicitly designed as "complete in 2 turns":
- **Turn 1**: Emit all FileRead requests for files needing updates in parallel
- **Turn 2**: Emit all FileWrite/FileEdit requests in parallel

### Trigger Timing (Stop Hooks)

Extraction is triggered **after each complete query loop ends** — when the model produces a final response with no tool_use — via `handleStopHooks` calling `executeExtractMemories()`.

State is managed through closures, with key variables including:

```typescript
let lastMemoryMessageUuid: string | undefined    // cursor: where last extraction stopped
let inProgress = false                           // prevents concurrent runs
let pendingContext: {...} | undefined            // calls arriving during a run are stored here
let turnsSinceLastExtraction = 0                // for throttling control
```

(`services/extractMemories/extractMemories.ts:225-240`)

**Concurrency control strategy**: If a new call arrives while extraction is in progress, the new call is "stashed" (stored in `pendingContext`) rather than discarded. When the current extraction completes, a "trailing extraction" runs immediately with the latest context, ensuring the last batch of messages is not missed.

**Mutual exclusion rule**: If the main Agent has itself written memory files (detected by `hasMemoryWritesSince`), the Fork Agent skips this extraction and only advances the cursor:

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // main Agent wrote, skip fork agent, advance cursor
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

(`services/extractMemories/extractMemories.ts:198-209`)

### Extraction Prompt Analysis

The core design philosophy of the extraction prompt is **information efficiency**:

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // pre-inject existing memory list, avoids Agent spending a turn on ls
  ].join('\n')
}
```

(`services/extractMemories/prompts.ts:20-47`)

Pre-injecting the existing memory manifest (`existingMemories`) is a key optimization — avoiding the Agent wasting a turn to list the directory, directly providing a structured file list (filename, type, timestamp, description) in the prompt.

### Session Memory Trigger Mechanism

Session Memory uses a different trigger mechanism — via `postSamplingHooks` rather than Stop Hooks — evaluating after each model sample whether an update is needed:

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

(`services/SessionMemory/sessionMemory.ts:130-150`)

Default trigger thresholds (`DEFAULT_SESSION_MEMORY_CONFIG`, `services/SessionMemory/sessionMemoryUtils.ts:29-33`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `minimumMessageTokensToInit` | 10,000 | Minimum token count required to initialize session memory |
| `minimumTokensBetweenUpdate` | 5,000 | Minimum token growth required between two updates |
| `toolCallsBetweenUpdates` | 3 | Minimum number of tool calls required between two updates |

These values can be dynamically adjusted via GrowthBook remote configuration (`tengu_sm_config`).

---

## 9.7 Intelligent Memory Recall

### Sonnet Selects Up to 5 Relevant Memories

Memory recall is not a full read — it **first scans frontmatter, then uses Sonnet to select at most 5 of the most relevant entries**.

The core flow is in `findRelevantMemories()` (`memdir/findRelevantMemories.ts:32-66`):

1. `scanMemoryFiles()` scans the memory directory, reads the first 30 lines (frontmatter) of each file, and returns `MemoryHeader[]`
2. Filters out memories already surfaced in previous turns (`alreadySurfaced`), saving 5 slots for new content
3. Uses Sonnet to call `selectRelevantMemories()`, selecting the most relevant filenames based on the query and file descriptions
4. Returns the paths and mtimes of the selected memories

### Relevance Judgment Logic

Sonnet's system prompt is carefully designed:

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

(`memdir/findRelevantMemories.ts:13-23`)

**Key design**: Reference documentation for recently used tools should not be selected (no need to reference docs while actively using them), but memories about **pitfalls/known issues** for the same tools should still be selected (pitfall warnings are most needed precisely when actively using the tool).

The API call uses structured output (JSON Schema) to ensure the returned format is parseable:

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

(`memdir/findRelevantMemories.ts:97-108`)

### How Memories Are Injected into Context

Selected memories are injected before user messages wrapped in `<system-reminder>` tags (`wrapMessagesInSystemReminder`). Memories older than 1 day get a freshness warning appended:

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

(`memdir/memoryAge.ts:38-47`)

This design addresses a real problem: users reported "making confident assertions based on stale memories" — referenced file paths or function names had been modified, but the citation in the memory made the assertion appear more credible rather than more suspicious.

**Anti-drift mechanism**: `MEMORY_DRIFT_CAVEAT` is injected into the system prompt, requiring the Agent to verify current state before answering based on memories:

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Team Memory Sync

### REST API Sync Mechanism

Team Memory implements server-side synchronization through `services/teamMemorySync/`, with the API design fully described at the top of `index.ts`:

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → metadata+hashes only
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert entries
404  = no data yet
```

(`services/teamMemorySync/index.ts:10-13`)

Sync depends on **OAuth authentication** (requires `CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`), with GitHub repositories (`owner/repo`) as the scope.

**Watcher mechanism**: `watcher.ts` uses `fs.watch({recursive: true})` to monitor team directory changes, triggering a push after a 2-second debounce (`DEBOUNCE_MS = 2000`). Native `fs.watch` is deliberately chosen over chokidar:

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS uses FSEvents (O(1) file descriptors), Linux uses inotify (O(number of subdirectories)), both superior to chokidar's kqueue approach.

### Optimistic Locking (If-Match)

Uploads use optimistic concurrency control, carrying ETags (checksums) in `If-Match` HTTP headers:

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

(`services/teamMemorySync/index.ts:uploadTeamMemory`)

When the server returns 412 Precondition Failed, it indicates a conflict (another user modified the shared memory in the meantime). The system uses the `GET ?view=hashes` endpoint (lightweight, returning only the SHA-256 hash for each key without content body) to refresh the local `serverChecksums`, then recalculates the delta and retries:

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### Conflict Resolution Strategy

The conflict resolution strategy is **server wins (server wins per-key)** — Pull overwrites local content with server content. Delta push only uploads keys where local differs from the server hash; the server uses upsert semantics (keys absent from the PUT are preserved).

Batch upload limits (`MAX_PUT_BODY_BYTES = 200_000`) prevent request bodies from being rejected by the API Gateway (observed: gateway returns HTML-formatted 413 at ~256-512KB, different from the application-layer structured 413). When the limit is exceeded, the upload is automatically split into multiple sequential PUTs; upsert semantics guarantee safety:

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // greedy bin packing: batch by byte count, each batch not exceeding MAX_PUT_BODY_BYTES
  ...
}
```

(`services/teamMemorySync/index.ts:batchDeltaByBytes`)

**Permanent failure suppression**: Some errors (no_oauth, no_repo, 4xx non-409/429) cannot be self-healed through retries. When the system detects such errors, it sets `pushSuppressedReason`, preventing the watcher-triggered push from falling into an infinite retry loop (one device without OAuth was observed sending 167K push events in 2.5 days).

---

## 9.9 Design Decision Analysis

### Why Use the Filesystem Instead of a Database

The filesystem + Markdown design has several key advantages:

1. **Agent can directly operate on it**: FileRead/FileWrite/FileEdit tools are Claude's native tools; no additional API layer is needed. Writing memory and writing code use the same toolset, minimizing cognitive burden.

2. **User-inspectable**: `~/.claude/projects/.../memory/` is an ordinary folder; users can directly `ls`, `cat`, `vim` it — fully transparent.

3. **Git-friendly**: Markdown files natively support diff, grep, git history, convenient for Team Memory delta calculation.

4. **Avoids unnecessary abstraction**: Databases require schema migrations, backup strategies, and query layers — over-engineering for "a few hundred KB of Markdown files."

### Why Limit MEMORY.md Size

The 200-line / 25KB limit is backed by real measurement data (p97/p100 observed values). Core reasons:

- MEMORY.md is loaded into the system prompt on **every request**, so its size directly affects token consumption
- An oversized index squeezes out space for genuinely useful context
- The forced limit encourages users and Agents to keep the index concise, writing only "hooks" per line rather than full content

This is a classic "use constraints to promote quality" design — not because it's technically impossible to accommodate more, but because constraints guide correct usage patterns.

### Memory Security Design Considerations

The system has multiple layers of security design:

**Path traversal protection**: `teamMemPaths.ts` implements three-layer checks — first string-level checks for `..`, URL-encoded traversal, and Unicode normalization attacks, then resolves symlinks via `realpath` to verify actual filesystem paths:

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

(`memdir/teamMemPaths.ts:130-133`)

**Secret scanning**: When writing to Team Memory, `scanForSecrets()` scans for 30 high-confidence credential patterns (from the gitleaks ruleset), including token formats for major platforms like AWS, GCP, GitHub, Anthropic, and OpenAI. Scanning is performed **double**: both before upload and before write:

- `teamMemSecretGuard.ts`'s `checkTeamMemSecrets()` intercepts writes at the `validateInput` stage of FileWriteTool/FileEditTool
- `readLocalTeamMemory()` scans again before pushing, skipping files containing sensitive information

**Minimum privilege tool control**: The extraction Agent's `canUseTool` function only allows:
- FileRead/Grep/Glob (read-only)
- Read-only Bash commands (ls/find/cat/stat/wc/head/tail)
- FileEdit/FileWrite with paths confined to the memory directory

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

(`services/extractMemories/extractMemories.ts:171-176`)

**ProjectSettings security exemption**: The `autoMemoryDirectory` setting only trusts policy/local/user sources, explicitly excluding projectSettings (`.claude/settings.json`):

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 Transferable Patterns

The following patterns from Claude Code's memory system design can be directly borrowed for Doramagic:

### Pattern 1: Non-Derivable Principle

**What should be memorized**: Any information that can be obtained by querying the current state (code, files, git) is not worth memorizing. Memory should only store "historical context" — why a decision was made, what pitfalls were encountered, the user's implicit preferences.

**Doramagic application**: The "UNSAID" and "WHY" layers extracted by Soul Extractor naturally conform to this principle. OpenClaw rule documentation is queryable and does not need to be memorized; but lessons like "this OpenClaw rule once caused a publish failure" are exactly what's worth memorizing.

### Pattern 2: Two-Step Write + Lightweight Index

The two-step write pattern of file + index ensures the index is always concise (forced constraint of 150 chars per line max), while content files can elaborate in detail. Index token consumption is fixed; content reading is on-demand.

**Doramagic application**: The memory system's `MEMORY.md` is similar to Doramagic's "brick catalog" — a lightweight loadable index pointing to detailed files that can be expanded on demand.

### Pattern 3: Fork Agent Background Extraction

Non-blocking main conversation, shared prompt cache, maximized cache hit rate — this is the standard pattern for background post-processing tasks. Key implementation details:
- `skipTranscript: true` avoids writing to the main session record
- `maxTurns: N` prevents the Agent from falling into a verification loop
- Cursor mechanism (`lastMemoryMessageUuid`) ensures only incremental processing each time
- Stash + trailing run ensures latest messages are not missed when the Agent is busy

### Pattern 4: Freshness-Aware Recall

Memories are not permanently valid facts but time-bounded observations. The system handles this through:
1. Appending an "N days ago" age hint at recall time
2. Planting anti-drift instructions in the system prompt (verify before citing)
3. Requiring Agents to actively update rather than retain stale memories when discovered

This is especially relevant to Doramagic's "knowledge extraction" scenario — extracted WHY/UNSAID will become stale as projects evolve, requiring a similar mechanism to maintain freshness.

### Pattern 5: Pre-Execution Secret Scanning

Before any "cross-boundary" write (writing to shared space, network upload), secrets should be scanned. The gitleaks ruleset provides a high-confidence pattern collection that can be directly reused. Key design: scanning is performed at the `validateInput` stage of write tools (not after the fact), ensuring secrets never touch any persistence path.

---

## 9.11 Source Index

| File | Lines | Core Responsibility |
|------|-------|-------------------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Session Memory main logic: trigger condition evaluation, Fork Agent calls, manual trigger API |
| `services/SessionMemory/prompts.ts` | 324 | Session Memory template, update prompt construction, section size analysis |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Session Memory state management: configuration, threshold evaluation, wait/sync utilities |
| `services/extractMemories/extractMemories.ts` | 615 | Persistent Memory extraction: Fork Agent calls, closure state, concurrency control |
| `services/extractMemories/prompts.ts` | 154 | Extraction prompt construction: auto-only and combined (with Team Memory) variants |
| `memdir/memdir.ts` | 507 | MEMORY.md truncation logic, memory prompt construction, directory creation guarantee |
| `memdir/paths.ts` | 278 | Auto Memory path resolution, enable/disable checks, path security validation |
| `memdir/memoryTypes.ts` | 271 | Four memory type definitions, frontmatter format, recall/anti-drift/non-derivable principles |
| `memdir/findRelevantMemories.ts` | 141 | Sonnet recall selection: scan frontmatter → 5 relevant memories |
| `memdir/memoryScan.ts` | 94 | Directory scan primitives: read frontmatter, format manifest |
| `memdir/memoryAge.ts` | 53 | Freshness calculation: days, human-readable text, staleness warnings |
| `memdir/teamMemPaths.ts` | 292 | Team Memory paths, path traversal protection (three-layer validation), symlink resolution |
| `memdir/teamMemPrompts.ts` | 100 | Team Memory + Auto Memory combined prompt construction |
| `services/teamMemorySync/index.ts` | 1,256 | Sync core: fetch/push logic, optimistic locking, batch splitting, conflict retry |
| `services/teamMemorySync/watcher.ts` | 387 | File watching: debounced push, permanent failure suppression, start/stop lifecycle |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 secret scanning rules (gitleaks subset), redact utilities |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema: TeamMemoryData, sync result types, SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | Pre-write secret interception: FileWriteTool/FileEditTool validateInput integration |

**Key Constants Reference**:

| Constant | Value | Location |
|----------|-------|----------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH` (Session Memory per section) | 2,000 tokens | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 tokens | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10,000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5,000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| Recall limit | 5 entries | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| Max memory files | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Frontmatter read lines | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Team Memory timeout | 30,000ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Push debounce delay | 2,000ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| Single file size limit | 250,000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| PUT request body limit | 200,000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
