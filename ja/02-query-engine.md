# 第2章：Query Engine — LLMインタラクションのコア

## 2.1 概要とポジショニング

**一言でのポジショニング：** Query EngineはClaude Codeの「鼓動」——LLM会話の完全なライフサイクルを所有し、ユーザー入力を権限制御、ツール実行、コンテキスト圧縮、コスト追跡を経た多ラウンドのagentic インタラクションに変換する。

**解決するコアの問題：** Async Generator Pipelineの中で「LLMストリーミングレスポンス → ツール呼び出し → 結果注入 → 再リクエスト」というループを、コンテキストウィンドウのオーバーフロー、トークン予算、API障害、ユーザー中断、権限承認等の少なくとも12種の異常分岐を処理しながら、いかに確実に編成するか。

**関連ファイルとコード量統計：**

| ファイル | 行数 | 責務 |
|------|------|------|
| `QueryEngine.ts` | 1,295 | セッションレベル状態管理、SDK/ヘッドレスエントリー |
| `query.ts` | 1,729 | クエリメインループ、ツール実行編成、多層回復 |
| `query/config.ts` | 46 | 不変設定スナップショット |
| `query/tokenBudget.ts` | 93 | Token予算 auto-continue決定 |
| `query/stopHooks.ts` | 473 | 停止時フック（セキュリティゲート、後処理） |
| `query/deps.ts` | 40 | 依存性注入インターフェース（テスト対応） |
| `services/api/claude.ts` | 3,419 | API呼び出し、ストリーミング解析、リトライ、キャッシュ戦略 |
| `cost-tracker.ts` | 323 | コスト追跡とusage累積 |
| **合計** | **~7,418** | — |

この7,400+行のコードはClaude Codeのコードベースの約1.4%を占めるが、プロダクト全体で最も重要なパス——ユーザーとClaudeのすべてのインタラクションはここを必ず通過する。

---

## 2.2 理論的基礎

### 2.2.1 Async Generator Pipeline（コルーチンパイプライン）

Query Engineのコアアーキテクチャは**ES2018 Async Generator**をストリーミング処理の基本単位として使用する。`query()`関数のシグネチャがこの設計を示す：

```typescript
// query.ts:162
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

これは偶然の構文選択ではなく、**コルーチン理論**の精確な応用。Async generatorが同時に持つ：
- **遅延評価**：コンシューマー駆動、すべてのAPIレスポンスを事前計算しない
- **双方向通信**：stream eventをyieldし、terminal reasonをreturn
- **リソース安全**：finallyブロックがストリームの解放を保証（`claude.ts`の`releaseStreamResources()`）

従来のcallbackやPromise chainでは「UIへのストリーミング出力」と「ツール実行結果の待機」という2つの要件を同時に満たすことができない。Async generatorはこの「producerがconsumerの消費を待って一時停止する」セマンティクスをネイティブにサポートする。

### 2.2.2 State Machine（暗黙の状態機）

`query.ts`のメインループは再帰呼び出しではなく（早期のコードコメントには`query_recursive_call`という名称が残っているが）、**while(true) + continue駆動の明示的な状態機**：

```typescript
// query.ts:218 (State型定義)
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

`continue`のたびにState全体が再構築され（不変更新）、`transition`フィールドに転移理由が記録される。これは古典的な**Mealy Machine**——出力は現在の状態と入力イベントの組み合わせによって決まる。

### 2.2.3 Backpressureとリソース管理

LLMストリーミングにおけるBackpressureは極めて重要。Claude Codeのアプローチ：

1. **for-await-ofによるネイティブのレート制限**：`claude.ts`のストリーム消費は`for await (const part of stream)`を使用し、consumerの処理速度がproducerのプッシュレートを直接決定する
2. **Stream idle watchdog**（`claude.ts:2397`）：90秒以内にchunkを受信しない場合、積極的にストリームをabortしてnon-streamingにフォールバック
3. **Generatorライフサイクル保証**：`finally`ブロックが`releaseStreamResources()`をすべての終了パス（`.return()`と例外を含む）で実行することを保証

