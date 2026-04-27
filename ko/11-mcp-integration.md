# 제 11 장：MCP 통합

## 11.1 개요 및 위치

### MCP란 무엇인가

MCP(Model Context Protocol)는 Anthropic이 주도적으로 설계한 개방형 프로토콜로, AI 애플리케이션과 외부 도구 서비스 간의 표준화된 통신 형식을 정의한다. 본질적으로 JSON-RPC 2.0 프로토콜로, 여러 전송 층(stdio, SSE, HTTP Streamable, WebSocket) 위에서 실행되며, 도구 발견(`tools/list`), 도구 호출(`tools/call`), 리소스 관리(`resources/list`/`resources/read`), Prompt 템플릿(`prompts/list`/`prompts/get`) 등의 표준 메시지 형식을 규정한다.

### Claude Code에서 MCP의 역할

Claude Code의 내장 도구 세트(Bash, Read, Edit 등)는 파일 시스템과 로컬 개발 시나리오를 커버한다. MCP의 설계 포지셔닝은 **개방적인 도구 확장 인터페이스**다: 모든 서드파티 서비스(Slack, GitHub, Jira, 데이터베이스, 브라우저 자동화 등)가 MCP 서버를 구현할 수 있으며, Claude Code는 표준 프로토콜을 통해 연결하여 이러한 외부 기능을 호출할 수 있다. 핵심 코드를 수정할 필요 없이.

아키텍처적으로, Claude Code는 순수한 **MCP 클라이언트**로, 어떤 MCP 서버 기능도 구현하지 않는다(단, 작업 디렉터리를 서버에 알리기 위해 `roots/list` 요청에 응답하는 것 제외). 연결된 각 MCP 서버의 도구는 `mcp__<serverName>__<toolName>` 형식의 Tool 객체로 동적으로 등록되며, 내장 도구와 동일한 실행 프레임워크를 공유한다.

### 코드 규모

MCP 통합에는 약 12,310줄의 TypeScript 코드가 포함되며, 다음 파일들에 분산되어 있다:

| 파일 | 줄수 | 역할 |
|------|------|------|
| `services/mcp/client.ts` | 3,348 | 연결 관리, 도구 발견, 실행 핵심 |
| `services/mcp/config.ts` | 1,578 | 설정 관리(다중 출처 병합, 정책 필터링) |
| `services/mcp/auth.ts` | 2,465 | OAuth 2.0 인증(XAA 교차 앱 액세스 포함) |
| `services/mcp/utils.ts` | 575 | 도구 필터링, 이름 해싱, Stale 감지 |
| `services/mcp/types.ts` | 258 | 타입 정의(Transport, ServerConfig, 연결 상태) |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | UI 폴딩 분류(Search/Read 도구 식별) |
| `tools/MCPTool/UI.tsx` | 402 | 도구 실행 결과 렌더링 |
| `services/mcp/channelPermissions.ts` | 240 | Channel 권한 중계 |
| `services/mcp/channelNotification.ts` | 316 | Channel 메시지 푸시 메커니즘 |
| `services/mcp/elicitationHandler.ts` | 313 | Elicitation(폼/URL 상호작용) 처리 |
| `skills/mcpSkillBuilders.ts` | 44 | Skill 빌더 레지스트리(의존성 그래프 분리) |

---

## 11.2 이론적 기반

### 프로토콜 기반 도구 확장 패턴

전통적인 플러그인 시스템은 보통 호스트 애플리케이션이 제공하는 SDK에 의존하며, 플러그인 개발자는 호스트의 내부 인터페이스를 알아야 한다. MCP는 **프로토콜 기반(protocol-driven)** 패턴을 채택한다: 호스트(Claude Code)와 플러그인(MCP 서버) 간의 모든 상호작용이 표준 JSON-RPC 메시지를 통해 이루어지며, 양측은 독립적으로 발전할 수 있다.

이것은 LSP(Language Server Protocol)의 설계 사상과 매우 일치한다:

| 차원 | LSP | MCP |
|------|-----|-----|
| 핵심 패턴 | 편집기 ↔ 언어 서버 | AI Agent ↔ 도구 서버 |
| 발견 메커니즘 | `initialize`에서 capabilities 교환 | `tools/list`, `resources/list`, `prompts/list` |
| 전송 층 | stdio, LSP over TCP | stdio, SSE, HTTP Streamable, WebSocket |
| 양방향 통신 | 지원 | 지원(notifications, elicitation) |
| 버전 협상 | 지원 | 지원(`protocolVersion`) |

