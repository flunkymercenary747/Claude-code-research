# 第 11 章：MCP 統合

## 11.1 概要と位置づけ

### MCP とは何か

MCP（Model Context Protocol）は Anthropic が主導して設計したオープンプロトコルであり、AI アプリケーションと外部ツールサービス間の標準化された通信フォーマットを定義する。本質的には JSON-RPC 2.0 プロトコルであり、複数のトランスポート層（stdio、SSE、HTTP Streamable、WebSocket）上で動作し、ツールの発見（`tools/list`）、ツールの呼び出し（`tools/call`）、リソース管理（`resources/list`/`resources/read`）、Prompt テンプレート（`prompts/list`/`prompts/get`）などの標準メッセージフォーマットを規定している。

### Claude Code における MCP の役割

Claude Code の組み込みツールセット（Bash、Read、Edit など）はファイルシステムとローカル開発シナリオをカバーしている。MCP の設計上の位置づけは**オープンなツール拡張インターフェース**である：あらゆるサードパーティサービス（Slack、GitHub、Jira、データベース、ブラウザ自動化など）が MCP サーバーを実装でき、Claude Code が標準プロトコルで接続後にこれらの外部機能を呼び出せる。コアコードを修正する必要はない。

アーキテクチャ上、Claude Code は純粋な **MCP クライアント**であり、MCP サーバー機能は実装していない（`roots/list` リクエストへの応答でサーバーに作業ディレクトリを通知することを除く）。各接続された MCP サーバーのツールは `mcp__<serverName>__<toolName>` フォーマットの Tool オブジェクトとして動的に登録され、組み込みツールと同じ実行フレームワークを共有する。

### コードの規模

MCP 統合は約 12,310 行の TypeScript コードを含み、以下のファイルに分散している：

| ファイル | 行数 | 責任 |
|------|------|------|
| `services/mcp/client.ts` | 3,348 | 接続管理、ツール発見、実行のコア |
| `services/mcp/config.ts` | 1,578 | 設定管理（複数ソースの統合、ポリシーフィルタリング） |
| `services/mcp/auth.ts` | 2,465 | OAuth 2.0 認証（XAA クロスアプリアクセスを含む） |
| `services/mcp/utils.ts` | 575 | ツールフィルタリング、名前ハッシュ、Stale 検出 |
| `services/mcp/types.ts` | 258 | 型定義（Transport、ServerConfig、接続状態） |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | UI 折りたたみ分類（Search/Read ツールの識別） |
| `tools/MCPTool/UI.tsx` | 402 | ツール実行結果のレンダリング |
| `services/mcp/channelPermissions.ts` | 240 | Channel 権限の中継 |
| `services/mcp/channelNotification.ts` | 316 | Channel メッセージのプッシュ機構 |
| `services/mcp/elicitationHandler.ts` | 313 | Elicitation（フォーム/URL インタラクション）の処理 |
| `skills/mcpSkillBuilders.ts` | 44 | Skill ビルダーレジストリ（依存グラフの分離） |

---

## 11.2 理論的基礎

### プロトコル駆動のツール拡張パターン

従来のプラグインシステムは通常、ホストアプリが提供する SDK に依存しており、プラグイン開発者はホストの内部インターフェースを理解する必要がある。MCP は**プロトコル駆動（protocol-driven）**パターンを採用している：ホスト（Claude Code）とプラグイン（MCP サーバー）の間のすべてのインタラクションは標準の JSON-RPC メッセージで完了し、両者は独立して進化できる。

これは LSP（Language Server Protocol）の設計思想と非常に一致している：

| 次元 | LSP | MCP |
|------|-----|-----|
| 核心パターン | エディター ↔ 言語サーバー | AI Agent ↔ ツールサーバー |
| 発見機構 | `initialize` で capabilities を交換 | `tools/list`、`resources/list`、`prompts/list` |
| トランスポート層 | stdio、LSP over TCP | stdio、SSE、HTTP Streamable、WebSocket |
| 双方向通信 | サポート | サポート（notifications、elicitation） |
| バージョンネゴシエーション | サポート | サポート（`protocolVersion`） |