### 2.2.4 なぜこれらの理論がLLMシナリオで特に重要か

従来のHTTP API呼び出しは「リクエスト-レスポンス」モデルでエラー処理が単純。LLMのagentic loopは独自の課題に直面する：

- **単一の呼び出しが10分間継続する可能性がある**（non-streaming limit）
- **レスポンスの途中で新しいI/Oがトリガーされる可能性がある**（ツール呼び出し）
- **コンテキストウィンドウは状態を持つ希少リソース**、「圧縮による情報損失」と「オーバーフロークラッシュ」の間のトレードオフが必要
- **コストがリアルタイムに累積**、いつでも中断可能である必要がある

これらの制約により、Event Loop + 状態機 + Backpressureが不可欠な理論的サポートとなる。

---

## 2.3 アーキテクチャとデータ構造

### 2.3.1 QueryEngineクラスのコアインターフェース

```typescript
// QueryEngine.ts:155-166
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
```

**設計のポイント：** QueryEngineはper-conversation（会話ごと）。各`submitMessage()`呼び出しは同じ会話の新しいターンで、状態はターンをまたいで保持される。

### 2.3.2 重要な型定義

**QueryEngineConfig**（`QueryEngine.ts:95-153`）——構築時に渡す不変設定：

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**QueryConfig**（`query/config.ts:18-31`）——per-query不変環境スナップショット：

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

ソースコードコメントの設計意図に注目（`config.ts:9-12`）："Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." これはbun:bundleコンパイル最適化と実行時設定の明確な境界線。

### 2.3.3 モジュール間の依存関係図

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- セッション状態管理
                    |  (1,295 lines)  |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- メインループ & 状態機
                    |  (1,729 lines)  |  <-- State, while(true) loop
                    +--------+--------+
                             |
              +--------------+------------------+
              |              |                  |
              v              v                  v
    +---------+---+  +-------+--------+  +------+-------+
    | query/      |  | query/         |  | query/       |
    | config.ts   |  | tokenBudget.ts |  | stopHooks.ts |
    | (46 lines)  |  | (93 lines)     |  | (473 lines)  |
    +-------------+  +----------------+  +--------------+
              |              |
              v              v
    +---------+--------------+--------+
    |         query/deps.ts           |
    |   QueryDeps (DIインターフェース) |
    +---------+-----------------------+
              |
              |  callModel / autocompact / microcompact
              v
    +---------+-----------+     +----------+--------+
    | services/api/       |     | cost-tracker.ts   |
    | claude.ts           |     | (323 lines)       |
    | (3,419 lines)       |     | addToTotalSession |
    | queryModelWith      |     | Cost, formatCost  |
    | Streaming           |     +-------------------+
    +---------+-----------+
              |
              v
    +---------+-----------+
    | withRetry / stream  |
    | SSE parsing         |
    | Non-streaming       |
    | fallback            |
    +---------------------+
```

**deps.tsの依存性注入設計**（`deps.ts:18-37`）は特に注目に値する：

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" —— これはTypeScriptの依存性注入のベストプラクティス：インターフェースを手書きする必要がなく、シグネチャが自動的に実装を追跡する。

---

## 2.4 コアアルゴリズムとフロー

### 2.4.1 query()メインループの完全実行フロー

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    query() エントリー
                        |
                +=======+========+  <--- while(true) メインループ開始
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        blocking-limit チェック   |
                |                |
                v                |
        callModel() [ストリーミング]|
                |                |
        +-------+--------+      |
        | ストリームイベント消費 |      |
        | tool_use 収集    |      |
        | streaming exec  |      |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           YES           |
       |              \          |
       v               v        |
  +---------+   runTools() or  |
  | 回復ロジック|   getRemainingR() |
  +---------+         |         |
       |              v         |
       |     getAttachments()   |
       |     memory prefetch    |
       |     skill discovery    |
       |              |         |
       v              v         |
  handleStopHooks()   |         |
       |         maxTurns?      |
       |              |         |
       v         state = next   |
  checkTokenBudget()  |         |
       |              +---------+
       v
  return Terminal { reason }
```

