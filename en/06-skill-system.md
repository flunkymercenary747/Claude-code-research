# Chapter 6: Skill System

## 6.1 Overview and Positioning

The Skill system is one of the most innovative architectures in Claude Code. It encodes reusable workflows as Markdown files, triggered via slash commands (`/skill-name`) or by proactive AI invocation. Essentially, Skills are "SOPs for AI" — condensing the steps, decision points, and success criteria of human experts performing complex tasks in a structured Markdown format, giving AI reproducible professional execution capabilities.

Unlike ordinary Prompts, the Skill system has the following core characteristics:

1. **Declarative + imperative fusion**: Frontmatter declares metadata (permissions, model, triggers), body is execution instructions
2. **Multi-source loading**: bundled, user-level, project-level, Plugin-level, MCP sources, merged by priority
3. **Two execution modes**: inline (inject into current session context) and fork (isolated execution in independent sub-agent)
4. **Conditional activation**: automatically activated by file path via `paths` frontmatter
5. **Dynamic discovery**: during the session, as users operate files, automatically discover and load Skills from deeper directories

The Skill system is not a simple command alias, but a complete workflow orchestration framework.

---

## 6.2 Theoretical Foundations

### Design Pattern for Reusable Workflows

The Skill system solves a core problem in AI tool usage: **how can specialized knowledge be condensed and made reproducible?** Traditional code reuse is through functions and classes, but "knowledge" executed by AI is described in natural language workflows that cannot be directly encapsulated in code functions.

Skill design borrows from SOP (Standard Operating Procedure) thinking — structurally recording expert execution processes, decision points, and success criteria, so that AI follows the same high-quality path each time it executes.

### Declarative vs. Imperative Workflow Definition

The Skill system supports both styles simultaneously:

- **Declarative**: declare `allowed-tools`, `model`, `context`, and other properties via frontmatter, letting the system automatically handle permission control and execution context configuration
- **Imperative**: Skill body can embed shell commands (`` `!`command`` ``) to execute directly, implementing "operations interspersed in instructions"

### Markdown-as-Code Philosophy

Choosing Markdown rather than JSON/YAML as the Skill format is a carefully considered design decision:

- **Human readability**: developers can directly read and edit Skills, understanding their intent
- **AI-native friendliness**: AI training data extensively includes Markdown; AI's understanding of Markdown is more natural than JSON
- **Progressive structuring**: can start with pure prose, gradually adding headers, steps, rules, without forcing complete structure
- **Version control friendly**: Markdown diffs are human-friendly; workflow changes are immediately visible in code review

---

## 6.3 Skill Format and Data Structures

### Skill Markdown File Format Specification

Skill files follow a fixed directory structure:

```
.claude/skills/<skill-name>/SKILL.md
```

File format is frontmatter + Markdown body:

```markdown
---
name: my-skill
description: One-sentence description of what this Skill does
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  Use when the user wants to... For example: "cherry-pick to release", "hotfix".
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## Steps

### 1. First Step
Specific action...

**Success criteria**: checkpoints that prove this step is complete
```

### Frontmatter Field Details

Below are all fields parsed by the `parseSkillFrontmatterFields` function (`loadSkillsDir.ts:184`):

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name (can differ from directory name) |
| `description` | string | One-sentence description, shown in `/help` |
| `allowed-tools` | string[] | Tool whitelist, supports `Bash(git:*)` prefix patterns |
| `argument-hint` | string | Parameter hint when user triggers, e.g., `"<branch-name>"` |
| `arguments` | string[] | Parameter name list, for `$arg_name` variable substitution |
| `when_to_use` | string | Tells AI when to proactively call this Skill, includes trigger phrases |
| `version` | string | Skill version number |
| `model` | string | Model override, e.g., `opus`, `sonnet`; `inherit` means inherit |
| `disable-model-invocation` | boolean | Prevent AI from proactively calling; only user manual trigger |
| `user-invocable` | boolean | Whether visible in `/help` (default `true`) |
| `context` | `"fork"` | Execute in independent sub-agent when set |
| `agent` | string | Specify agent type |
| `effort` | EffortValue | Affects model thinking depth |
| `paths` | string[] | gitignore-syntax path patterns for conditional activation |
| `hooks` | HooksSettings | Hook configuration during Skill execution |
| `shell` | FrontmatterShell | Inline shell command execution configuration |

### SkillDefinition Type