LSP は「各エディターが各言語に対応する必要がある」という M×N の爆発問題を解決した。MCP は「各 AI ツールが各外部サービスに対応する必要がある」という同様の問題を解決する。

### クライアント・サーバープロトコルの設計原則

MCP の二つの重要な設計選択が Claude Code の実装に深い影響を与えている：

**能力ネゴシエーション（Capability Negotiation）**：サーバーは接続時に `ServerCapabilities` を通じてサポートする機能のサブセット（`tools`、`prompts`、`resources`、`elicitation`、`experimental`）を宣言し、クライアントはサーバーが宣言した機能のみを呼び出す。これにより Claude Code は各サーバータイプに対して特殊な分岐を書く必要がなく、`capabilities` チェックで統一的に動作を決定できる。

**ツールアノテーション（Tool Annotations）**：MCP 2025-03 バージョンで `tool.annotations` フィールドが導入された。サーバーは `readOnlyHint`、`destructiveHint`、`openWorldHint` などのセマンティックマーカーを宣言できる。Claude Code はこれらのマーカーをツールの `isReadOnly()`、`isDestructive()`、`isOpenWorld()` メソッドに直接マッピングする。ツール名の静的ホワイトリストを維持せずに安全上の判断ができる。

---

## 11.3 MCP クライアントアーキテクチャ

### MCPClient クラスの核心インターフェース

Claude Code は MCP クライアントを直接実装せず、`@modelcontextprotocol/sdk` が提供する `Client` クラスをラップしている。`connectToServer` が核心エントリー関数であり（`client.ts`）、`lodash/memoize` で接続レベルのキャッシュを行う。キャッシュキーは `${name}-${jsonStringify(serverRef)}`：

```typescript
// client.ts（約第 540 行）
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... serverRef.type に基づいて transport を初期化
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... 接続、タイムアウト、能力ネゴシエーション
  },
  getServerCacheKey,
)
```

### 接続管理（確立、維持、切断）

**接続の確立**：`connectToServer` は `serverRef.type` に基づいて対応する transport を作成し、`client.connect(transport)` を開始して 30 秒のタイムアウトを設定する（`getConnectionTimeoutMs()`、`MCP_TIMEOUT` 環境変数で上書き可能）：