### 2.4.2 ツール呼び出しループ（Tool-call Loop）

ツール呼び出しのコアイノベーションは**streaming tool execution**——LLMのストリーミングレスポンスが*まだ進行中の時*にツールの実行を開始する：

```typescript
// query.ts:443 (StreamingToolExecutor初期化)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

ストリーミング消費ループ内（`query.ts:536`）：

```typescript
if (
  streamingToolExecutor &&
  !toolUseContext.abortController.signal.aborted
) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

ツールは`content_block_stop`時に実行提出され、assistantレスポンス全体の終了を待たない。つまり、LLMが3つのtool_use blockを出力した場合、最初の2つは3番目がまだストリーミング中に実行完了している可能性がある。

### 2.4.3 ストリーミング処理の具体的実装

`claude.ts`の`queryModel()`はSSEストリーム解析を手動実装し、**意図的にAnthropicのSDKのBetaMessageStreamを回避している**：

```typescript
// claude.ts コメント (約 line 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

ストリーミング状態累積モデル：

```
message_start → partialMessage = part.message, usage = initial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → AssistantMessageをyield（ブロックごと！）
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (終了)
```

重要な設計：**各content blockが独立してAssistantMessageをyield**。これにより、LLMのレスポンスにtext + tool_useが含まれる場合、UIはtextが完了した後すぐに表示でき、tool_useのJSON完了を待たなくて済む。

### 2.4.4 リトライとエラー回復の5層メカニズム

Claude Codeのエラー回復アーキテクチャはDefense in Depth（縦深防御）で、合計5層：

**第1層：withRetry（APIレベル）** —— `claude.ts`の`withRetry()`が429（レート制限）、529（過負荷）、5xx等のリトライ可能なエラーを処理、指数バックオフとモデルフォールバックを含む。

**第2層：Streaming → Non-streaming Fallback** —— ストリーミング接続が中断した場合（`claude.ts:2592`）：

```typescript
// Non-streamingモードでリトライにフォールバック
const result = yield* executeNonStreamingRequest(...)
```

stream idle watchdog（90秒無データタイムアウト）と404ストリーム作成フォールバックも含む。

**第3層：max_output_tokens回復** —— `query.ts`の3段階の段階的回復：

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// 第一歩：64kトークンにエスカレート（ESCALATED_MAX_TOKENS）
// 第二〜四歩：「直接再開——謝罪なし、要約なし」を求めるmetaメッセージを注入
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**第4層：Prompt-too-long回復** —— 三段階連動：
1. Context collapse drain（蓄積された折りたたみを排出）
2. Reactive compact（緊急フル圧縮）
3. エラーを表面化して終了（無限ループを防ぐため）

ソースコードには無限ループを防ぐ明示的なガード（`query.ts`コメント）がある：

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**第5層：Model Fallback** —— `query.ts:673-719`で、`FallbackTriggeredError`がキャッチされた際に、フォールバックモデルに切り替えてリクエスト全体をリトライ。

### 2.4.5 トークンカウントと予算管理

Token予算システムは2つの独立したメカニズムに分かれる：

**メカニズムA：API task_budget** —— サーバーサイドで認識するtoken予算、compaction境界をまたいで追跡：

```typescript
// query.ts:270-280 (taskBudgetRemainingのcompaction追跡)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**メカニズムB：クライアントサイドのtoken budget auto-continue**（`tokenBudget.ts`）——ターン出力が予算の90%に達しない場合に自動的に続きを書く：

```typescript
// tokenBudget.ts:46-62
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // サブエージェントはauto-continueを実行しない
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }
  // ...
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
  // ...
}
```

**diminishing returns検出**に注目：連続3回の継続且つ毎回の増分が500トークン未満の場合に自動停止し、モデルが非効率な出力に予算を浪費するのを防ぐ。

