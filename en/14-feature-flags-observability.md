# Chapter 14: Feature Flags and Observability

## 14.1 Overview and Purpose

Claude Code's observability system is a multi-layer, multi-objective system covering everything from compile-time feature trimming to runtime behavior tracking. The entire system is built on three major pillars:

1. **Feature Flag System**: Dual-track design — compile-time `feature()` calls (implementing Dead Code Elimination via `bun:bundle`) and runtime GrowthBook dynamic configuration. The former controls the feature boundaries delivered to different user groups; the latter supports toggling feature switches without re-deployment.

2. **Observability Pipeline**: Based on the OpenTelemetry standard, supporting gRPC/HTTP/Protobuf three export protocols, uniformly collecting Metrics, Logs, and Traces signals, with an internal debugging layer provided via Perfetto trace format.

3. **Analytics Collection**: Dual-route routing — Datadog (external monitoring) + First-Party 1P event logging (internal BigQuery/proto), using event name prefix `tengu_*` to identify all business events, with GrowthBook dynamic sampling configuration to control data volume.

The core design principle of this system is **layered isolation**: user privacy first (no code content or file paths recorded by default), internal/external build differences (ant-only vs external), and graceful degradation (every layer has a kill-switch).

---

## 14.2 Theoretical Foundation

### Feature Flag-Driven Development

Feature Flags enable teams to develop features at different stages in parallel within the same codebase and enable them on demand. Claude Code uses a two-layer flag mechanism:

- **Compile-time flags**: Using `feature()` calls provided by `bun:bundle`, performing Dead Code Elimination during bundling. Entire code blocks that don't exist in the external version are completely deleted, reducing bundle size and preventing internal logic from being reverse-engineered.
- **Runtime flags**: Dynamically fetched from the server via GrowthBook SDK, supporting scenarios like A/B testing, canary releases, and emergency kill-switches.

### Three Pillars of Observability

The OpenTelemetry community defines observability as three major signals:

- **Metrics**: Time-series numerical data, such as API latency and token consumption. Claude Code uses `@opentelemetry/sdk-metrics` to export via PeriodicExportingMetricReader every 60 seconds.
- **Logs**: Structured event records. All `logEvent()` calls ultimately export in bulk via OTel `LoggerProvider` + `BatchLogRecordProcessor`.
- **Traces**: Distributed call chains. Claude Code establishes a hierarchical Span tree of Interaction → LLM Request → Tool Call via `sessionTracing.ts`, supporting parent-child relationship tracking in multi-Agent scenarios.

### A/B Testing in CLI Tools

Unlike web products, CLI tool A/B testing faces unique challenges: no browser fingerprinting, multiple platforms and release channels, offline operation scenarios. Claude Code's strategies:

- User dimension targeting: `GrowthBookUserAttributes` carries attributes like `platform`, `subscriptionType`, and `rateLimitTier`, supporting layered experiments.
- Local disk caching: After each successful fetch from the server, the feature values are written to `cachedGrowthBookFeatures` in `~/.claude/config.json`, ensuring last-known values are available when offline.
- Exposure deduplication: Within the same session, each feature's experiment exposure event is only recorded once (`loggedExposures` Set).

---

## 14.3 Feature Flag System

### GrowthBook Integration

GrowthBook is an open-source Feature Flag and A/B testing platform. Claude Code integrates via the official `@growthbook/growthbook` SDK, located in `src/services/analytics/growthbook.ts` (1,155 lines).

**Initialization flow**:

```typescript
// growthbook.ts:529-600 (simplified)
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

Key design: `memoize` ensures GrowthBook client is initialized only once throughout the process lifecycle; when auth changes (login/logout), the client is destroyed and rebuilt via `refreshGrowthBookAfterAuthChange()`, rather than trying to update `apiHostRequestHeaders` (which the SDK doesn't support after initialization).

**User attribute model** (`growthbook.ts:31-46`):

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

**Refresh strategy**:
- External users: refresh every 6 hours (`6 * 60 * 60 * 1000`)
- Internal employees (ant): refresh every 20 minutes

**Cache architecture** (three priority levels):
1. In-memory `remoteEvalFeatureValues` Map (latest in-process values)
2. Disk cache `~/.claude/config.json`'s `cachedGrowthBookFeatures` (cross-process persistence)
3. Legacy `cachedStatsigGates` (migration compatibility layer, being gradually phased out)

**API compatibility workaround** (`growthbook.ts:320-390`): The server-returned remoteEval response uses the `value` field, while the SDK expects `defaultValue`. The code has explicit format conversion logic with a TODO comment waiting for a server-side fix.

**Environment variable override** (ant internal users only):
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### Complete Compile-Time Feature Flag Listing (80+)

Implemented via `bun:bundle`'s `feature()` calls for dead code elimination. The following are all compile-time flags extracted from the source code:

| Flag Name | Location | Controlled Feature |
|-----------|----------|-------------------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Perfetto tracing (ant-only) |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | Enhanced telemetry beta |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | Auto mode / scheduled task system |
| `KAIROS_BRIEF` | `commands.ts` | KAIROS brief mode |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | KAIROS channel support |
| `KAIROS_DREAM` | `commands.ts` | KAIROS dream mode |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | GitHub webhook subscription |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | KAIROS push notifications |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Agent triggers (scheduled tasks) |
| `AGENT_TRIGGERS_REMOTE` | — | Remote Agent triggers |
| `AGENT_MEMORY_SNAPSHOT` | — | Agent memory snapshot |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | Conversation classifier |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | Verification agent |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | Built-in explore/plan agents |
| `COORDINATOR_MODE` | `builtInAgents.ts` | Coordinator mode |
| `FORK_SUBAGENT` | `commands.ts` | Fork subagent |
| `BUDDY` | `commands.ts` | Buddy feature |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Unix Domain Socket inbox |
| `BRIDGE_MODE` | `commands.ts` | Bridge mode (CCR) |
| `DAEMON` | `commands.ts` | Daemon mode |
| `VOICE_MODE` | `commands.ts` | Voice mode |
| `ULTRAPLAN` | `commands.ts` | UltraPlan command |
| `ULTRATHINK` | — | UltraThink feature |
| `TORCH` | `commands.ts` | TORCH command (dynamic loading) |
| `MCP_SKILLS` | `commands.ts` | MCP skill support |
| `CHICAGO_MCP` | `metadata.ts` | Chicago MCP built-in server (computer-use) |
| `WORKFLOW_SCRIPTS` | `commands.ts` | Workflow scripts |
| `CCR_REMOTE_SETUP` | `commands.ts` | CCR remote setup command |
| `CCR_AUTO_CONNECT` | — | CCR auto-connect |
| `CCR_MIRROR` | — | CCR mirror mode |
| `PROACTIVE` | `commands.ts` | Proactive mode |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | Experimental skill search |
| `HISTORY_SNIP` | `commands.ts` | History snippet feature |
| `HISTORY_PICKER` | — | History picker |
| `WEB_BROWSER_TOOL` | — | Web browser tool |
| `QUICK_SEARCH` | — | Quick search |
| `MONITOR_TOOL` | — | Monitor tool |
| `OVERFLOW_TEST_TOOL` | — | Overflow test tool |
| `BREAK_CACHE_COMMAND` | — | Force cache breakpoint command |
| `TREE_SITTER_BASH` | — | Tree-sitter Bash parsing |
| `TREE_SITTER_BASH_SHADOW` | — | Tree-sitter shadow comparison |
| `BASH_CLASSIFIER` | — | Bash security classifier |
| `TERMINAL_PANEL` | — | Terminal panel |
| `NATIVE_CLIPBOARD_IMAGE` | — | Native clipboard image support |
| `NATIVE_CLIENT_ATTESTATION` | — | Native client attestation |
| `AUTO_THEME` | — | Auto theme |
| `POWERSHELL_AUTO_MODE` | — | PowerShell auto mode |
| `TOKEN_BUDGET` | — | Token budget display |
| `STREAMLINED_OUTPUT` | — | Streamlined output mode |
| `CONNECTOR_TEXT` | — | Connector text |
| `CONTEXT_COLLAPSE` | — | Context collapse |
| `COMPACTION_REMINDERS` | — | Compaction reminders |
| `CACHED_MICROCOMPACT` | — | Cached micro-compaction |
| `REACTIVE_COMPACT` | — | Reactive compaction |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Prompt cache breakpoint detection |
| `EXTRACT_MEMORIES` | — | Automatic memory extraction |
| `LODESTONE` | — | Lodestone feature |
| `TEAMMEM` | — | Team memory |
| `TEMPLATES` | — | Templates feature |
| `FILE_PERSISTENCE` | — | File persistence |
| `BG_SESSIONS` | — | Background sessions |
| `DOWNLOAD_USER_SETTINGS` | — | User settings download |
| `UPLOAD_USER_SETTINGS` | — | User settings upload |
| `NEW_INIT` | — | New initialization flow |
| `HARD_FAIL` | — | Hard fail mode |
| `SLOW_OPERATION_LOGGING` | — | Slow operation logging |
| `SHOT_STATS` | — | Request statistics |
| `MEMORY_SHAPE_TELEMETRY` | — | Memory shape telemetry |
| `COWORKER_TYPE_TELEMETRY` | — | Coworker type telemetry |
| `ANTI_DISTILLATION_CC` | — | Anti-distillation protection |
| `RUN_SKILL_GENERATOR` | — | Skill generator |
| `SKILL_IMPROVEMENT` | — | Skill improvement |
| `REVIEW_ARTIFACT` | — | Code review artifact |
| `MESSAGE_ACTIONS` | — | Message actions |
| `AWAY_SUMMARY` | — | Away summary |
| `COMMIT_ATTRIBUTION` | — | Commit attribution |
| `UNATTENDED_RETRY` | — | Unattended retry |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | libc type detection (injected at build time) |

### Runtime Feature Flags vs. Compile-Time Feature Flags

| Dimension | Compile-time (`feature()`) | Runtime (GrowthBook) |
|-----------|---------------------------|---------------------|
| Execution timing | Bundling phase | Asynchronous loading after process startup |
| Code retention | Eliminated branches don't exist in the artifact | Code exists but logic controlled by flag value |
| Update method | Publish new version | Server push, effective within 20 minutes at fastest |
| Typical use | ant-only features, experimental tools, platform-specific code | A/B testing, canary, kill-switch, dynamic configuration |
| Override method | Build variables | `CLAUDE_INTERNAL_FC_OVERRIDES` environment variable (ant only) |

### Dead Code Elimination Mechanism

`bun:bundle`'s `feature()` is a special built-in function of the Bun bundler, directly replacing `feature('X')` with `true` or `false` at build time based on build-time definitions, then having constant folding and dead code elimination trim away permanently-false branches.

Example (`perfettoTracing.ts:216-220`):
```typescript
// In external builds, this entire if block is completely deleted
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... all Perfetto initialization code
}
```

This mechanism not only reduces bundle size but also prevents internal tool code from being exposed in external artifacts.

### Known Important Runtime Feature Flags

The following are some known `tengu_*` GrowthBook flags and their functions:

| Flag Name | Type | Feature Description |
|-----------|------|-------------------|
| `tengu_auto_mode_config` | Object | Auto mode configuration (enabled/opt-in) |
| `tengu_1p_event_batch_config` | Object | 1P event batch export configuration |
| `tengu_event_sampling_config` | Object | Event sampling rate configuration dictionary |
| `tengu_log_datadog_events` | Boolean | Datadog event reporting toggle |
| `tengu_frond_boric` | Object | Analytics sink kill-switch (disable by sink name) |
| `tengu_quartz_lantern` | Boolean | FileWriteTool atomic write behavior control |
| `tengu_hive_evidence` | Boolean | Task update/Todo write behavior control |
| `tengu_plum_vx3` | Boolean | Toggle for WebSearchTool using Haiku model |
| `tengu_kairos_cron` | Object | KAIROS scheduled task configuration |
| `tengu_kairos_cron_durable` | Boolean | Durable scheduled task support |
| `tengu_agent_list_attach` | Boolean | AgentTool list attach behavior |
| `tengu_amber_stoat` | Boolean | Built-in agent availability control |
| `tengu_slim_subagent_claudemd` | Boolean | Slim subagent CLAUDE.md loading |
| `tengu_glacier_2xr` | Boolean | ToolSearch mode decision control |
| `tengu_max_version_config` | Object | Max version limit (force upgrade kill-switch) |
| `tengu_prompt_cache_1h_config` | Object | Prompt cache 1-hour configuration |
| `tengu_sm_compact_config` | Object | Session Memory compaction configuration |
| `tengu_ant_model_override` | String | ant-specific model override |
| `enhanced_telemetry_beta` | Boolean | Enhanced telemetry beta toggle |

---

## 14.4 Observability System

### OpenTelemetry Integration

Claude Code fully implements OpenTelemetry three-signal support, with the core entry point at `src/utils/telemetry/instrumentation.ts` (825 lines).

**Initialization bootstrap** (`instrumentation.ts:bootstrapTelemetry()`):
In ant builds, reads configuration from `ANT_OTEL_*` prefix variables and maps them to standard `OTEL_*` variables. For external users, follows the standard OTel environment variable configuration specification; default temporality is set to `delta` (incremental rather than cumulative).

**Three-signal exporter configuration** (lazy-loading design):

```typescript
// instrumentation.ts:169-190 (simplified)
// OTLP/Prometheus exporters use dynamic import lazy-loading
// to avoid loading @grpc/grpc-js (~700KB) when not needed
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

