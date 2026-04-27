# Chapter 11: MCP Integration

## 11.1 Overview and Purpose

### What Is MCP

MCP (Model Context Protocol) is an open protocol designed and led by Anthropic, defining a standardized communication format between AI applications and external tool services. At its core, it is a JSON-RPC 2.0 protocol running over multiple transport layers (stdio, SSE, HTTP Streamable, WebSocket), specifying standard message formats for tool discovery (`tools/list`), tool invocation (`tools/call`), resource management (`resources/list`/`resources/read`), and Prompt templates (`prompts/list`/`prompts/get`).

### MCP's Role in Claude Code

Claude Code's built-in toolset (Bash, Read, Edit, etc.) covers file system and local development scenarios. MCP is designed as an **open tool extension interface**: any third-party service (Slack, GitHub, Jira, databases, browser automation, etc.) can implement an MCP server, and Claude Code connects via the standard protocol to invoke these external capabilities without modifying core code.

Architecturally, Claude Code is a pure **MCP client** and does not implement any MCP server capabilities (except responding to `roots/list` requests to inform servers of the working directory). Each connected MCP server's tools are dynamically registered as Tool objects in the format `mcp__<serverName>__<toolName>`, sharing the same execution framework as built-in tools.

### Code Scale

MCP integration involves approximately 12,310 lines of TypeScript code, distributed across the following files:

| File | Lines | Responsibility |
|------|-------|---------------|
| `services/mcp/client.ts` | 3,348 | Connection management, tool discovery, execution core |
| `services/mcp/config.ts` | 1,578 | Configuration management (multi-source merging, policy filtering) |
| `services/mcp/auth.ts` | 2,465 | OAuth 2.0 authentication (including XAA cross-app access) |
| `services/mcp/utils.ts` | 575 | Tool filtering, name hashing, stale detection |
| `services/mcp/types.ts` | 258 | Type definitions (Transport, ServerConfig, connection state) |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | UI collapse classification (Search/Read tool identification) |
| `tools/MCPTool/UI.tsx` | 402 | Tool execution result rendering |
| `services/mcp/channelPermissions.ts` | 240 | Channel permission relay |
| `services/mcp/channelNotification.ts` | 316 | Channel message push mechanism |
| `services/mcp/elicitationHandler.ts` | 313 | Elicitation (form/URL interaction) handling |
| `skills/mcpSkillBuilders.ts` | 44 | Skill builder registry (dependency graph decoupling) |

---

## 11.2 Theoretical Foundation

### Protocol-Driven Tool Extension Pattern

Traditional plugin systems typically rely on SDKs provided by the host application, requiring plugin developers to understand the host's internal interfaces. MCP adopts a **protocol-driven** approach: all interactions between the host (Claude Code) and plugins (MCP servers) are completed through standard JSON-RPC messages, allowing both sides to evolve independently.

This is highly consistent with the LSP (Language Server Protocol) design philosophy:

| Dimension | LSP | MCP |
|-----------|-----|-----|
| Core pattern | Editor ↔ Language Server | AI Agent ↔ Tool Server |
| Discovery mechanism | Exchange capabilities during `initialize` | `tools/list`, `resources/list`, `prompts/list` |
| Transport layer | stdio, LSP over TCP | stdio, SSE, HTTP Streamable, WebSocket |
| Bidirectional communication | Supported | Supported (notifications, elicitation) |
| Version negotiation | Supported | Supported (`protocolVersion`) |

LSP solved the M×N explosion problem of "every editor needing to integrate with every language"; MCP solves the same class of problem for "every AI tool needing to integrate with every external service."

### Client-Server Protocol Design Principles

Two key design choices in MCP have a profound impact on Claude Code's implementation:

**Capability Negotiation**: Servers declare the supported feature subset via `ServerCapabilities` at connection time (`tools`, `prompts`, `resources`, `elicitation`, `experimental`), and the client only invokes capabilities the server has declared. This means Claude Code doesn't need special branches for each server type — behavior is uniformly determined by `capabilities` checks.

**Tool Annotations**: MCP version 2025-03 introduced the `tool.annotations` field, allowing servers to declare semantic markers like `readOnlyHint`, `destructiveHint`, and `openWorldHint`. Claude Code directly maps these markers to the tool's `isReadOnly()`, `isDestructive()`, and `isOpenWorld()` methods, enabling security decisions without maintaining a static allowlist of tool names.