`bundledSkills.ts` defines `BundledSkillDefinition` (lines 12-41), while filesystem Skills correspond to the `Command` type (`src/types/command.js`). Both converge into a unified `Command` object in `createSkillCommand` (`loadSkillsDir.ts:269`):

```typescript
// loadSkillsDir.ts:316-400
return {
  type: 'prompt',
  name: skillName,
  description,
  allowedTools,
  argumentHint,
  argNames: argumentNames.length > 0 ? argumentNames : undefined,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context: executionContext,
  agent,
  effort,
  paths,
  contentLength: markdownContent.length,
  isHidden: !userInvocable,
  progressMessage: 'running',
  loadedFrom,
  hooks,
  skillRoot: baseDir,
  async getPromptForCommand(args, toolUseContext) { ... }
} satisfies Command
```

---

## 6.4 Skill Loading Mechanism

### Complete Loading Flow of loadSkillsDir

`getSkillDirCommands` (`loadSkillsDir.ts:638`) is the entry point for the entire loading flow, using `lodash-es/memoize` to cache results, avoiding repeated I/O:

```
At startup
  ├── policySettings: ~/.claude-managed/.claude/skills/ (enterprise managed)
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/ (from cwd up to home)
  ├── additionalDirs: extra directories specified by --add-dir
  └── legacyCommands: .claude/commands/ (backward compatible)

During session (dynamic discovery)
  └── user reads/writes files → discoverSkillDirsForPaths() → addSkillDirectories()
```

Loading results are deduplicated via `realpath` (`loadSkillsDir.ts:728-763`), avoiding duplicate loading caused by symlinks.

### Multi-Source Loading Priority

Code comments explicitly state the loading priority (`loadSkillsDir.ts:677-714`):

```
managed (enterprise policy) < user (user level) < project (project level) < additional (--add-dir)
```

This is the "more specific = higher priority" principle: project-level overrides user-level, because projects have specific needs.

**Special cases:**
- `--bare` mode: skip auto-discovery, only load directories explicitly specified by `--add-dir`
- `skillsLocked` (plugin-only policy): prevent loading user/project-level Skills, only allow Plugin sources
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS` environment variable: skip managed-level Skills

### Skill Discovery and Matching Logic

**Static discovery** (at startup): `getSkillDirCommands` scans each level's `~/.claude/skills/` directory, only supporting directory format (`skill-name/SKILL.md`), not single `.md` files.

**Dynamic discovery** (during session): when users read/write files, `discoverSkillDirsForPaths` (`loadSkillsDir.ts:861`) walks up the file path, checking each directory for `.claude/skills/`; when found, loads via `addSkillDirectories`. Directories marked by `.gitignore` are skipped (preventing `node_modules` Skill pollution).

**Conditional activation** (`paths` frontmatter): Skills with `paths` fields are initially not visible to the model, stored in the `conditionalSkills` Map. When users operate files matching the path, `activateConditionalSkillsForPaths` (`loadSkillsDir.ts:997`) uses the `ignore` library (gitignore syntax) for matching; hits are moved to `dynamicSkills` for activation.

---

## 6.5 SkillTool Execution Flow

### Complete Path from /skill-name to Execution

`SkillTool` (`tools/SkillTool/SkillTool.ts:330`) is a standard `Tool` implementation; AI executes Skills by calling this tool. Complete execution path:

```
User types /skill-name or AI decides to call SkillTool
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── strip leading slash (compatibility handling)
  │     ├── check _canonical_ prefix (remote Skills, experimental)
  │     ├── findCommand() look up registered Command
  │     ├── check disableModelInvocation flag
  │     └── confirm type === 'prompt'
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── check deny rules
  │     ├── check remote canonical Skill (auto-allow)
  │     ├── check allow rules
  │     ├── skillHasOnlySafeProperties() → auto-allow safe Skills
  │     └── default: show user dialog (behavior: 'ask')
  │
  └── call (SkillTool.ts:580)
        ├── check context === 'fork' → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent() (independent sub-agent)
        └── otherwise (inline) → processPromptSlashCommand()
              └── inject newMessages + contextModifier into current session