### 2.4.6 Thinking Mode処理

ソースコードの「wizard comment」がThinking modeの3つの鉄則を要約（`query.ts:105-118`）：

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

`claude.ts`のthinkingパラメータ構築ロジック（約line 2242）：

```typescript
if (
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING) &&
  modelSupportsAdaptiveThinking(options.model)
) {
  thinking = {
    type: 'adaptive',
  } satisfies BetaMessageStreamParams['thinking']
} else {
  let thinkingBudget = getMaxThinkingTokensForModel(options.model)
  // ...
  thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
  thinking = {
    budget_tokens: thinkingBudget,
    type: 'enabled',
  }
}
```

Adaptive thinkingをサポートするモデルは優先的にadaptive（予算の事前設定不要）を使用し、そうでなければ enabled + budget_tokensにフォールバック。

### 2.4.7 Prompt Cache境界管理

Claude Codeのキャッシュ戦略は印象的。コア設計は`addCacheBreakpoints()`（`claude.ts:3045`）：

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**cache_controlマーカーを1つだけ配置**し、最後のメッセージに置く（`skipCacheWrite`の場合は最後から2番目に）。これはinferenceチームのKVキャッシュページマネージャー（Mycro）と協調して最適化した結果。

1時間TTLキャッシュにはさらに精細な「session stability latching」メカニズム（`claude.ts:380-420`）がある——eligibilityが確定したらセッション全体で固定し、途中でGrowthBook設定が変化してcache_control TTLが反転してキャッシュが壊れるのを防ぐ。

---

## 2.5 設計上の意思決定分析

### 2.5.1 重要なトレードオフ

**トレードオフ1：ストリーミング実行 vs. 完全性保証**

StreamingToolExecutorはLLMがまだ出力中の時にツールの実行を開始し、顕著なレイテンシ最適化をもたらすが、複雑性も導入する——ストリーミングが途中でフォールバックした場合、実行済みのツールを破棄する必要がある：

```typescript
// query.ts:534-538 (streamingフォールバック時のクリーンアップ)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

この問題はすでにバグを引き起こしている（`claude.ts:2575`コメントでinc-4258を参照：double tool execution）。

**トレードオフ2：キャッシュ安定性 vs. 動的機能**

複数のbetaヘッダーは「sticky-on latch」パターンを使用（`claude.ts:2102-2126`）——一度アクティベートされたらセッション全体を通じて保持され、機能がオフになっても：

```typescript
// claude.ts:2104 コメント
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

これは「キャッシュヒット率を機能の柔軟性より優先」という明確なトレードオフ。

**トレードオフ3：状態機 vs. 再帰**

メインループは初期の再帰的な`query()`呼び出しから`while(true)` + State再構築へと進化した。ソースコードのコメントには`query_recursive_call`というcheckpoint名が残っているが、実際にはイテレーティブになっている。利点：
- スタックオーバーフローのリスクなし（長い会話では数百ターンになる可能性がある）
- Stateの再構築が明示的で、デバッグが容易
- `transition`フィールドが状態遷移の完全な監査証跡を提供

### 2.5.2 ソースコードコメントが露わにする既知の問題

1. **SDK text deltaの重複**（`claude.ts:2350`）：

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Non-streaming fallbackとstreaming tool executionの競合**（`claude.ts:2575`）：

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Task budgetのcompaction跨ぎでのtoken計数の乖離**（`query.ts:268`）：

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 LangChain等のアプローチとの設計上の違い

| 次元 | Claude Code Query Engine | LangChain AgentExecutor |
|------|--------------------------|-------------------------|
| ストリーミング基本単位 | ES Async Generator（ネイティブ） | Callback + Stream wrapper |
| 状態管理 | 明示的なState struct + 不変更新 | Mutable AgentState dict |
| ツール実行 | ストリーミング並列可能（StreamingToolExecutor） | 直列のawait |
| リトライ | 5層縦深防御 + model fallback | 単純なmax_iterations |
| 依存性注入 | QueryDeps + typeof シグネチャ同期 | 実行時のduck typing |
| キャッシュ | inferenceのKVキャッシュと深く連携 | なし（ブラックボックスAPI呼び出し） |