All three transport protocols are supported: `grpc`, `http/json`, `http/protobuf`, selected via the `OTEL_EXPORTER_OTLP_PROTOCOL` environment variable.

**Resource attributes**: Service name is `claude-code`, carrying platform architecture, WSL version, subscription type, service version, and other attributes, automatically filled via `envDetector`, `hostDetector`, `osDetector`.

### gRPC Data Transmission

gRPC is the recommended transport protocol for enterprise scenarios, providing bidirectional streaming and strongly-typed protobuf encoding. In Claude Code:

- gRPC exporter (`@opentelemetry/exporter-metrics-otlp-grpc`) as a lazy-loaded dependency, avoiding impact on startup time
- mTLS configuration supported via `getMTLSConfig()`, usable with self-signed certificates in enterprise intranet scenarios
- Proxy support transparently handled via `getProxyUrl()` + `HttpsProxyAgent`

Subprocesses do not inherit OTEL-related environment variables (`subprocessEnv.ts`):
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Perfetto Tracing

Perfetto is a high-performance system-level tracing framework developed by Google. Claude Code implements a compatibility layer for its Chrome Trace Event format (`src/utils/telemetry/perfettoTracing.ts`, 1,120 lines, ant-only).

**How to enable**:
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # write to ~/.claude/traces/trace-<session-id>.json
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # write to specified path
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # write periodically every 60 seconds
```

**Span types traced**:

| Span Name | Category | Carried Information |
|-----------|----------|-------------------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | attempt number |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**Memory management** (event cap of 100,000):
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// When cap is reached, delete the oldest half, amortized O(1) cost
// Insert trace_truncated marker to make the gap visible in ui.perfetto.dev
```