---

## 11.3 MCP Client Architecture

### MCPClient Class Core Interface

Claude Code does not directly implement an MCP client — it wraps the `Client` class provided by `@modelcontextprotocol/sdk`. `connectToServer` is the core entry function (`client.ts`), using `lodash/memoize` for connection-level caching with key `${name}-${jsonStringify(serverRef)}`:

```typescript
// client.ts (around line 540)
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... initialize transport based on serverRef.type
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... connect, timeout, capability negotiation
  },
  getServerCacheKey,
)
```

### Connection Management (Establish, Maintain, Disconnect)

**Establishing connections**: `connectToServer` creates the corresponding transport based on `serverRef.type`, then initiates `client.connect(transport)`, with a 30-second timeout (`getConnectionTimeoutMs()`, overridable via the `MCP_TIMEOUT` environment variable):

```typescript
// client.ts (around line 1000)
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

**Maintaining connections**: Error detection and automatic reconnection are implemented by overriding `client.onerror` and `client.onclose`. For remote transports (SSE/HTTP), a `consecutiveConnectionErrors` counter is maintained; after 3 consecutive terminal errors (`ECONNRESET`/`ETIMEDOUT`/`EPIPE`, etc.), `closeTransportAndRejectPending` is triggered. This calls `client.close()` to reject all pending `callTool()` calls, clears the memoize cache, and triggers automatic reconnection on the next request:

```typescript
// client.ts (around line 1250)
client.onclose = () => {
  // clear all related caches; next call triggers reconnection
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Session expiration handling**: HTTP-transport MCP servers may return HTTP 404 + JSON-RPC error code `-32001` (Session Not Found). Claude Code detects this specific error pattern, triggers reconnection, and transparently retries in `fetchToolsForClient.call()` (at most once):

```typescript
// client.ts (around line 150)
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**Disconnecting**: stdio transport uses a three-phase signal escalation: first `SIGINT` (100ms wait), then `SIGTERM` (400ms wait), finally `SIGKILL`, with a total disconnect time cap of 600ms to avoid blocking CLI exit.

### Dynamic Tool Discovery and Registration

`fetchToolsForClient` (with LRU cache, capacity 20) sends `tools/list` to the server, wrapping each tool as an object conforming to the internal `Tool` interface:

- **Naming rule**: `mcp__${normalizeNameForMCP(serverName)}__${toolName}` (underscore-separated format)
- **Description truncation**: Descriptions exceeding `MAX_MCP_DESCRIPTION_LENGTH = 2048` characters are truncated with `… [truncated]`, preventing overly long documentation from OpenAPI-generating servers from polluting the context
- **Permission mapping**: `tool.annotations.readOnlyHint` → `isReadOnly()`, `tool.annotations.destructiveHint` → `isDestructive()`
- **Collapse classification**: calls `classifyMcpToolForCollapse(serverName, toolName)` to determine if it is a Search/Read-type tool

Similarly, `fetchCommandsForClient` sends `prompts/list`, converting MCP Prompts into `/command` objects; `fetchResourcesForClient` sends `resources/list`, injecting `ListMcpResourcesTool` and `ReadMcpResourceTool` tools for servers supporting resources.

### Message Transport Layer

Claude Code supports 6 transport types:

| Type | Applicable Scenario | Transport Class |
|------|--------------------|----|
| `stdio` | Local subprocess (most community servers) | `StdioClientTransport` |
| `sse` | Remote SSE server (with OAuth) | `SSEClientTransport` |
| `sse-ide` | IDE extension internal SSE (no OAuth) | `SSEClientTransport` (simplified config) |
| `http` | MCP Streamable HTTP (latest spec) | `StreamableHTTPClientTransport` |
| `ws` | WebSocket transport | Custom `WebSocketTransport` |
| `ws-ide` | IDE extension internal WebSocket | `WebSocketTransport` (with `X-Claude-Code-Ide-Authorization`) |

In special scenarios, Chrome Extension MCP servers and Computer Use MCP servers run in **in-process mode**, establishing memory pipes via `createLinkedTransportPair()` to avoid the ~325 MB subprocess overhead.

HTTP transport has an important engineering detail: each POST request must carry the `Accept: application/json, text/event-stream` header (required by the MCP Streamable HTTP spec). Claude Code uniformly injects this header via `wrapFetchWithTimeout` to prevent it from being lost in some runtime environments:

```typescript
// client.ts (around line 460)
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// inside wrapFetchWithTimeout:
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 MCP Configuration Management

### Server Configuration Format

`types.ts` uses Zod to define 7 server configuration schemas, aggregated into `McpServerConfigSchema` via `z.union([...])`:

```typescript
// types.ts (lines 28-115, summary)
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // backward compatible: no type field = stdio
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

`ScopedMcpServerConfig` adds `scope` (configuration source) and `pluginSource` (identifier of the plugin's source) fields on top of the base configuration, used for Channel permission verification.

### Multi-Source Configuration Merging (enterprise > local > project > user > dynamic)

`getClaudeCodeMcpConfigs` (`config.ts`) implements multi-layer configuration merging, with priority from high to low:

1. **enterprise** (`managed-mcp.json`): Enterprise exclusive mode; when this file exists, all other sources are blocked
2. **local** (project-private, stored in user global config, bound to CWD)
3. **project** (`.mcp.json`, traverses directory tree upward, nearest takes precedence)
4. **user** (global `~/.claude/config.json`'s `mcpServers` field)
5. **dynamic** (CLI `--mcp-config` parameter runtime injection)

Project configuration requires additional **user approval gating**: upon first encountering a server in `.mcp.json`, an approval dialog pops up. `getProjectMcpServerStatus()` reads `enabledMcpjsonServers`/`disabledMcpjsonServers` settings, returning `approved`/`rejected`/`pending`. Non-interactive mode (`-p` parameter, SDK calls) with `isSettingSourceEnabled('projectSettings')` auto-approves.

After configuration merging, **deduplication** is also performed: Plugin servers are deduplicated by "signature" (stdio servers use command arrays, remote servers use URLs), preventing the same underlying server from being connected twice; claude.ai Connectors use the same mechanism to avoid duplication with manually configured entries.

### Environment Variable Expansion

Configuration files can use `${ENV_VAR}` syntax; `expandEnvVarsInString` (`config.ts`/`envExpansion.ts`) expands these when reading the configuration. Undefined variables are collected into a `missingVars` list and reported to the user.

---

## 11.5 MCP Authentication System

### OAuth 2.0 Integration

`ClaudeAuthProvider` (`auth.ts`) implements the MCP SDK's `OAuthClientProvider` interface, managing the entire OAuth lifecycle. The authentication flow follows RFC 6749 authorization code flow + PKCE (Proof Key for Code Exchange), receiving callbacks via a local HTTP server:

1. **Metadata discovery**: First probes RFC 9728 (`/.well-known/oauth-protected-resource`), falls back to RFC 8414 (`/.well-known/oauth-authorization-server`) on failure, then tries path-aware discovery (for backward compatibility)
2. **DCR (Dynamic Client Registration)**: Automatically registers an OAuth client on first authentication; `clientId`/`clientSecret` are stored in the system Keychain
3. **Token exchange**: A local random port receives the authorization code and exchanges it for access_token + refresh_token
4. **Token refresh**: `checkAndRefreshOAuthTokenIfNeeded()` detects expiration before calls and refreshes, with intelligent retry on failure

**Slack compatibility layer**: Some OAuth servers (notably Slack) return HTTP 200 for the token endpoint but carry an error body, violating RFC 6749 expectations. Claude Code rewrites such responses to HTTP 400 via `normalizeOAuthErrorBody`, making the SDK's error classification logic work correctly:

```typescript
// auth.ts (around line 250)
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // detect whether this is an OAuthErrorResponse masquerading as 200
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // normalize Slack's non-standard error code 'invalid_refresh_token' to 'invalid_grant'
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### Multiple Authentication Method Support

In addition to standard OAuth, Claude Code also supports:

- **Step-Up Auth**: Some operations require elevated permission scopes; the server returns HTTP 403 with new scope requirements, and Claude Code detects this and re-triggers the OAuth flow
- **XAA (Cross-App Access / SEP-990)**: In enterprise scenarios, a single sign-on via a unified IdP (supporting OIDC) authorizes multiple MCP servers, using RFC 8693 (Token Exchange) + RFC 7523 (JWT Bearer) composite flow, without needing separate browser windows for each server
- **Static Headers**: Inject static authentication headers via configuration files or `headersHelper` scripts (suitable for API Key authentication)

### Token Management

Token data structures are stored in system secure storage (macOS Keychain / Linux Secret Service), keyed by `${serverName}|${SHA256(config)[:16]}`, ensuring servers with the same name but different configurations use independent token slots.

`auth-cache` (`mcp-needs-auth-cache.json`) records servers that recently returned 401, with a 15-minute TTL, avoiding repeated probing of servers guaranteed to fail on each startup. Cache reads are shared via Promise (`authCachePromise`), preventing N concurrent reads of the same file during batch connections.

---

## 11.6 MCP Tool Execution

### MCPTool Execution Flow

When the LLM decides to call `mcp__slack__send_message`, the execution flow is:

1. **Routing**: The `call()` function registered by `fetchToolsForClient` is called with the JSON input generated by the LLM
2. **Reconnection check**: `ensureConnectedClient(client)` checks if the connection is still valid, reconnecting if necessary
3. **Progress notification**: Emits an `mcp_progress: started` event via the `onProgress` callback
4. **Tool call**: `callMCPToolWithUrlElicitationRetry` (wrapping `callMCPTool`) sends a `tools/call` request to the server
5. **Result processing**: Special handling for images and large binary content (persisted to disk, reference passed), large text content is truncated
6. **Progress notification**: Emits an `mcp_progress: completed` event (with duration)

Transparent retry logic for session expiration:

```typescript
// client.ts (around line 2100)
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // automatically retry once
    }
    throw error
  }
}
```

### classifyForCollapse — Context Collapse Classification for Tool Results

`classifyForCollapse.ts` maintains two static Sets: `SEARCH_TOOLS` (about 100 tool names) and `READ_TOOLS` (about 300 tool names), covering 40+ mainstream MCP servers (Slack, GitHub, Linear, Datadog, Sentry, Jira, Asana, Gmail, Grafana, PagerDuty, etc.).

Classification rules: tool names are first normalized via `normalize()` (camelCase/kebab-case uniformly converted to snake_case), then checked against the two Sets:

```typescript
// classifyForCollapse.ts (lines 587-598)
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

**Design intent**: Search/Read tool results are typically long but of limited value for subsequent LLM reasoning (retrieval intermediate state). Once classified, the UI layer can collapse these results in the conversation history, saving visual space and context window. Note that classification is **conservative** (unknown tools are not collapsed) and **based only on tool name**, not distinguishing server names, because mainstream server tool names are stable cross-instance identifiers.

### Permissions and Sandbox Control

Before executing an MCP tool, `checkPermissions()` is called, returning a `passthrough` state (always needing to display a permission prompt), with the prompt containing a shortcut suggesting the user add that tool name to the `allow` rule list.

Tool call timeout is controlled by the `MCP_TOOL_TIMEOUT` environment variable, defaulting to `100_000_000` milliseconds (approximately 27.8 hours, essentially "infinite"), allowing time-consuming MCP servers to complete normally.

---

## 11.7 MCP Channel System

The Channel system is an extended use of MCP: allowing external messaging platforms (Telegram, Discord, iMessage, Slack, etc.) to push messages to an active Claude Code session (feature flag: `KAIROS`/`KAIROS_CHANNELS`, runtime gate: `tengu_harbor`).

### Channel Permission Management

`channelPermissions.ts` implements a **permission delegation** mechanism: when Claude Code encounters an operation requiring user approval, it can simultaneously send a prompt to the user's phone through the Channel server, and when the user replies `yes <5-letter-ID>`, the server parses it and notifies Claude Code of the approval via a `notifications/claude/channel/permission` event.

The 5-letter ID uses a 25-character alphabet (removing `l` to prevent confusion with `1`/`I`), generated via FNV-1a hash, with a profanity filter (`ID_AVOID_SUBSTRINGS` list, about 24 words) to ensure inappropriate content doesn't appear in work messages:

```typescript
// channelPermissions.ts (lines 86-110)
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

Channel servers must simultaneously declare both `capabilities.experimental['claude/channel']` and `capabilities.experimental['claude/channel/permission']` capabilities to serve as permission relays, preventing accidental opening of security boundaries.

### Channel Notification Mechanism

`channelNotification.ts` defines the complete gating logic for receiving inbound messages (`gateChannelServer`), checking in sequence:

1. Server capability declaration (`claude/channel`)
2. Runtime switch (`tengu_harbor`)
3. OAuth authentication (only claude.ai accounts supported, not API Keys)
4. Team/enterprise policy (`channelsEnabled: true`)
5. Session `--channels` parameter (channels the user has explicitly declared as trusted)
6. Marketplace source verification (preventing `slack@evil` from impersonating `slack@anthropic`)

Inbound messages are wrapped in `<channel source="serverName" meta_key="value">content</channel>` format and injected into the session queue. `SleepTool` polling (approximately 1-second intervals) wakes up the model, which can then decide how to respond.

### Elicitation Handling

`elicitationHandler.ts` handles interactive requests proactively initiated by the server (MCP Elicitation spec). Two modes are supported:

- **Form mode**: Server requests the user to fill out a form (`requestedSchema` field defines JSON Schema)
- **URL mode**: Server requests the user to visit a URL to complete an action (e.g., OAuth authorization)

Processing flow: first runs the Hook system (programmatic response); if no Hook responds, the request is queued to `AppState.elicitation.queue`, waiting for the UI to render the form or open a browser; after user action, the `respond()` callback triggers the response:

```typescript
// elicitationHandler.ts (lines 69-90)
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. try Hook first (programmatic response)
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. show UI, wait for user response
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

URL mode also supports `ElicitationCompleteNotificationSchema`: the server actively notifies Claude Code after completing an operation, marking the corresponding queue entry `completed: true`, which the UI uses to update the display state.

---

## 11.8 MCP Skill Building

`skills/mcpSkillBuilders.ts` is a minimal dependency graph decoupling module (44 lines) solving a circular dependency problem:

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts  (circular)
```

The solution introduces a Write-Once Registry:

```typescript
// mcpSkillBuilders.ts (full content)
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

The `loadSkillsDir.ts` module calls `registerMCPSkillBuilders()` during initialization, which completes early during application startup (via `commands.ts`'s static import), ensuring the registry is ready when MCP servers connect.

Skill discovery mechanism (Feature Flag: `MCP_SKILLS`): `fetchMcpSkillsForClient` reads the server's `skill://` resources, parsing Markdown-format skill files (with frontmatter metadata) into Skill Commands, registering them as `/serverName:skillName` format slash commands, enabling MCP servers to automatically provide reusable AI workflows.

---

## 11.9 Design Decision Analysis

### Why MCP Rather Than a Custom Plugin Protocol

**Problem**: Claude Code needs to support many third-party tool integrations; a custom plugin API would create ecosystem lock-in.

**Decision**: Adopt the MCP open standard, sharing the MCP community ecosystem.

**Rationale**:
- By early 2025, the MCP ecosystem already had hundreds of open-source servers ready for direct reuse
- Plugin developers can implement MCP servers in any language (Python, Go, Java, etc.), unconstrained by the TypeScript ecosystem
- Anthropic is both the developer of Claude Code and the lead of the MCP specification, enabling solving needs at the spec level rather than patching the client

**Cost**: All protocol complexity (authentication, transport layer differences, version compatibility) is borne by the client; `auth.ts`'s 2,465 lines are largely the result of handling OAuth specification deficiencies and various vendors' non-compliant implementations.

### Handling MCP Authentication Complexity

**Problem**: MCP spec's description of authentication is "advisory," and actual server implementations vary wildly (Slack returns errors with 200, some servers don't implement Token Revocation, etc.).