LSP는 "모든 편집기가 모든 언어를 지원해야 한다"는 M×N 폭발 문제를 해결했다; MCP는 "모든 AI 도구가 모든 외부 서비스를 지원해야 한다"는 동류 문제를 해결한다.

### 클라이언트-서버 프로토콜의 설계 원칙

MCP의 두 가지 핵심 설계 선택이 Claude Code 구현에 깊은 영향을 미친다:

**기능 협상(Capability Negotiation)**: 서버는 연결 시 `ServerCapabilities`를 통해 지원하는 기능 하위 집합(`tools`, `prompts`, `resources`, `elicitation`, `experimental`)을 선언하며, 클라이언트는 서버가 선언한 기능만 호출한다. 이는 Claude Code가 각 서버 유형에 대한 특수 분기를 작성할 필요 없이, `capabilities` 확인을 통해 동작을 결정할 수 있음을 의미한다.

**도구 어노테이션(Tool Annotations)**: MCP 2025-03 버전은 `tool.annotations` 필드를 도입하여, 서버가 `readOnlyHint`, `destructiveHint`, `openWorldHint` 등의 의미론적 마커를 선언할 수 있게 했다. Claude Code는 이러한 마커를 직접 도구의 `isReadOnly()`, `isDestructive()`, `isOpenWorld()` 메서드에 매핑하여, 도구 이름의 정적 화이트리스트를 유지하지 않고도 보안 결정을 내릴 수 있다.

---

## 11.3 MCP 클라이언트 아키텍처

### MCPClient 클래스의 핵심 인터페이스

Claude Code는 MCP 클라이언트를 직접 구현하지 않고, `@modelcontextprotocol/sdk`가 제공하는 `Client` 클래스를 캡슐화한다. `connectToServer`는 핵심 진입 함수(`client.ts`)로, `lodash/memoize`를 사용하여 연결 수준 캐싱을 하며, 캐시 키는 `${name}-${jsonStringify(serverRef)}`다:

```typescript
// client.ts(약 540번 줄)
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... serverRef.type에 따라 transport 초기화
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... 연결, 타임아웃, 기능 협상
  },
  getServerCacheKey,
)
```

### 연결 관리(설정, 유지, 해제)

**연결 설정**: `connectToServer`는 `serverRef.type`에 따라 해당 transport를 생성한 후 `client.connect(transport)`를 발동하고, 30초 타임아웃을 설정한다(`getConnectionTimeoutMs()`, `MCP_TIMEOUT` 환경 변수로 재정의 가능):