**Multi-Agent hierarchical tracing**: Each Agent (including sub-Agents) maps to an independent process ID, with hierarchy recorded via `parent_agent` metadata events, appearing as independent swim lanes in the Perfetto UI.

**Write strategy** (three-layer guarantee):
1. `cleanup registry` async callback (normal exit)
2. `process.on('beforeExit')` handler (backup)
3. `process.on('exit')` synchronous write (last line of defense; async not available here)

### OpenTelemetry Session Tracing

`src/utils/telemetry/sessionTracing.ts` (927 lines) is the enhanced telemetry entry point for external users, based on standard OTel Spans rather than Perfetto format.

**Enable conditions** (`sessionTracing.ts:170-185`):
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

**AsyncLocalStorage context propagation**: Each Interaction and Tool Call uses independent ALS storage for SpanContext, ensuring Spans don't get mixed in concurrent multi-Agent scenarios. WeakRef weak references prevent long-lived Span memory leaks; orphaned Spans older than 30 minutes are cleaned up at 60-second intervals.

**logEvent event system**

All business events are uniformly dispatched via `logEvent()` in `src/services/analytics/index.ts`:

```typescript
// index.ts (simplified)
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // only allows boolean | number | undefined
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

Key design: the metadata type intentionally excludes `string`, forcing developers to use the `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type conversion, preventing accidental recording of code content or file paths at the type level.