```

### Skill Context Injection

In inline execution, `call` returns `newMessages` and `contextModifier` (`SkillTool.ts:767-840`):

- **newMessages**: message list expanded from the Skill, injected into current conversation context
- **contextModifier**: function to modify `ToolUseContext`, used for:
  - Stacking `allowedTools` (tool permissions declared by the Skill)
  - Overriding `mainLoopModel` (if Skill specifies a model)
  - Overriding `effortValue` (if Skill specifies effort)

Worth noting that `contextModifier` uses chained calls (`SkillTool.ts:777`), correctly handling multiple stacked contextModifiers rather than simply overriding.

### Skill Variable Substitution

`getPromptForCommand` in `createSkillCommand` (`loadSkillsDir.ts:343-398`) performs the following substitutions before returning Skill content:

1. **Parameter substitution**: `$arg_name` → `substituteArguments()` injects user-provided parameters
2. **Directory variable**: `${CLAUDE_SKILL_DIR}` → absolute path of the directory containing the Skill file
3. **Session ID**: `${CLAUDE_SESSION_ID}` → current session ID
4. **Shell command execution**: `` `!`command`` `` → result inlined (only for non-MCP Skills)

MCP Skills disable shell command execution (`loadSkillsDir.ts:372`), preventing remote untrusted Skills from injecting arbitrary shell commands.

### Skill Interaction with Tools

In forked execution mode (`executeForkedSkill`, `SkillTool.ts:121`), Skills run in a completely isolated sub-agent:

- Launched independently via `runAgent()` with an independent token budget
- Tool use messages during execution are reported via `onProgress` callbacks; UI can show progress
- Execution results extracted via `extractResultText`, returned to parent agent
- Memory released via `clearInvokedSkillsForAgent` (`SkillTool.ts:286`)

---

## 6.6 Bundled Skills Complete List and Analysis

Built-in Skills are registered via `registerBundledSkill()` (`bundledSkills.ts:55`), initialized when the CLI starts. Below is the analysis of all 17 built-in Skills:

### 1. `update-config` (`updateConfig.ts`, 475 lines)

**Function**: Configure Claude Code's `settings.json`, including Permissions, Hooks, Model, MCP, and all other configuration items.

**Characteristics**: Skill body is dynamically generated — uses `toJSONSchema(SettingsSchema())` to auto-generate JSON Schema documentation from Zod schema, ensuring documentation always stays in sync with actual types. Includes complete Hooks documentation (all Hook events, Hook types, JSON output formats).

**Trigger scenarios**: when users want to configure behavior automation, permission rules, environment variables, model settings.

### 2. `schedule` (`scheduleRemoteAgents.ts`, 447 lines)

**Function**: Manage remote scheduled Agents (cron triggers), create, update, list, run scheduled tasks.

**Characteristics**: Checks multiple prerequisites before calling (OAuth tokens, repository info, MCP connectors, cloud environments) and injects this dynamic information into the Skill prompt. Interacts with users via `AskUserQuestion` tool.

**Trigger scenarios**: when users want to create scheduled-running Claude Code agents (like daily code reviews, automated reports).

### 3. `keybindings-help` (`keybindings.ts`, 339 lines)

**Function**: Help users customize keyboard shortcuts, modify `~/.claude/keybindings.json`.

**Characteristics**: dynamically generates documentation via `generateContextsTable()`, `generateActionsTable()` from code constants, and lists non-rebindable shortcuts via `generateReservedShortcuts()` to prevent user mistakes.

**Trigger scenarios**: when users want to rebind shortcuts, add chord bindings, or modify the submit key.

### 4. `lorem-ipsum` (`loremIpsum.ts`, 282 lines)

**Function**: Generate a fixed number of single-token word placeholder text for token counting and performance testing.

**Characteristics**: Uses a list of API-validated single-token words, ensuring the `lorem` parameter can precisely control token count. Commonly used for benchmarking and token billing analysis.

**Trigger scenarios**: test text requiring a precise number of tokens.

### 5. `skillify` (`skillify.ts`, 197 lines)

**Function**: Automatically convert the current session's operation process into a reusable SKILL.md file.

**Characteristics**: This is the Skill system's "self-reproduction" mechanism. By reading session memory and user message history, guides users through 4 rounds of `AskUserQuestion` dialogue to confirm workflow name, steps, parameters, and trigger conditions, ultimately generating a standard-format SKILL.md and writing it to disk.

**Limitation**: only available to `USER_TYPE === 'ant'` (Anthropic internal employees).

**Trigger scenarios**: at session end, when users want to solidify just-completed operation flows into reusable Skills.

### 6. `claude-api` (`claudeApi.ts`, 196 lines + `claudeApiContent.ts`, 220 lines)

**Function**: Help developers build applications using the Claude API or Anthropic SDK.

**Characteristics**:
- Auto-detects current project language (by scanning file extensions, supports Python/TypeScript/Java/Go/Ruby/C#/PHP/curl)
- Lazy loading (247KB of `.md` content only loaded when called), avoiding impact on startup time
- Includes language-specific API documentation, Agent SDK patterns, streaming, etc.
- Writes documents to temporary directories via `files` mechanism; model can read on demand with Read/Grep tools

**Trigger scenarios**: code imports `anthropic` or user asks how to use the Claude API.

### 7. `batch` (`batch.ts`, 124 lines)

**Function**: Decompose large-scale code changes (migrations, refactoring, batch renames) into 5-30 parallel worktree agent executions.

**Characteristics**: Three-phase execution model — Plan (enter Plan Mode for deep research and decomposition) → Spawn Workers (parallel launch of background agents with `isolation: "worktree"`) → Track Progress (real-time status table rendering). Each worker operates in an independent git worktree, not affecting each other; creates PRs when complete.

**Trigger scenarios**: large-scale code migrations, full-repo refactoring, batch modifications.

### 8. `loop` (`loop.ts`, 92 lines)

**Function**: Repeat a prompt or slash command at fixed intervals.

**Characteristics**: intelligently parses time intervals (supports `5m`, `2h` prefix format and `every 20m` suffix format), converts to cron expressions, and calls `ScheduleCronTool` to register scheduled tasks. Executes once immediately after setup, without waiting for the first scheduled trigger.

**Trigger scenarios**: when users want to periodically check deployment status or run a Skill on a cycle.

### 9. `remember` (`remember.ts`, 82 lines)

**Function**: Review auto-memory entries, propose promoting them to `CLAUDE.md`, `CLAUDE.local.md`, or team memory.

**Characteristics**: Uses a "propose first, then confirm" principle — doesn't directly modify files, instead presents a classification report (to be promoted/to be cleaned/uncertain/no action needed), then executes after user approval. Distinguishes project-level conventions (CLAUDE.md), personal preferences (CLAUDE.local.md), and org-level knowledge (team memory).

**Limitation**: only available when `USER_TYPE === 'ant'` and auto-memory is enabled.

**Trigger scenarios**: when users want to organize memory and avoid unlimited auto-memory accumulation.

### 10. `simplify` (`simplify.ts`, 69 lines)

**Function**: Perform a three-dimensional code review on the current git diff (code reuse, code quality, efficiency) and directly fix the issues found.

**Characteristics**: simultaneously launches three parallel sub-agents, each responsible for:
- **Code reuse Agent**: finds reinventing the wheel, points to existing utility functions
- **Code quality Agent**: finds redundant state, parameter bloat, copy-paste, leaky abstractions, etc.
- **Efficiency Agent**: finds unnecessary computation, missing concurrency, N+1 patterns, memory leaks, etc.

After three agents complete, merges findings and directly fixes them rather than just reporting.

**Trigger scenarios**: quality review after completing a piece of code; also automatically called by the worker flow in the `batch` Skill.

### 11. `debug` (`debug.ts`)

**Function**: Diagnose debug logs from the current Claude Code session to help troubleshoot issues.

**Characteristics**: reads the last few lines of debug logs via tail (up to 64KB), avoiding memory spikes from oversized log files in long sessions. For non-Anthropic employees, enables debug logging first then reads. Marked `disableModelInvocation: true` to prevent AI from calling automatically (only user manual trigger).

### 12. `stuck` (`stuck.ts`)

**Function**: Diagnose other frozen or stuck Claude Code processes on the machine and send a report to a Slack channel.

**Characteristics**: Anthropic internal diagnostic tool. Detects high CPU (≥90% sustained), D state (I/O hanging), T state (Ctrl+Z stopped), Z state (zombie processes), high memory (≥4GB), and other anomalies. Uses two-message structure to send Slack reports (top-level summary + thread details).

### 13. `verify` (`verify.ts`)

**Function**: Verify code changes meet expectations by running the application.

**Characteristics**: reads Skill body from `verifyContent.ts` (SKILL.md parsing), writes auxiliary files to temporary directories via `files` mechanism. Only available to `USER_TYPE === 'ant'`.

### 14. `claudeInChrome` (`claudeInChrome.ts`)

**Function**: Launch a headless session connected to a real Chrome browser with the Side Panel extension, allowing Claude to control the browser in real time.

### 15. `claudeCodeGuide` (embedded in the `AgentTool` system)

Used for Claude Code internal onboarding flows.

---

## 6.7 Relationship Between Skills and Commands

### Boundary Between the Two

In Claude Code's design, Skills and Commands were once different concepts but are now unified:

- **Historically**: the `/commands/` directory stored simple prompt commands (`.md` files); the `/skills/` directory stored more complex, directory-structured workflows (`skill-name/SKILL.md`)
- **Now**: both are loaded by `loadSkillsDir.ts`, uniformly converted to `Command` type; `/commands/` is marked `loadedFrom: 'commands_DEPRECATED'` (`loadSkillsDir.ts:608`)

Current actual differences are only in the loading path:
- `/skills/skill-name/SKILL.md`: new format, recommended; supports `baseDir` (Skills can carry auxiliary files)
- `/commands/skill-name.md` or `/commands/skill-name/SKILL.md`: old format, backward compatible

### When to Use a Skill vs. a Command

| Scenario | Recommended Approach |
|----------|---------------------|
| Multi-file workflow (Skill with auxiliary resource files) | `/skills/` directory format |
| Simple prompt reuse (a single md file suffices) | still usable with `/commands/` (compatible) |
| Need `${CLAUDE_SKILL_DIR}` variable | must use `/skills/` directory format |
| Need `files:` embedded resources (bundled skill) | `BundledSkillDefinition.files` |
| Built into CLI binary | `registerBundledSkill()` |

---

## 6.8 Design Decision Analysis

### Why Markdown Instead of JSON/YAML

Skill execution instructions (body) are written in natural language for AI to understand and follow. JSON/YAML can only encode structured data; they cannot directly express complex instructions like "first search for related files, then analyze dependencies, be careful not to modify test files."

Markdown balances both: frontmatter (YAML) handles structured metadata; body (Markdown) handles human-readable execution instructions. This is a pragmatic format choice.

### Skill Permission Control

Permission control uses a "whitelist + ask" mechanism (`SkillTool.ts:871-900`):

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // CommandBase properties...
  'name', 'description', 'hasUserSpecifiedDescription', ...
  // NOT included: 'allowedTools', 'hooks', 'paths', etc.
])
```