```typescript
// client.ts(약 1000번 줄)
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

**연결 유지**: `client.onerror`와 `client.onclose`를 오버라이드하여 오류 감지와 자동 재연결을 구현한다. 원격 전송(SSE/HTTP)의 경우 `consecutiveConnectionErrors` 카운터를 유지하며, 연속 3회의 치명적 오류(`ECONNRESET`/`ETIMEDOUT`/`EPIPE` 등) 후 `closeTransportAndRejectPending`을 트리거하여 `client.close()`를 호출하고 모든 대기 중인 `callTool()`을 거부하며, memoize 캐시를 지워 다음 요청 시 자동 재연결이 이루어지도록 한다:

```typescript
// client.ts(약 1250번 줄)
client.onclose = () => {
  // 모든 관련 캐시 지우기, 다음 호출 시 재연결 트리거
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**세션 만료 처리**: HTTP 전송 MCP 서버는 HTTP 404 + JSON-RPC 오류 코드 `-32001`(Session Not Found)을 반환할 수 있다. Claude Code는 이 특정 오류 패턴을 감지하여 재연결을 트리거하고 `fetchToolsForClient.call()`에서 투명하게 재시도한다(최대 1회):

```typescript
// client.ts(약 150번 줄)
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**연결 해제**: stdio 전송은 3단계 신호 업그레이드를 사용한다: 먼저 `SIGINT`(100ms 대기), 그 다음 `SIGTERM`(400ms 대기), 마지막으로 `SIGKILL`. 총 해제 시간 상한은 600ms로, CLI 종료 차단을 방지한다.

### 도구 동적 발견 및 등록

`fetchToolsForClient`(LRU 캐시, 용량 20)는 서버에 `tools/list`를 전송하고 각 도구를 내부 `Tool` 인터페이스에 맞는 객체로 래핑한다:

- **명명 규칙**: `mcp__${normalizeNameForMCP(serverName)}__${toolName}`(밑줄 연결 형식)
- **설명 잘라내기**: `MAX_MCP_DESCRIPTION_LENGTH = 2048` 문자를 초과하는 description은 잘리고 `… [truncated]`가 추가되어, OpenAPI 생성 서버의 초장문이 컨텍스트를 오염시키는 것을 방지
- **권한 매핑**: `tool.annotations.readOnlyHint` → `isReadOnly()`, `tool.annotations.destructiveHint` → `isDestructive()`
- **폴딩 분류**: `classifyMcpToolForCollapse(serverName, toolName)`을 호출하여 Search/Read 류 도구인지 판단

마찬가지로, `fetchCommandsForClient`는 `prompts/list`를 전송하여 MCP Prompt를 `/명령` 객체로 변환한다; `fetchResourcesForClient`는 `resources/list`를 전송하여 리소스를 지원하는 서버에 `ListMcpResourcesTool`과 `ReadMcpResourceTool` 도구를 주입한다.

### 메시지 전송 층

Claude Code는 6가지 전송 유형을 지원한다:

| 유형 | 적용 시나리오 | Transport 클래스 |
|------|----------|-------------|
| `stdio` | 로컬 서브프로세스(대부분의 커뮤니티 서버) | `StdioClientTransport` |
| `sse` | 원격 SSE 서버(OAuth 포함) | `SSEClientTransport` |
| `sse-ide` | IDE 확장 내부 SSE(OAuth 없음) | `SSEClientTransport`(간소화 설정) |
| `http` | MCP Streamable HTTP(최신 사양) | `StreamableHTTPClientTransport` |
| `ws` | WebSocket 전송 | 커스텀 `WebSocketTransport` |
| `ws-ide` | IDE 확장 내부 WebSocket | `WebSocketTransport`(`X-Claude-Code-Ide-Authorization` 포함) |

특수 시나리오에서, Chrome Extension MCP 서버와 Computer Use MCP 서버는 **인프로세스 모드(In-Process)**로 실행되며, `createLinkedTransportPair()`로 메모리 파이프라인을 구축하여 ~325 MB의 서브프로세스 오버헤드를 피한다.

HTTP 전송에는 중요한 엔지니어링 세부사항이 있다: 각 POST 요청은 `Accept: application/json, text/event-stream` 헤더를 전달해야 한다(MCP Streamable HTTP 사양 요구). Claude Code는 `wrapFetchWithTimeout`을 통해 이 헤더를 통일적으로 주입하여 일부 런타임 환경에서 헤더가 누락되는 것을 방지한다:

```typescript
// client.ts(약 460번 줄)
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// wrapFetchWithTimeout 내:
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 MCP 설정 관리

### 서버 설정 형식

`types.ts`는 Zod를 사용하여 7가지 서버 설정 Schema를 정의하고, `z.union([...])` 으로 `McpServerConfigSchema`로 집계한다:

```typescript
// types.ts(28-115번 줄, 요약)
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // 하위 호환: type 필드 없음 = stdio
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

`ScopedMcpServerConfig`는 기본 설정 위에 `scope`(설정 출처)와 `pluginSource`(플러그인을 제공하는 출처 식별자) 필드를 추가하여 Channel 권한 검증에 사용한다.

### 다중 출처 설정 병합(enterprise > local > project > user > dynamic)

`getClaudeCodeMcpConfigs`(`config.ts`)는 다층 설정 병합을 구현하며, 우선순위는 높은 것에서 낮은 것 순으로:

1. **enterprise**(`managed-mcp.json`): 기업 독점 모드, 이 파일이 존재하면 다른 모든 출처를 차단
2. **local**(프로젝트 전용, 사용자 전역 설정에 저장, CWD에 바인딩)
3. **project**(`.mcp.json`, 디렉터리 트리 위로 순회, 가까운 것 우선)
4. **user**(전역 `~/.claude/config.json`의 `mcpServers` 필드)
5. **dynamic**(CLI `--mcp-config` 파라미터 런타임 주입)

Project 설정에는 추가적인 **사용자 승인 게이트**가 필요하다: `.mcp.json`의 서버를 처음 만나면 승인 대화 상자가 팝업된다. `getProjectMcpServerStatus()`가 `enabledMcpjsonServers`/`disabledMcpjsonServers` 설정을 읽어 `approved`/`rejected`/`pending`을 반환한다. 비대화형 모드(`-p` 파라미터, SDK 호출)에서 `isSettingSourceEnabled('projectSettings')`가 true이면 자동 승인된다.

설정 병합 후 **중복 제거**도 실행된다: Plugin 서버는 "시그니처"(stdio 서버는 명령 배열, 원격 서버는 URL)로 중복 제거되어 동일한 기반 서버에 두 번 연결되는 것을 방지한다; claude.ai Connector도 동일한 메커니즘으로 수동 설정과의 중복을 방지한다.

### 환경 변수 확장

설정 파일에서 `${ENV_VAR}` 문법을 사용할 수 있으며, `expandEnvVarsInString`(`config.ts`/`envExpansion.ts`)가 설정 읽기 시 확장한다. 정의되지 않은 변수는 `missingVars` 목록에 수집되어 사용자에게 보고된다.

---

## 11.5 MCP 인증 시스템

### OAuth 2.0 통합

`ClaudeAuthProvider`(`auth.ts`)는 MCP SDK의 `OAuthClientProvider` 인터페이스를 구현하며, 전체 OAuth 생명주기를 담당한다. 인증 흐름은 RFC 6749 인가 코드 흐름 + PKCE(Proof Key for Code Exchange)를 따르며, 로컬 HTTP 서버를 통해 콜백을 받는다:

1. **메타데이터 발견**: 먼저 RFC 9728(`/.well-known/oauth-protected-resource`)을 탐색하고, 실패하면 RFC 8414(`/.well-known/oauth-authorization-server`)로 폴백하고, 최종적으로 경로 인식 발견을 시도(하위 호환 유지)
2. **DCR(동적 클라이언트 등록)**: 첫 인증 시 OAuth 클라이언트를 자동 등록하고, `clientId`/`clientSecret`을 시스템 Keychain에 저장
3. **Token 교환**: 로컬 임의 포트로 인가 코드를 받고, access_token + refresh_token으로 교환
4. **Token 갱신**: `checkAndRefreshOAuthTokenIfNeeded()`를 통해 호출 전 만료를 감지하고 갱신하며, 실패 시 스마트 재시도

**Slack 호환 층**: 일부 OAuth 서버(특히 Slack)는 token 엔드포인트에서 HTTP 200을 반환하지만 오류 본문을 전달하여 RFC 6749를 위반한다. Claude Code는 `normalizeOAuthErrorBody`를 통해 이런 응답을 HTTP 400으로 재작성하여 SDK의 오류 분류 로직이 정상 작동하도록 한다:

```typescript
// auth.ts(약 250번 줄)
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // OAuthErrorResponse로 위장한 200인지 감지
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // Slack의 비표준 오류 코드 'invalid_refresh_token'을 'invalid_grant'로 표준화
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### 다중 인증 방식 지원

표준 OAuth 외에도, Claude Code는 다음을 지원한다:

- **Step-Up Auth**: 일부 작업은 권한 범위 확대가 필요하며, 서버가 HTTP 403과 새로운 scope 요구사항을 반환하면 Claude Code가 이를 감지하여 OAuth 흐름을 다시 트리거
- **XAA(Cross-App Access / SEP-990)**: 기업 시나리오에서 통합 IdP(OIDC 지원)를 통해 한 번 로그인하여 여러 MCP 서버에 권한을 부여하며, RFC 8693(Token Exchange) + RFC 7523(JWT Bearer) 복합 흐름을 채택. 각 서버에 별도의 브라우저 창을 팝업할 필요 없음
- **Static Headers**: 설정 파일이나 `headersHelper` 스크립트를 통해 정적 인증 헤더 주입(API Key 인증에 적합)

### Token 관리

Token 데이터 구조는 시스템 보안 저장소(macOS Keychain / Linux Secret Service)에 저장되며, 키는 `${serverName}|${SHA256(config)[:16]}`으로, 동일한 이름이지만 다른 설정을 가진 서버가 독립적인 token 슬롯을 사용하도록 보장한다.

`auth-cache`(`mcp-needs-auth-cache.json`)는 최근 401을 반환한 서버를 기록하며, TTL은 15분으로, 각 시작 시마다 반드시 실패할 서버를 반복 탐색하는 것을 방지한다. 캐시 읽기는 Promise 공유(`authCachePromise`)를 통해 일괄 연결 시 N번의 동시 읽기를 방지한다.

---

## 11.6 MCP 도구 실행

### MCPTool의 실행 흐름

LLM이 `mcp__slack__send_message` 호출을 결정하면 실행 흐름은 다음과 같다:

1. **라우팅**: `fetchToolsForClient`가 등록한 `call()` 함수가 LLM이 생성한 JSON input을 파라미터로 호출됨
2. **재연결 확인**: `ensureConnectedClient(client)`가 연결이 여전히 유효한지 확인하고, 필요시 재연결
3. **진행 알림**: `onProgress` 콜백을 통해 `mcp_progress: started` 이벤트 발송
4. **도구 호출**: `callMCPToolWithUrlElicitationRetry`(「`callMCPTool`을 캡슐화)가 서버에 `tools/call` 요청 전송
5. **결과 처리**: 이미지, 큰 바이너리 내용에 대한 특수 처리(디스크에 영속화, 참조 전달), 초대형 텍스트 내용에 대한 잘라내기
6. **진행 알림**: `mcp_progress: completed` 이벤트 발송(소요 시간 포함)

세션 만료의 투명 재시도 로직:

```typescript
// client.ts(약 2100번 줄)
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // 한 번 자동 재시도
    }
    throw error
  }
}
```

### classifyForCollapse — 도구 결과의 컨텍스트 폴딩 분류

`classifyForCollapse.ts`는 두 개의 정적 Set을 유지한다: `SEARCH_TOOLS`(약 100개 도구명)와 `READ_TOOLS`(약 300개 도구명)로, 40개 이상의 주류 MCP 서버(Slack, GitHub, Linear, Datadog, Sentry, Jira, Asana, Gmail, Grafana, PagerDuty 등)를 커버한다.

분류 규칙: 도구명이 먼저 `normalize()`를 거쳐(camelCase/kebab-case를 모두 snake_case로 변환), 두 Set에 있는지 조회한다:

```typescript
// classifyForCollapse.ts(587-598번 줄)
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

**설계 의도**: Search/Read 류 도구의 결과는 보통 길지만, 이후 LLM 추론에 대한 가치는 제한적이다(검색 중간 상태). 표시 후, UI 층은 대화 기록에서 이 결과를 기본적으로 접어 시각적 공간과 context window를 절약할 수 있다. 분류는 **보수적**이며(알 수 없는 도구는 접지 않음), **도구명만을 기반**으로 서버명을 구분하지 않는다. 주류 서버의 도구명은 인스턴스 간에 안정적인 식별자이기 때문이다.

### 권한 및 샌드박스 제어

MCP 도구 실행 전 `checkPermissions()`를 호출하며, 이 메서드는 `passthrough` 상태를 반환한다(즉, 항상 권한 프롬프트를 표시). 프롬프트 정보에는 사용자가 해당 도구명을 `allow` 규칙 목록에 추가하도록 제안하는 빠른 작업이 포함된다.

도구 호출 타임아웃은 `MCP_TOOL_TIMEOUT` 환경 변수로 제어하며, 기본값은 `100_000_000` 밀리초(약 27.8시간, 사실상 "무한")로, 시간이 걸리는 MCP 서버가 정상적으로 완료될 수 있도록 한다.

---

## 11.7 MCP Channel 시스템

Channel 시스템은 MCP의 확장 용도다: 외부 메시지 플랫폼(Telegram, Discord, iMessage, Slack 등)이 진행 중인 Claude Code 세션에 메시지를 푸시할 수 있게 한다(feature flag: `KAIROS`/`KAIROS_CHANNELS`, runtime gate: `tengu_harbor`).

### Channel 권한 관리

`channelPermissions.ts`는 **권한 위임** 메커니즘을 구현한다: Claude Code가 사용자 승인이 필요한 작업을 만날 때, Channel 서버를 통해 사용자 휴대폰에 동시에 프롬프트를 보낼 수 있으며, 사용자가 `yes <5자리 ID>`로 응답하면 서버가 이를 파싱하고 `notifications/claude/channel/permission` 이벤트로 Claude Code에 승인을 알린다.

5자리 ID는 25문자 알파벳(혼동 방지를 위해 `l` 제외)을 사용하며, FNV-1a 해시로 생성되고, 비속어 필터링이 포함되어 있다(`ID_AVOID_SUBSTRINGS` 목록, 약 24개 단어). 업무 메시지에 부적절한 내용이 나타나지 않도록 보장한다:

```typescript
// channelPermissions.ts(86-110번 줄)
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

Channel 서버는 동시에 `capabilities.experimental['claude/channel']`과 `capabilities.experimental['claude/channel/permission']` 두 기능을 선언해야 권한 중계자가 될 수 있으며, 보안 경계가 의도치 않게 개방되는 것을 방지한다.

### Channel 알림 메커니즘

`channelNotification.ts`는 인바운드 메시지를 받기 위한 완전한 게이트 로직(`gateChannelServer`)을 정의하며, 순차적으로 확인한다:

1. 서버 기능 선언(`claude/channel`)
2. Runtime 스위치(`tengu_harbor`)
3. OAuth 인증(claude.ai 계정 로그인만 지원, API Key 불가)
4. 팀/기업 정책(`channelsEnabled: true`)
5. 세션 `--channels` 파라미터(사용자가 명시적으로 신뢰를 선언한 Channel)
6. Marketplace 출처 검증(`slack@evil`이 `slack@anthropic`을 사칭하는 것 방지)

인바운드 메시지는 `<channel source="serverName" meta_key="value">content</channel>` 형식으로 세션 큐에 주입되며, `SleepTool` 폴링(약 1초 간격)으로 깨어난 후 모델이 응답 방법을 결정할 수 있다.

### Elicitation 처리

`elicitationHandler.ts`는 서버가 능동적으로 시작하는 상호작용 요청(MCP Elicitation 사양)을 처리한다. 두 가지 모드를 지원한다:

- **form 모드**: 서버가 사용자에게 양식 작성을 요청(`requestedSchema` 필드로 JSON Schema 정의)
- **url 모드**: 서버가 사용자에게 URL 방문을 통한 작업 완료 요청(예: OAuth 인가)

처리 흐름: 먼저 Hook 시스템을 실행(프로그래밍 방식 응답 가능), Hook 응답이 없으면 요청을 `AppState.elicitation.queue`에 큐잉하여 UI가 양식을 렌더링하거나 브라우저를 열기를 기다리고, 사용자 작업 후 `respond()` 콜백이 응답을 트리거한다:

```typescript
// elicitationHandler.ts(69-90번 줄)
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. 먼저 Hook 시도(프로그래밍 방식 응답)
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. UI 표시, 사용자 응답 대기
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

URL 모드는 `ElicitationCompleteNotificationSchema`도 지원한다: 서버가 작업 완료 후 Claude Code에 능동적으로 알리며, 해당 큐 항목이 `completed: true`로 표시되고 UI가 이에 따라 표시 상태를 업데이트한다.

---

## 11.8 MCP Skill 구축

`skills/mcpSkillBuilders.ts`는 매우 간결한 의존성 그래프 분리 모듈(44줄)로, 순환 의존성 문제를 해결한다:

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts  (순환)
```

해결책은 Write-Once 레지스트리 도입이다:

```typescript
// mcpSkillBuilders.ts(전체)
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

`loadSkillsDir.ts` 모듈은 초기화 시 `registerMCPSkillBuilders()`를 호출한다. 이것은 애플리케이션 시작 시(`commands.ts`의 정적 import를 통해) 미리 완료되어, MCP 서버 연결 시 레지스트리가 이미 준비되어 있도록 보장한다.

Skill 발견 메커니즘(Feature Flag: `MCP_SKILLS`): `fetchMcpSkillsForClient`는 서버의 `skill://` 리소스를 읽고, Markdown 형식의 skill 파일(frontmatter 메타데이터 포함)을 Skill Command로 파싱하여 `/serverName:skillName` 형식의 slash 명령으로 등록한다. MCP 서버가 재사용 가능한 AI 워크플로를 자동으로 제공할 수 있게 한다.

---

## 11.9 설계 결정 분석

### 커스텀 플러그인 프로토콜 대신 MCP를 채택한 이유

**문제**: Claude Code는 대량의 서드파티 도구 통합을 지원해야 하며, 커스텀 플러그인 API는 생태계 잠금을 초래할 것이다.

**결정**: MCP 개방형 표준을 채택하여 MCP 커뮤니티 생태계를 공유한다.

**이유**:
- 2025년 초 MCP 생태계에는 이미 수백 개의 오픈소스 서버가 있어 직접 재사용 가능
- 플러그인 개발자는 어떤 언어로도 MCP 서버를 구현할 수 있어(Python, Go, Java 등) TypeScript 생태계에 제한되지 않음
- Anthropic은 Claude Code의 개발자이자 MCP 사양의 주도자이므로, 사양 층에서 요구사항을 해결할 수 있어 클라이언트에 패치를 적용할 필요 없음

**대가**: 프로토콜 복잡성(인증, 전송 층 차이, 버전 호환성)이 모두 클라이언트에 전가되며, `auth.ts`의 2,465줄 중 상당 부분은 OAuth 사양 결함과 각 벤더의 비준수 구현을 처리하는 것이다.

### MCP 인증의 복잡성 처리

**문제**: MCP 사양의 인증 설명은 "권고적"이며, 실제 서버 구현은 천차만별이다(Slack은 200으로 오류를 반환하고, 일부 서버는 Token Revocation을 구현하지 않는 등).

**결정**: `auth.ts`에 완전한 호환 층을 구축하여 알려진 비준수 동작을 처리한다.

핵심 전략:
- `normalizeOAuthErrorBody`: 200으로 위장된 오류 응답 처리
- `NONSTANDARD_INVALID_GRANT_ALIASES`: Slack 등의 비표준 오류 코드 표준화
- RFC 7009 revocation의 이중 시도(먼저 표준 방식, 401을 받으면 Bearer로 재시도)
- 두 가지 auth 발견 경로(RFC 9728 → RFC 8414 → 경로 인식 폴백)

### 도구 폴딩 분류의 설계 고려사항

**문제**: MCP 도구 결과는 매우 길 수 있으며(검색 결과, 로그 출력), 대화 기록에 대량 인라인이 있으면 가독성이 떨어지고 context window가 낭비된다.

**결정**: 휴리스틱 분류 대신 명시적인 도구명 화이트리스트를 채택하여, 알려진 Search/Read 류 도구 결과를 UI 층에서 기본적으로 접는다.

**트레이드오프**:
- 장점: 확실성이 강하고, 오판 없음, 알려진 도구의 동작 일관성
- 단점: 정적 목록 유지 필요(`classifyForCollapse.ts`는 현재 40개 이상의 서버를 커버), 새 서버 추가 시 수동 업데이트 필요
- 보수적 전략(알 수 없는 도구는 접지 않음)은 새 서버가 잘못된 폴딩으로 정보를 잃는 것을 방지

---

## 11.10 이식 가능한 패턴

Claude Code MCP 통합의 엔지니어링 실천에서 도출한 패턴들로, 외부 도구나 서비스를 통합해야 하는 다른 시스템에 적용 가능하다:

**1. 유형 판단 대신 기능 협상 우선**
프로토콜 클라이언트를 구현할 때, 서버 유형이나 이름으로 if-else 분기를 만드는 것이 아니라 항상 `capabilities` 확인을 통해 동작을 결정하라. 이는 새로운 기능 지원을 점진적으로 추가할 수 있게 하며, 기존 로직에 영향을 미치지 않는다.

**2. Memoize + Cache Invalidation 패턴**
연결, 도구 발견 등 비용이 많이 드는 작업에는 memoize 캐싱을 사용하되, 연결이 끊어질 때 즉시 캐시를 무효화해야 한다(`client.onclose`에서 모든 관련 캐시 항목 지우기). LRU 캐시(용량 20)를 사용하여 메모리 누수를 방지한다.

**3. Write-Once 레지스트리로 순환 의존성 해결**
모듈 A가 모듈 B의 함수에 의존하고 모듈 B가 간접적으로 모듈 A에 의존할 때, 외부 의존성이 없는 레지스트리 모듈을 도입하여 애플리케이션 초기화 시 모듈 B가 레지스트리에 구현을 주입하고 모듈 A가 레지스트리에서 읽도록 한다. `mcpSkillBuilders.ts`는 최소한으로 복사 가능한 템플릿이다.

**4. 프로토콜 호환 층 집중 관리**
OAuth/HTTP 사양의 비준수 구현은 한 곳(예: `auth.ts`)에서 처리해야 하며, 호출 위치에 분산되어서는 안 된다. `normalizeOAuthErrorBody`는 이 패턴의 전형적인 예시다: 전송 층에서 통일적으로 처리하는 순수 함수로, 호출 위치는 서버가 사양을 준수하는지 걱정할 필요 없다.

**5. 동시 연결의 계층화 제한**
다른 유형의 작업은 리소스 소비가 다르다. stdio 서버는 서브프로세스 fork가 필요하고(CPU + 메모리 집약), 네트워크 서버는 TCP 연결만 수립하면 된다(I/O 집약). 두 유형의 작업에 다른 동시성 제한을 사용(local: 3, remote: 20)하면 시스템을 보호하면서 처리량을 최대화할 수 있다.

**6. needs-auth 상태의 이중 확인**
인증이 필요한 원격 서비스의 경우, **시간 기반 캐시**(15분 TTL)와 **상태 기반 확인**(discovery 상태가 있지만 token 없음) 이중 판단을 결합하여 성공할 가능성이 없는 연결을 건너뛰어 시작 시마다의 무효 탐색 지연을 피한다.

---

## 11.11 소스 코드 인덱스

| 핵심 구현 | 파일:줄번호 |
|---------|---------|
| `MCPServerConnection` 유니온 타입 정의 | `services/mcp/types.ts:170-200` |
| `ConfigScope` 열거형(7개 출처) | `services/mcp/types.ts:10-22` |
| `connectToServer` 주 함수(memoized) | `services/mcp/client.ts:540` |
| Transport 초기화 분기(6가지 유형) | `services/mcp/client.ts:570-930` |
| 연결 타임아웃 처리 | `services/mcp/client.ts:1000-1040` |
| 연결 끊김 오류 감지 및 재연결 트리거 | `services/mcp/client.ts:1200-1320` |
| stdio 3단계 종료(SIGINT/SIGTERM/SIGKILL) | `services/mcp/client.ts:1370-1490` |
| `fetchToolsForClient`(도구 등록) | `services/mcp/client.ts:1830-2050` |
| `getMcpToolsCommandsAndResources`(일괄 연결 진입점) | `services/mcp/client.ts:2580` |
| `isMcpSessionExpiredError` | `services/mcp/client.ts:150-165` |
| `wrapFetchWithTimeout`(HTTP Accept 주입) | `services/mcp/client.ts:450-510` |
| 설정 출처 병합 `getClaudeCodeMcpConfigs` | `services/mcp/config.ts:1050` |
| 기업 독점 모드 판단 | `services/mcp/config.ts:1080-1090` |
| Project 서버 승인 게이트 | `services/mcp/utils.ts:210-250` |
| 플러그인 서버 중복 제거(시그니처 해시) | `services/mcp/config.ts:215-270` |
| `ClaudeAuthProvider`(OAuth 핵심) | `services/mcp/auth.ts:500+` |
| `normalizeOAuthErrorBody`(Slack 호환) | `services/mcp/auth.ts:250-290` |
| `performMCPXaaAuth`(교차 앱 인증) | `services/mcp/auth.ts:700+` |
| `getServerKey`(Token 저장 키 생성) | `services/mcp/auth.ts:390-405` |
| `hasMcpDiscoveryButNoToken`(빠른 실패) | `services/mcp/auth.ts:420-435` |
| `classifyMcpToolForCollapse`(폴딩 분류) | `tools/MCPTool/classifyForCollapse.ts:587-598` |
| `SEARCH_TOOLS` / `READ_TOOLS` 화이트리스트 | `tools/MCPTool/classifyForCollapse.ts:20-585` |
| `shortRequestId`(Channel 권한 ID) | `services/mcp/channelPermissions.ts:140-160` |
| `gateChannelServer`(6층 게이트) | `services/mcp/channelNotification.ts:190-310` |
| `registerElicitationHandler` | `services/mcp/elicitationHandler.ts:65-150` |
| `registerMCPSkillBuilders`(순환 의존성 분리) | `skills/mcpSkillBuilders.ts:30-44` |