---

## 14.5 Analytics Collection

### Dual-Route Architecture

All events are routed through `sink.ts` to two backends:

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog (if tengu_log_datadog_events gate is enabled)
    │     only transmits events within DATADOG_ALLOWED_EVENTS allowlist
    │     removes _PROTO_* keys (PII marked fields)
    └─→ 1P First-Party Logger (OpenTelemetry BatchLogRecordProcessor)
          sends to /api/event_logging/batch
          retains _PROTO_* keys (routes to BigQuery protected columns)
```

**Datadog integration** (`datadog.ts`):
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Batch sending: 100 entries/batch, 15-second flush interval
- Network timeout: 5 seconds
- Allowlist mechanism: approximately 50 core events (`DATADOG_ALLOWED_EVENTS` Set)
- Disabled conditions: Bedrock/Vertex/Foundry third-party clouds, test environments, users who opted out of telemetry

**1P Event Logging (FirstPartyEventLoggingExporter)**:
- Uses OpenTelemetry standard `LogRecordExporter` interface
- Batch export: default 200 entries/batch, 5-second scheduling delay
- Failure retry: exponential backoff (base 500ms, max 30s, up to 8 retries)
- Persistent failure queue: failed events written to `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl`, retried on next startup
- Proto serialization: uses generated `ClaudeCodeInternalEvent` protobuf type

### User Behavior Tracking

Approximately 400+ `tengu_*` event names cover the complete user interaction lifecycle. Core event categories:

**Session lifecycle**: `tengu_started`, `tengu_init`, `tengu_exit`, `tengu_cancel`

**API calls**: `tengu_api_query`, `tengu_api_success`, `tengu_api_error`, `tengu_api_retry`

**Tool usage**: `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_in_prompt_permanent`

**Permission requests**: `tengu_internal_bash_tool_use_permission_request`, `tengu_tool_use_show_permission_request`, `tengu_tool_use_granted_by_classifier`

**OAuth authentication**: `tengu_oauth_flow_start`, `tengu_oauth_success`, `tengu_oauth_token_refresh_*` (complete lock state machine tracking)

**MCP servers**: `tengu_mcp_server_connection_succeeded`, `tengu_mcp_server_connection_failed`, `tengu_mcp_oauth_flow_*`

**Update mechanism**: `tengu_binary_download_attempt`, `tengu_native_update_complete`, `tengu_binary_download_failure`

### Performance Metrics Collection

The API Call Span in `sessionTracing.ts` calculates the following derived metrics:

```typescript
// perfettoTracing.ts (endLLMRequestPerfettoSpan simplified)
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second (input token processing rate)

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second (sampling rate)

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // Cache hit rate (percentage)
```

### Event Sampling Control

Sampling rates for each event are controlled via GrowthBook dynamic configuration `tengu_event_sampling_config`:

```typescript
// firstPartyEventLogger.ts (shouldSampleEvent simplified)
// null = 100% sampling (no configuration)
// 0 = completely dropped
// rate (0-1) = random sampling, writing sample_rate to metadata
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // example: 10% sampling
}
```

### Error Reporting

Multi-layer error event system:
- `tengu_uncaught_exception`, `tengu_unhandled_rejection`: process-level uncaught errors
- `tengu_api_error`, `tengu_query_error`: API call errors
- `tengu_streaming_error`: streaming response errors
- `tengu_atomic_write_error`: file write errors
- `tengu_compact_failed`: session compaction failures

---

## 14.6 Diagnostics and Debugging

### /doctor Command

`src/commands/doctor/index.ts` registers the `/doctor` command:

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

This command executes as `local-jsx` type (rendering React components directly within the REPL), checking: installation integrity, MCP server connection status, keybinding configuration validity, and environment dependencies (ripgrep, etc.).

### Diagnostic Tracing System

In IDE integration scenarios, Claude Code receives code diagnostic information via the Language Server Protocol. When a file is saved (`didSave` event), the TypeScript Server sends new diagnostic messages, which the system injects as `<new-diagnostics>` XML tags to pass to the model:

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### Heap Memory Diagnostics

`src/utils/heapDumpService.ts` provides process-level memory diagnostics, synchronously collecting memory usage snapshots when triggering heap dumps, outputting to `~/Desktop/<session-id>-diagnostics.json`, containing `heapUsed`, `external`, `rss`, and analysis suggestions. Corresponding analytics event: `tengu_heap_dump`.

### Error Recovery Logging

`src/utils/telemetry/bigqueryExporter.ts` implements a BigQuery metrics exporter, integrated with the OTel Metrics pipeline, for long-term performance monitoring and capacity planning within ant internally. The `1p_failed_events` persistent queue ensures critical events are not lost even during network failures.

---

## 14.7 Design Decision Analysis

### Pros and Cons of Compile-Time Flags

**Advantages**:
1. **Zero runtime overhead**: Eliminated code branches don't exist in the artifact; no conditional judgment overhead whatsoever
2. **Security isolation**: ant-only feature code is completely invisible to external users and cannot be reverse-engineered
3. **Bundle size optimization**: Large modules (e.g., `@grpc/grpc-js` ~700KB) only exist in builds that need them
4. **Type safety**: TypeScript type checking operates before bundling, without affecting runtime

**Disadvantages**:
1. **Release dependency**: Changing flag status requires publishing a new version; cannot hot-update
2. **Test matrix explosion**: N compile-time flags theoretically require 2^N build combination tests
3. **Debugging complexity**: When external users report issues, some code paths simply don't exist in their builds

### Balancing Privacy and Observability

Claude Code uses multiple lines of defense for privacy protection:

1. **Type system defense**: `LogEventMetadata` only allows `boolean | number | undefined`, prohibiting direct string reporting. To record a string, `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` must be explicitly used — this is a `never` type that cannot actually hold a value. It merely forces developers to write a type annotation indicating they have manually verified the string contains no code or paths.

2. **MCP tool name anonymization**: The MCP tool name format `mcp__<server>__<tool>` could expose users' private service configurations; by default anonymized to `mcp_tool`. Only server names in the cowork entry, official MCP registry, or explicitly declared as built-in retain the original name.

3. **PII marked fields**: `_PROTO_*` prefix metadata keys indicate PII-sensitive fields, routed only to 1P protected BigQuery columns; `sink.ts` strips these fields before forwarding to Datadog.

4. **Third-party cloud disabling**: Enterprise customers using Bedrock/Vertex/Foundry have all Anthropic-side analytics (including Datadog and 1P) disabled by default.

### Why Lazy-Load Telemetry

OTLP-related packages (gRPC ~700KB, proto ~300KB) use dynamic `import()` lazy-loading because:

1. **Startup time sensitivity**: A CLI tool's primary performance metric is Time-to-First-Output; any unnecessary initialization should be deferred
2. **Protocol mutually exclusive**: A process only uses one transport protocol; statically importing all variants (6 packages) is pure waste
3. **Bun optimization compatible**: Lazy-loading conforms to Bun's module resolution optimization pattern, statically analyzed and bundled on demand

---

## 14.8 Transferable Patterns

The following design patterns have high reference value for other projects:

### 1. Type System to Prevent PII Leakage

Through a `never` type marker type, force developers to explicitly confirm the absence of sensitive information at compile time. Zero cost (no runtime overhead), 100% protection (bypassing requires explicit type assertion). Applicable to any system with data reporting requirements.

### 2. Dual-Level Feature Flag Architecture

Compile-time (code layering) + runtime (behavior control) dual-track, corresponding to different release speed requirements:
- Structural features (presence or absence of entire modules) → compile-time
- Behavior tuning (parameters, ratios, algorithm selection) → runtime

### 3. Sink Kill-Switch Pattern

The `tengu_frond_boric` GrowthBook configuration allows independently disabling any analytics backend by name (`datadog`, `firstParty`) without publishing a new version. This is a general emergency circuit breaker pattern, suitable for all event systems with multiple downstream sinks.

### 4. Persistent Retry for Failed Events

When 1P event export fails, write to a local JSONL file and retry on next startup. This ensures critical telemetry data is not lost during network failures, particularly suitable for tools running in offline or unstable network environments.

### 5. Experiment Exposure Deduplication

GrowthBook experiment exposure events (for A/B test result analysis) are deduplicated via a session-level Set, ensuring the same feature's exposure is recorded only once on the analysis side, preventing inflated exposure counts from multiple calls to the same flag.

---

## 14.9 Source Index

| File Path (relative to `src/`) | Lines | Core Responsibility |
|-------------------------------|-------|-------------------|
| `services/analytics/growthbook.ts` | 1,155 | GrowthBook SDK integration, Feature Flag reading, A/B exposure recording |
| `services/analytics/index.ts` | 173 | logEvent public API, event queue, Sink interface definition |
| `services/analytics/sink.ts` | 114 | Dual-route implementation (Datadog + 1P), initialization |
| `services/analytics/datadog.ts` | 307 | Datadog batch log sending, allowlist filtering |
| `services/analytics/firstPartyEventLogger.ts` | 449 | OpenTelemetry LoggerProvider initialization, sampling control |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | 1P event HTTP export, persistent retry, proto serialization |
| `services/analytics/metadata.ts` | 973 | Event metadata enrichment, MCP tool name anonymization, PII handling |
| `services/analytics/config.ts` | 38 | isAnalyticsDisabled() shared logic |
| `services/analytics/sinkKillswitch.ts` | 25 | Sink-level Kill-Switch (tengu_frond_boric) |
| `utils/telemetry/instrumentation.ts` | 825 | OTel SDK initialization, three-signal (Metrics/Logs/Traces) configuration |
| `utils/telemetry/sessionTracing.ts` | 927 | OTel Span management, AsyncLocalStorage context propagation |
| `utils/telemetry/perfettoTracing.ts` | 1,120 | Perfetto Chrome Trace format tracing (ant-only) |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Beta tracing extended attributes |
| `utils/telemetry/bigqueryExporter.ts` | 252 | BigQuery metrics exporter |
| `utils/telemetry/pluginTelemetry.ts` | 289 | Plugin telemetry encapsulation |
| `utils/telemetry/events.ts` | 75 | OTel event type definitions |
| `commands/doctor/index.ts` | 12 | /doctor command registration |