`skillHasOnlySafeProperties()` checks whether a Skill only uses "safe properties" — if the Skill doesn't declare `allowedTools`, `hooks`, `paths`, or other sensitive properties, it's automatically allowed without user confirmation. This is good security design: newly added properties are unsafe by default and must be explicitly reviewed before being added to the whitelist.

### Secure File Write Mechanism

Built-in Skills embed auxiliary files via the `files` field; writes to disk use strict security measures (`bundledSkills.ts:171-194`):

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

Uses `O_NOFOLLOW | O_EXCL` to prevent symlink attacks; file permissions are `0o600` (owner read/write only). The write directory contains a random nonce per process startup, preventing path prediction attacks.

### MCP Skill Integration Strategy

MCP Skills implement an elegant dependency inversion via `mcpSkillBuilders.ts` (`mcpSkillBuilders.ts:1-43`):

MCP discovery logic (`mcpSkills.ts`) needs to use `createSkillCommand` and `parseSkillFrontmatterFields`, but direct imports would cause circular dependencies. The solution:

1. `loadSkillsDir.ts` calls `registerMCPSkillBuilders()` at module initialization to register these two functions
2. `mcpSkills.ts` retrieves them when needed via `getMCPSkillBuilders()`

This design also solves a Bun bundling technical limitation: in Bun bundles, dynamic imports with variable (non-literal) paths cannot be resolved, so `await import(variable)` can't be used; only this registry pattern works.