**Decision**: Build a complete compatibility layer in `auth.ts` handling known non-compliant behaviors.

Key strategies:
- `normalizeOAuthErrorBody`: handles responses where 200 masks an error
- `NONSTANDARD_INVALID_GRANT_ALIASES`: standardizes non-standard error codes from Slack and others
- RFC 7009 revocation double-attempt (try standard way first; if 401 received, retry with Bearer)
- Two auth discovery paths (RFC 9728 → RFC 8414 → path-aware fallback)

### Design Considerations for Tool Collapse Classification

**Problem**: MCP tool results can be extremely long (search results, log output), but large amounts of inline content in conversation history reduces readability and wastes context window.

**Decision**: Use explicit tool name allowlists rather than heuristic classification; default to collapsing results in the UI layer for known Search/Read tools.

**Tradeoffs**:
- Advantage: Strong determinism, no false positives, consistent behavior for known tools
- Disadvantage: Requires maintaining a static list (`classifyForCollapse.ts` currently covers 40+ servers); new servers require manual updates
- Conservative strategy (unknown tools not collapsed) ensures new servers won't lose information due to incorrect collapsing

---

## 11.10 Transferable Patterns

The following patterns come from the engineering practice of Claude Code's MCP integration, applicable to other systems that need to integrate external tools or services:

**1. Capability Negotiation Over Type Checking**
When implementing protocol clients, always determine behavior via `capabilities` checks rather than if-else branches based on server type or name. This allows new capability support to be added incrementally without affecting existing logic.

**2. Memoize + Cache Invalidation Pattern**
Expensive operations like connections and tool discovery use memoize caching, but the cache must be immediately invalidated when the connection drops (clearing all related cache entries in `client.onclose`). Use LRU cache (capacity 20) to prevent memory leaks.

**3. Write-Once Registry for Solving Circular Dependencies**
When module A needs to depend on functions from module B, but module B indirectly depends on module A, introduce a zero-external-dependency registry module. During application initialization, module B injects its implementation into the registry; module A reads from the registry. `mcpSkillBuilders.ts` is the minimal copyable template.

**4. Protocol Compatibility Layer Centralized Management**
Non-compliant OAuth/HTTP spec implementations should be handled in one location (e.g., `auth.ts`) rather than scattered at call sites. `normalizeOAuthErrorBody` is the canonical example of this pattern: a pure function that uniformly handles non-compliant servers at the transport layer, so call sites don't need to care whether the server is spec-compliant.

**5. Layered Rate Limiting for Concurrent Connections**
Different types of operations consume resources differently. stdio servers require forking subprocesses (CPU + memory intensive), while network servers only need TCP connections (I/O intensive). Using different concurrency limits for the two types (local: 3, remote: 20) maximizes throughput while protecting the system.