最も根本的な違い：Claude Codeは**inference-aware**——prompt cacheの物理メカニズム（Mycroページマネージャー）を理解し、それに基づいて最適化するが、オープンソースフレームワークはAPIをブラックボックスとしてしか扱えない。

---

## 2.6 移植可能なパターン

### 2.6.1 Query Engineから抽出した汎用エンジニアリングパターン

**パターン1：Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

各状態遷移の**理由**をstateに記録し、デバッグとテレメトリーをfirst-class concernにする。多段階の意思決定が必要なシステムに適用可能。

**パターン2：`typeof`による型付き依存性注入**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

インターフェースを手書きする必要がなく、シグネチャが実装と自動同期。重いI/Oをモックする必要があるシステムに適用可能。

**パターン3：Withholding Pattern（エラー暴露の遅延）**

回復可能なエラー（prompt-too-long、max_output_tokens）は、回復ロジックが実行された後に暴露するかどうかを決定するまで、コンシューマーにyieldしない：

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

これにより「エラーがすでに回復された」状況でSDKのコンシューマーが偽のエラー信号を受け取るのを防ぐ。

**パターン4：Session-stable Latching**

キャッシュキーに影響する設定項目は、一度アクティベートされたらセッション全体でロック：

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // bootstrap stateに書き込み
}
```

「設定の変化がコストのかかるリソースの無効化を引き起こす可能性がある」すべてのシナリオに適用可能。

### 2.6.2 Doramagicが参考にできること

Doramagicの`flow_controller`パイプラインはQuery Engineと同様の編成問題に直面しており、参考にできる：

1. **State + Transitionパターン**：Doramagicの12状態機は同様の`{ state, transition: { reason } }`構造を使用でき、デバッグを簡略化しつつログ監査をネイティブにサポート
2. **typeofによる依存性注入**：DoramagicのLLM呼び出し層はQueryDepsパターンを使用でき、テスト時にモジュール全体をモックせずにfakeモデルを注入
3. **diminishing returns検出**（`tokenBudget.ts`）：DoramagicのSoul Extractorがiterative refinementを行う際に、同様の「連続N回の増分が閾値を下回ったら停止」戦略を使用し、LLMが低品質な出力にtokenを浪費するのを防ぐ

---

## 2.7 ソースコードインデックス

| ファイル | 行数 | 一言での責務 |
|------|------|----------|
| `QueryEngine.ts` | 1,295 | セッションレベルのowner：mutableMessages、totalUsageを保持し、submitMessage()をquery()呼び出しに変換し、SDKメッセージルーティングを処理 |
| `query.ts` | 1,729 | メインループ状態機：while(true)でcompaction → API call → tool execution → stop hooks → budget checkを編成 |
| `query/config.ts` | 46 | 不変のQueryConfigスナップショット：sessionId + 4つのruntime gates（feature() gatesはtree-shakingのために意図的に除外） |
| `query/tokenBudget.ts` | 93 | クライアントサイドのtoken budget auto-continue：90%完成度閾値 + diminishing returns早期停止 |
| `query/stopHooks.ts` | 473 | ターン終了フック編成：Stop hooks → TaskCompleted hooks → TeammateIdle hooks、blocking errorの再注入をサポート |
| `query/deps.ts` | 40 | 4つのI/O依存性注入インターフェース：callModel、microcompact、autocompact、uuid |
| `services/api/claude.ts` | 3,419 | APIの全ライフサイクル：パラメータ構築 → stream作成 → SSE解析 → content block累積 → cost計算 → non-streaming fallback → cache breakpoint管理 |
| `cost-tracker.ts` | 323 | セッションレベルのコスト累積：モデル別のusage追跡、セッションの永続化/復元、フォーマット出力 |