---

## 6.9 Transferable Patterns

### Doramagic Skill System Comparison

| Dimension | Claude Code Skill | Doramagic Skill |
|-----------|------------------|-----------------|
| File format | `SKILL.md` (Markdown + YAML frontmatter) | `SKILL.md` (same format) |
| Directory structure | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| Execution engine | SkillTool (AI tool call) | OpenClaw tool call |
| Source priority | policy < user < project < plugin | OpenClaw platform rules |
| Built-in Skills | 15+, compiled to binary | under construction |
| Parameter substitution | `$arg_name`, frontmatter `arguments` | same mechanism |
| Execution context | inline / fork (sub-agent) | inline (current stage) |
| Conditional activation | `paths` frontmatter | not yet implemented |
| Dynamic discovery | file operations trigger auto-discovery | not yet implemented |

### Core Borrowable Patterns

**1. `skillify` pattern: workflow self-reproduction**

Claude Code's `skillify` Skill is an extremely elegant design — having AI analyze just-executed operations and guide users through a dialogue to solidify them into reusable Skills. Doramagic can similarly implement a `/dora-skillify` to solidify a successful knowledge extraction process into a project-specific Skill.

**2. AI proactive invocation mechanism of `when_to_use`**

The `when_to_use` frontmatter field lets AI know when to proactively call a Skill without users explicitly entering a slash command. Doramagic's Skills should also focus on this field, enabling knowledge extraction to be automatically triggered at appropriate moments.