**6. Dual-Check for needs-auth State**
For remote services requiring authentication, combine **time-based caching** (15-minute TTL) with **state-based checking** (has discovery state but no token) to skip connections that are guaranteed to fail, avoiding invalid probe delays on each startup.

---

## 11.11 Source Index

| Key Implementation | File:Line |
|-------------------|-----------|
| `MCPServerConnection` union type definition | `services/mcp/types.ts:170-200` |
| `ConfigScope` enum (7 sources) | `services/mcp/types.ts:10-22` |
| `connectToServer` main function (memoized) | `services/mcp/client.ts:540` |
| Transport initialization branches (6 types) | `services/mcp/client.ts:570-930` |
| Connection timeout handling | `services/mcp/client.ts:1000-1040` |
| Disconnect error detection and reconnection trigger | `services/mcp/client.ts:1200-1320` |
| stdio three-phase close (SIGINT/SIGTERM/SIGKILL) | `services/mcp/client.ts:1370-1490` |
| `fetchToolsForClient` (tool registration) | `services/mcp/client.ts:1830-2050` |
| `getMcpToolsCommandsAndResources` (batch connection entry) | `services/mcp/client.ts:2580` |
| `isMcpSessionExpiredError` | `services/mcp/client.ts:150-165` |
| `wrapFetchWithTimeout` (HTTP Accept injection) | `services/mcp/client.ts:450-510` |
| Config source merging `getClaudeCodeMcpConfigs` | `services/mcp/config.ts:1050` |
| Enterprise exclusive mode determination | `services/mcp/config.ts:1080-1090` |
| Project server approval gating | `services/mcp/utils.ts:210-250` |
| Plugin server deduplication (signature hash) | `services/mcp/config.ts:215-270` |
| `ClaudeAuthProvider` (OAuth core) | `services/mcp/auth.ts:500+` |
| `normalizeOAuthErrorBody` (Slack compatibility) | `services/mcp/auth.ts:250-290` |
| `performMCPXaaAuth` (cross-app authentication) | `services/mcp/auth.ts:700+` |
| `getServerKey` (token storage key generation) | `services/mcp/auth.ts:390-405` |
| `hasMcpDiscoveryButNoToken` (fast fail) | `services/mcp/auth.ts:420-435` |
| `classifyMcpToolForCollapse` (collapse classification) | `tools/MCPTool/classifyForCollapse.ts:587-598` |
| `SEARCH_TOOLS` / `READ_TOOLS` allowlists | `tools/MCPTool/classifyForCollapse.ts:20-585` |
| `shortRequestId` (Channel permission ID) | `services/mcp/channelPermissions.ts:140-160` |
| `gateChannelServer` (6-layer gating) | `services/mcp/channelNotification.ts:190-310` |
| `registerElicitationHandler` | `services/mcp/elicitationHandler.ts:65-150` |