```typescript
// client.ts（約第 1000 行）
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

**接続の維持**：`client.onerror` と `client.onclose` を上書きすることでエラー検出と自動再接続を実現する。リモートトランスポート（SSE/HTTP）については `consecutiveConnectionErrors` カウンターを維持し、3 回連続でターミナルエラー（`ECONNRESET`/`ETIMEDOUT`/`EPIPE` など）が発生すると `closeTransportAndRejectPending` をトリガーし、`client.close()` を呼び出してすべての保留中の `callTool()` を拒否し、memoize キャッシュをクリアして次のリクエスト時に自動再接続する：

```typescript
// client.ts（約第 1250 行）
client.onclose = () => {
  // 関連するすべてのキャッシュをクリア、次の呼び出し時に再接続をトリガー
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**セッション期限切れの処理**：HTTP トランスポートの MCP サーバーは HTTP 404 + JSON-RPC エラーコード `-32001`（Session Not Found）を返すことがある。Claude Code はこの特定のエラーパターンを検出して再接続をトリガーし、`fetchToolsForClient.call()` で透過的にリトライする（最大 1 回）：

```typescript
// client.ts（約第 150 行）
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**接続の切断**：stdio トランスポートは三段階のシグナルエスカレーションを使用する：まず `SIGINT`（100ms 待機）、次に `SIGTERM`（400ms 待機）、最後に `SIGKILL`。切断時間の合計上限は 600ms で、CLI の終了をブロックしない。

### ツールの動的発見と登録

`fetchToolsForClient`（LRU キャッシュ、容量 20）はサーバーに `tools/list` を送信し、各ツールを内部の `Tool` インターフェースに準拠したオブジェクトとしてラップする：

- **命名規則**：`mcp__${normalizeNameForMCP(serverName)}__${toolName}`（アンダースコア連結形式）
- **説明の切り捨て**：`MAX_MCP_DESCRIPTION_LENGTH = 2048` 文字を超える description は切り捨てられ、`… [truncated]` が付加される。OpenAPI 生成サーバーの超長ドキュメントがコンテキストを汚染するのを防ぐ
- **権限マッピング**：`tool.annotations.readOnlyHint` → `isReadOnly()`、`tool.annotations.destructiveHint` → `isDestructive()`
- **折りたたみ分類**：`classifyMcpToolForCollapse(serverName, toolName)` を呼び出して Search/Read 類ツールかどうかを判断

同様に、`fetchCommandsForClient` は `prompts/list` を送信して MCP Prompt を `/コマンド` オブジェクトに変換し、`fetchResourcesForClient` は `resources/list` を送信してリソースをサポートするサーバーに `ListMcpResourcesTool` と `ReadMcpResourceTool` を注入する。

### メッセージトランスポート層

Claude Code は 6 種のトランスポートタイプをサポートする：

| タイプ | 適用シナリオ | Transport クラス |
|------|----------|-------------|
| `stdio` | ローカルサブプロセス（ほとんどのコミュニティサーバー） | `StdioClientTransport` |
| `sse` | リモート SSE サーバー（OAuth あり） | `SSEClientTransport` |
| `sse-ide` | IDE 拡張内部 SSE（OAuth なし） | `SSEClientTransport`（簡略化された設定） |
| `http` | MCP Streamable HTTP（最新仕様） | `StreamableHTTPClientTransport` |
| `ws` | WebSocket トランスポート | カスタム `WebSocketTransport` |
| `ws-ide` | IDE 拡張内部 WebSocket | `WebSocketTransport`（`X-Claude-Code-Ide-Authorization` あり） |

特殊なシナリオでは、Chrome Extension MCP サーバーと Computer Use MCP サーバーが**プロセス内モード（In-Process）**で動作し、`createLinkedTransportPair()` でメモリパイプを確立して、約 325 MB のサブプロセスオーバーヘッドを回避する。

HTTP トランスポートには重要なエンジニアリングの詳細がある：各 POST リクエストに `Accept: application/json, text/event-stream` ヘッダーが必要（MCP Streamable HTTP 仕様の要件）。Claude Code は `wrapFetchWithTimeout` を通じてこのヘッダーを統一的に注入し、一部のランタイム環境でヘッダーが失われるのを防ぐ：

```typescript
// client.ts（約第 460 行）
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// wrapFetchWithTimeout 内：
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 MCP 設定管理

### サーバー設定フォーマット

`types.ts` は Zod を使用して 7 種のサーバー設定スキーマを定義し、`z.union([...])` で `McpServerConfigSchema` に集約している：

```typescript
// types.ts（第 28-115 行、概要）
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // 後方互換性：type フィールドなし = stdio
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

`ScopedMcpServerConfig` は基本設定に `scope`（設定の由来）と `pluginSource`（プラグインを提供した由来識別子）フィールドを追加し、Channel の権限検証に使用する。

### 複数ソースの設定統合（enterprise > local > project > user > dynamic）

`getClaudeCodeMcpConfigs`（`config.ts`）は複数層の設定統合を実装しており、優先度は高い順から：

1. **enterprise**（`managed-mcp.json`）：エンタープライズ専用モード、このファイルが存在する場合は他のすべてのソースを遮断
2. **local**（プロジェクトプライベート、ユーザーのグローバル設定内に保存、CWD にバインド）
3. **project**（`.mcp.json`、ディレクトリツリーを上位に走査、近い方が優先）
4. **user**（グローバル `~/.claude/config.json` の `mcpServers` フィールド）
5. **dynamic**（CLI `--mcp-config` パラメーターでランタイム時に注入）

Project 設定には追加の**ユーザー承認ゲート**が必要である：`.mcp.json` 内のサーバーに初めて遭遇すると、承認ダイアログが表示される。`getProjectMcpServerStatus()` は `enabledMcpjsonServers`/`disabledMcpjsonServers` の設定を読んで `approved`/`rejected`/`pending` を返す。非インタラクティブモード（`-p` パラメーター、SDK 呼び出し）かつ `isSettingSourceEnabled('projectSettings')` の場合は自動承認。

設定の統合後に**重複排除**も実行される：Plugin サーバーは「署名」（stdio サーバーはコマンド配列、リモートサーバーは URL）で重複排除し、同じ基盤サービスへの二重接続を防ぐ。claude.ai Connector も同じ機構で手動設定との重複を回避する。

### 環境変数の展開

設定ファイルでは `${ENV_VAR}` 構文を使用でき、`expandEnvVarsInString`（`config.ts`/`envExpansion.ts`）が設定を読み込む際に展開する。未定義の変数は `missingVars` リストに収集されてユーザーに報告される。

---

## 11.5 MCP 認証システム

### OAuth 2.0 の統合

`ClaudeAuthProvider`（`auth.ts`）は MCP SDK の `OAuthClientProvider` インターフェースを実装し、OAuth の完全なライフサイクルを担う。認証フローは RFC 6749 の認可コードフロー + PKCE（Proof Key for Code Exchange）に準拠し、ローカル HTTP サーバーでコールバックを受け取る：

1. **メタデータ発見**：まず RFC 9728（`/.well-known/oauth-protected-resource`）を探索し、失敗すれば RFC 8414（`/.well-known/oauth-authorization-server`）にフォールバック、最終的にパスを意識した発見を試みる（後方互換性を維持）
2. **DCR（動的クライアント登録）**：初回認証時に OAuth クライアントを自動登録し、`clientId`/`clientSecret` をシステムの Keychain に保存
3. **Token 交換**：ローカルのランダムポートで認証コードを受け取り、access_token + refresh_token に交換
4. **Token 更新**：`checkAndRefreshOAuthTokenIfNeeded()` を通じて呼び出し前に期限切れを検出して更新し、失敗時はスマートリトライ

**Slack 互換層**：一部の OAuth サーバー（特に Slack）はトークンエンドポイントで HTTP 200 にエラーボディを添付して返し、RFC 6749 の期待に違反している。Claude Code は `normalizeOAuthErrorBody` を通じてこのようなレスポンスを HTTP 400 に書き換え、SDK のエラー分類ロジックが正常に動作するようにする：

```typescript
// auth.ts（約第 250 行）
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // OAuthErrorResponse を 200 に偽装しているかどうかを検出
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // Slack の非標準エラーコード 'invalid_refresh_token' を 'invalid_grant' に標準化
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### 複数の認証方式のサポート

標準 OAuth に加えて、Claude Code は以下もサポートする：

- **Step-Up Auth**：一部の操作で権限スコープの昇格が必要な場合、サーバーが HTTP 403 と新しいスコープ要件を返し、Claude Code が検出して OAuth フローを再開する
- **XAA（Cross-App Access / SEP-990）**：エンタープライズシナリオで統合 IdP（OIDC をサポート）を通じて一度ログインするだけで複数の MCP サーバーに認可できる。RFC 8693（Token Exchange）+ RFC 7523（JWT Bearer）の複合フローを採用し、各サーバーに対して個別にブラウザウィンドウを表示する必要がない
- **Static Headers**：設定ファイルまたは `headersHelper` スクリプトを通じて静的認証ヘッダーを注入（API Key 認証に適用）

### Token 管理

Token データはシステムのセキュアストレージ（macOS Keychain / Linux Secret Service）に保存され、キーは `${serverName}|${SHA256(config)[:16]}`。同名でも設定が異なるサーバーは独立した Token スロットを使用する。

`auth-cache`（`mcp-needs-auth-cache.json`）は最近 401 を返したサーバーを記録し、TTL は 15 分。起動時に必ず失敗するサーバーへの繰り返し探索を避ける。キャッシュの読み込みは Promise で共有（`authCachePromise`）し、バッチ接続時に同一ファイルに N 回並行アクセスするのを防ぐ。

---

## 11.6 MCP ツールの実行

### MCPTool の実行フロー

LLM が `mcp__slack__send_message` の呼び出しを決定したとき、実行フローは以下の通り：

1. **ルーティング**：`fetchToolsForClient` が登録した `call()` 関数が呼び出され、引数は LLM が生成した JSON 入力
2. **再接続チェック**：`ensureConnectedClient(client)` が接続が依然として有効かをチェックし、必要に応じて再接続
3. **進捗通知**：`onProgress` コールバックを通じて `mcp_progress: started` イベントを発行
4. **ツール呼び出し**：`callMCPToolWithUrlElicitationRetry`（`callMCPTool` をラップ）がサーバーに `tools/call` リクエストを送信
5. **結果処理**：画像、大きなバイナリコンテンツの特殊処理（ディスクに永続化して参照を渡す）、超大型テキストコンテンツの切り捨て
6. **進捗通知**：`mcp_progress: completed` イベントを発行（経過時間を含む）

セッション期限切れの透過的リトライロジック：

```typescript
// client.ts（約第 2100 行）
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // 一度だけ自動リトライ
    }
    throw error
  }
}
```

### classifyForCollapse — ツール結果のコンテキスト折りたたみ分類

`classifyForCollapse.ts` は二つの静的 Set を維持している：`SEARCH_TOOLS`（約 100 個のツール名）と `READ_TOOLS`（約 300 個のツール名）。これらは 40 以上の主要 MCP サーバー（Slack、GitHub、Linear、Datadog、Sentry、Jira、Asana、Gmail、Grafana、PagerDuty など）をカバーする。

分類ルール：ツール名をまず `normalize()`（camelCase/kebab-case を snake_case に統一変換）し、次に二つの Set に含まれるかを確認する：

```typescript
// classifyForCollapse.ts（第 587-598 行）
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

**設計意図**：Search/Read 類ツールの結果は通常長いが、後続の LLM 推論に対する価値は限られている（検索の中間状態）。マーク後、UI 層はこれらの結果を会話履歴で折りたため、視覚的なスペースとコンテキストウィンドウを節約できる。分類は**保守的**（未知のツールは折りたたまない）であり、かつ**ツール名のみに基づく**。サーバー名は区別しない。主要サーバーのツール名はインスタンス間で安定した識別子だからだ。

### 権限とサンドボックス制御

MCP ツールの実行前に `checkPermissions()` を呼び出す。このメソッドは `passthrough` 状態を返す（つまり常に権限プロンプトを表示する必要がある）。プロンプトにはツール名を `allow` ルールリストに追加する操作を提案するショートカットが含まれる。

ツール呼び出しのタイムアウトは `MCP_TOOL_TIMEOUT` 環境変数で制御され、デフォルトは `100_000_000` ミリ秒（約 27.8 時間、実質「無制限」）。時間のかかる操作の MCP サーバーが正常に完了できるようにするためだ。

---

## 11.7 MCP Channel システム

Channel システムは MCP の拡張用途である：外部メッセージプラットフォーム（Telegram、Discord、iMessage、Slack など）が進行中の Claude Code セッションにメッセージをプッシュできるようにする（feature flag: `KAIROS`/`KAIROS_CHANNELS`、runtime gate: `tengu_harbor`）。

### Channel 権限管理

`channelPermissions.ts` は**権限委譲**機構を実装している：Claude Code がユーザー承認が必要な操作に遭遇した際、Channel サーバーを通じてユーザーの携帯に通知を送ることができる。ユーザーが `yes <5文字ID>` と返信すると、サーバーが解析して `notifications/claude/channel/permission` イベントで Claude Code に承認を通知する。

5 文字 ID は 25 文字のアルファベット（`l` を省いて `1`/`I` との混同を防ぐ）を使用し、FNV-1a ハッシュで生成される。不適切な内容フィルター（`ID_AVOID_SUBSTRINGS` リスト、約 24 語）を含み、業務メッセージに不適切な内容が出現しないことを保証する：

```typescript
// channelPermissions.ts（第 86-110 行）
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

Channel サーバーは `capabilities.experimental['claude/channel']` と `capabilities.experimental['claude/channel/permission']` の両方を宣言しなければ権限中継者になれない。意図せずセキュリティ境界が開放されるのを防ぐためだ。

### Channel 通知機構

`channelNotification.ts` は受信メッセージの完全なゲートロジック（`gateChannelServer`）を定義しており、順に確認する：

1. サーバーの能力宣言（`claude/channel`）
2. Runtime スイッチ（`tengu_harbor`）
3. OAuth 認証（claude.ai アカウントのみ、API Key はサポートしない）
4. チーム/エンタープライズポリシー（`channelsEnabled: true`）
5. セッションの `--channels` パラメーター（ユーザーが明示的に信頼した Channel）
6. Marketplace ソースの検証（`slack@evil` が `slack@anthropic` を偽装するのを防ぐ）

受信メッセージは `<channel source="serverName" meta_key="value">content</channel>` フォーマットでセッションキューに注入される。`SleepTool` のポーリング（約 1 秒間隔）が起動した後、モデルが対応方法を決定できる。

### Elicitation の処理

`elicitationHandler.ts` はサーバーが積極的に開始するインタラクションリクエスト（MCP Elicitation 仕様）を処理する。二種類のモードをサポートする：

- **form モード**：サーバーがユーザーにフォームへの入力を要求（`requestedSchema` フィールドが JSON Schema を定義）
- **url モード**：サーバーがユーザーに URL へのアクセスを要求して操作を完了させる（OAuth 認可など）

処理フロー：まず Hook システムを実行（プログラマブルな応答）、Hook が応答しない場合はリクエストを `AppState.elicitation.queue` に追加して UI がフォームをレンダリングするかブラウザを開くのを待ち、ユーザー操作後に `respond()` コールバックが応答をトリガーする：

```typescript
// elicitationHandler.ts（第 69-90 行）
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. まず Hook を試みる（プログラマブルな応答）
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. UI を表示し、ユーザー応答を待つ
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

URL モードは `ElicitationCompleteNotificationSchema` もサポートしている：サーバーが操作を完了後に Claude Code に積極的に通知し、対応するキューアイテムに `completed: true` を付与して UI が表示状態を更新する。

---

## 11.8 MCP Skill の構築

`skills/mcpSkillBuilders.ts` は極めてシンプルな依存グラフ分離モジュール（44 行）であり、循環依存の問題を解決している：

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts  (循環)
```

解決策は一度だけ書き込めるレジストリ（Write-Once Registry）の導入：

```typescript
// mcpSkillBuilders.ts（全文）
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

`loadSkillsDir.ts` モジュールの初期化時に `registerMCPSkillBuilders()` を呼び出す。これはアプリ起動時（`commands.ts` の静的 import を通じて）に事前に完了し、MCP サーバーの接続時にはレジストリが既に準備できていることを保証する。

Skill の発見機構（Feature Flag: `MCP_SKILLS`）：`fetchMcpSkillsForClient` がサーバーの `skill://` リソースを読み込み、Markdown フォーマット（frontmatter メタデータを含む）の skill ファイルを解析して Skill Command として登録し、`/serverName:skillName` フォーマットのスラッシュコマンドとして提供する。これにより MCP サーバーが自動的に再利用可能な AI ワークフローを提供できる。

---

## 11.9 設計上の意思決定分析

### なぜカスタムプラグインプロトコルではなく MCP を採用したのか

**問題**：Claude Code は大量のサードパーティツール統合をサポートする必要があり、カスタムプラグイン API はエコシステムのロックインを生む。

**決定**：MCP オープン標準を採用し、MCP コミュニティエコシステムと共有する。

**理由**：
- 2025 年初頭に MCP エコシステムには数百のオープンソースサーバーが直接利用可能だった
- プラグイン開発者は任意の言語（Python、Go、Java など）で MCP サーバーを実装でき、TypeScript エコシステムに縛られない
- Anthropic は Claude Code の開発者でもあり、MCP 仕様のリード者でもある。クライアントにパッチを当てるのではなく、仕様レベルでニーズを解決できる

**コスト**：プロトコルの複雑さ（認証、トランスポート層の差異、バージョン互換性）はすべてクライアントが負担する。`auth.ts` の 2,465 行の多くは OAuth 仕様の欠陥や各ベンダーの非準拠実装を処理した結果だ。

### MCP 認証の複雑さへの対処

**問題**：MCP 仕様の認証の記述は「推奨的」であり、実際のサーバー実装は千差万別（Slack が 200 でエラーを返す、一部のサーバーが Token Revocation を実装しないなど）。

**決定**：`auth.ts` に完全な互換層を構築し、既知の非準拠動作を処理する。

重要な戦略：
- `normalizeOAuthErrorBody`：200 に偽装したエラーレスポンスの処理
- `NONSTANDARD_INVALID_GRANT_ALIASES`：Slack などの非標準エラーコードの標準化
- RFC 7009 revocation の二重試行（まず標準方式、401 を受け取ったら Bearer で再試行）
- 二つの auth 発見パス（RFC 9728 → RFC 8414 → パスを意識したフォールバック）

### ツール折りたたみ分類の設計考慮

**問題**：MCP ツールの結果は非常に長くなる可能性があり（検索結果、ログ出力）、大量がインライン表示されると可読性が低下し、コンテキストウィンドウが無駄になる。

**決定**：ヒューリスティックな分類ではなく、明示的なツール名のホワイトリストを採用し、既知の Search/Read 類ツールの結果を UI 層でデフォルト折りたたみにする。

**トレードオフ**：
- 利点：確定性が高く、誤判断がなく、既知ツールの動作が一貫している
- 欠点：静的リストの維持が必要（`classifyForCollapse.ts` は現在 40 以上のサーバーをカバー）、新しいサーバーは手動更新が必要
- 保守的戦略（未知のツールは折りたたまない）により、新しいサーバーが誤った折りたたみで情報を失うことはない

---

## 11.10 移植可能なパターン

以下のパターンは Claude Code MCP 統合のエンジニアリング実践に由来し、外部ツールやサービスを統合する必要がある他のシステムに適用できる：

**1. 型判断より能力ネゴシエーションを優先する**
プロトコルクライアントを実装する際は、サーバータイプや名前で if-else 分岐するのではなく、常に `capabilities` チェックで動作を決定する。これにより新しい能力のサポートを増分的に追加でき、既存のロジックに影響しない。

**2. Memoize + Cache Invalidation パターン**
接続やツール発見などのコストの高い操作には memoize キャッシュを使用するが、接続が切断された際には即座に無効化しなければならない（`client.onclose` ですべての関連キャッシュエントリをクリア）。LRU キャッシュ（容量 20）でメモリリークを防ぐ。

**3. 一度だけ書き込むレジストリで循環依存を解決する**
モジュール A がモジュール B の関数に依存し、モジュール B が間接的にモジュール A に依存する場合、外部依存がゼロのレジストリモジュールを導入する。アプリの初期化時にモジュール B がレジストリに実装を注入し、モジュール A はレジストリから読み取る。`mcpSkillBuilders.ts` が最小限の再利用可能なテンプレートだ。

**4. プロトコル互換層を一箇所で集中管理する**
OAuth/HTTP 仕様の非準拠実装は一箇所（例：`auth.ts`）で処理し、呼び出し箇所に散在させない。`normalizeOAuthErrorBody` はこのパターンの典型例：純粋な関数、トランスポート層で統一処理した後、呼び出し箇所はサーバーが準拠しているかどうかを気にしない。

**5. 並行接続の階層的なレート制限**
操作の種類によってリソース消費が異なる。stdio サーバーはサブプロセスの fork が必要（CPU + メモリ集約型）、ネットワークサーバーは TCP 接続の確立のみ（I/O 集約型）。二種類の操作に異なる並行制限を使用する（ローカル：3、リモート：20）ことで、システムを保護しながらスループットを最大化できる。

**6. needs-auth 状態の二重チェック**
認証が必要なリモートサービスに対して、**時間ベースのキャッシュ**（15 分 TTL）と**状態ベースのチェック**（discovery 状態はあるが token がない）の二重判断を組み合わせて、成功しないであろう接続をスキップし、起動時の無効な探索遅延を避ける。

---

## 11.11 ソースコードインデックス

| 重要な実装 | ファイル:行番号 |
|---------|---------|
| `MCPServerConnection` ユニオン型定義 | `services/mcp/types.ts:170-200` |
| `ConfigScope` 列挙（7 つのソース） | `services/mcp/types.ts:10-22` |
| `connectToServer` メイン関数（memoized） | `services/mcp/client.ts:540` |
| Transport 初期化の分岐（6 種類） | `services/mcp/client.ts:570-930` |
| 接続タイムアウトの処理 | `services/mcp/client.ts:1000-1040` |
| 切断エラーの検出と再接続のトリガー | `services/mcp/client.ts:1200-1320` |
| stdio の三段階クローズ（SIGINT/SIGTERM/SIGKILL） | `services/mcp/client.ts:1370-1490` |
| `fetchToolsForClient`（ツール登録） | `services/mcp/client.ts:1830-2050` |
| `getMcpToolsCommandsAndResources`（バッチ接続エントリー） | `services/mcp/client.ts:2580` |
| `isMcpSessionExpiredError` | `services/mcp/client.ts:150-165` |
| `wrapFetchWithTimeout`（HTTP Accept 注入） | `services/mcp/client.ts:450-510` |
| 設定ソースの統合 `getClaudeCodeMcpConfigs` | `services/mcp/config.ts:1050` |
| エンタープライズ専用モードの判断 | `services/mcp/config.ts:1080-1090` |
| Project サーバーの承認ゲート | `services/mcp/utils.ts:210-250` |
| Plugin サーバーの重複排除（署名ハッシュ） | `services/mcp/config.ts:215-270` |
| `ClaudeAuthProvider`（OAuth コア） | `services/mcp/auth.ts:500+` |
| `normalizeOAuthErrorBody`（Slack 互換） | `services/mcp/auth.ts:250-290` |
| `performMCPXaaAuth`（クロスアプリ認証） | `services/mcp/auth.ts:700+` |
| `getServerKey`（Token ストレージキー生成） | `services/mcp/auth.ts:390-405` |
| `hasMcpDiscoveryButNoToken`（高速失敗） | `services/mcp/auth.ts:420-435` |
| `classifyMcpToolForCollapse`（折りたたみ分類） | `tools/MCPTool/classifyForCollapse.ts:587-598` |
| `SEARCH_TOOLS` / `READ_TOOLS` ホワイトリスト | `tools/MCPTool/classifyForCollapse.ts:20-585` |
| `shortRequestId`（Channel 権限 ID）| `services/mcp/channelPermissions.ts:140-160` |
| `gateChannelServer`（6 層ゲート）| `services/mcp/channelNotification.ts:190-310` |
| `registerElicitationHandler` | `services/mcp/elicitationHandler.ts:65-150` |