**3. Dynamic skill discovery and conditional activation**

The mechanism of activating Skills by file path is very suitable for Doramagic's project-specific knowledge scenarios: when users operate files in a certain domain, automatically activate the corresponding domain's extraction Skill (e.g., activating a frontend architecture analysis Skill when operating TypeScript files).

**4. `files` mechanism for auxiliary resource management**

Built-in Skills embed reference documents and example code in Skills via the `files` field; the model reads them on-demand rather than injecting them all into context at once. Doramagic's large Skills (like Soul Extractor) can adopt this pattern to manage extraction templates and reference materials.

**5. Security model: allowedTools whitelist + auto-allow safe Skills**

Skills can only use tools declared in frontmatter. Claude Code further distinguishes "safe Skills" (no special permissions) from "confirmation-required Skills" (with allowedTools/hooks), automatically allowing the former to reduce friction. This permission model is worth the OpenClaw platform borrowing.

---

## 6.10 Source Index

| File | LOC | Role |
|------|-----|------|
| `skills/loadSkillsDir.ts` | 1,087 | Skill loading core: discovery, parsing, deduplication, conditional activation, dynamic discovery |
| `skills/bundledSkills.ts` | 220 | Built-in Skill registry, file extraction, secure writes |
| `tools/SkillTool/SkillTool.ts` | 1,108 | Skill execution tool: validation, permissions, inline/fork execution |
| `skills/mcpSkillBuilders.ts` | 44 | MCP Skill builder registry (breaking circular dependencies) |
| `skills/bundled/updateConfig.ts` | 475 | update-config: settings.json configuration helper |
| `skills/bundled/scheduleRemoteAgents.ts` | 447 | schedule: scheduled remote agent management |
| `skills/bundled/keybindings.ts` | 339 | keybindings-help: keyboard shortcut configuration |
| `skills/bundled/loremIpsum.ts` | 282 | lorem-ipsum: placeholder text with precise token count |
| `skills/bundled/skillify.ts` | 197 | skillify: auto-solidify session workflow as Skill |
| `skills/bundled/claudeApi.ts` | 196 | claude-api: Claude API development helper (multi-language) |
| `skills/bundled/claudeApiContent.ts` | 220 | claude-api's 247KB documentation content (inlined at build) |
| `skills/bundled/batch.ts` | 124 | batch: large-scale parallel worktree changes |
| `skills/bundled/loop.ts` | 92 | loop: repeat prompt execution at intervals |
| `skills/bundled/remember.ts` | 82 | remember: memory review and promotion |
| `skills/bundled/simplify.ts` | 69 | simplify: three-dimensional code review and fix |
| `skills/bundled/debug.ts` | ~60 | debug: session debug log diagnosis |
| `skills/bundled/stuck.ts` | ~60 | stuck: process freeze diagnosis + Slack report |
| `skills/bundled/verify.ts` | ~30 | verify: run app to verify code changes |
| `skills/bundled/claudeInChrome.ts` | ~40 | claude-in-chrome: Chrome browser control |
| `skills/bundled/index.ts` | - | Registration entry for all built-in Skills |

**Key function index:**

| Function | File:Line | Description |
|----------|-----------|-------------|
| `getSkillDirCommands` | `loadSkillsDir.ts:638` | Main loading entry (memoized) |
| `parseSkillFrontmatterFields` | `loadSkillsDir.ts:184` | Frontmatter field parsing |
| `createSkillCommand` | `loadSkillsDir.ts:269` | Build Command object |
| `loadSkillsFromSkillsDir` | `loadSkillsDir.ts:407` | Load from `/skills/` directory |
| `discoverSkillDirsForPaths` | `loadSkillsDir.ts:861` | Dynamically discover Skill directories |
| `activateConditionalSkillsForPaths` | `loadSkillsDir.ts:997` | Conditional Skill activation |
| `registerBundledSkill` | `bundledSkills.ts:55` | Register built-in Skill |
| `executeForkedSkill` | `SkillTool.ts:121` | Fork mode execution |
| `skillHasOnlySafeProperties` | `SkillTool.ts:871+` | Safe Skill determination |
| `registerMCPSkillBuilders` | `mcpSkillBuilders.ts:31` | MCP builder registration |
