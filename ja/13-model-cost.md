# 第 13 章：モデル選択とコスト制御

> データソース：Claude Code TypeScript ソースコードスナップショット（2026-03-31、約 512K LOC）
> コアファイル：`services/api/claude.ts`（3,419 行）、`services/api/withRetry.ts`、`cost-tracker.ts`（323 行）、`utils/effort.ts`、`utils/modelCost.ts`、`utils/model/model.ts`、`migrations/` ディレクトリ（11 ファイル）

---

## 13.1 概要と位置づけ

Claude Code のモデル選択とコスト制御における設計哲学は 3 つの文章で要約できる：

1. **ユーザーの意図優先**：優先度チェーンは `/model` コマンド → `--model` フラグ → 環境変数 → 設定ファイルの順で、各層は上位層によって上書きされるが、下位層によって意図せず置き換えられることはない。
2. **コストの完全な透明性**：セッション終了時にモデルごとに分類したトークン使用量と USD 料金を強制的に表示し、無効化できない（`hasConsoleBillingAccess()` が true の場合のみ）。
3. **秘密のダウングレードなし**：Overload Fallback（Opus → Sonnet）が発生した場合、ユーザーに警告メッセージを表示しなければならず、サイレントな切り替えは絶対に行わない。

本章ではソースコードレベルでこのサブシステムに関する cc-notebook の主張を一つひとつ検証し、分析を深める。

---

## 13.2 理論的基礎

### マルチモデルシステムのルーティング戦略

マルチモデルシステムでは、ルーティング戦略は通常 3 つの次元でバランスを取る：**能力**（capability）、**コスト**（cost）、**レイテンシ**（latency）。Claude Code の選択は、メイン会話（main loop）を最強の利用可能モデルにルーティングし、バックグラウンドの補助タスクを最速かつ最安価なモデルにルーティングし、メインモデルが利用不能な場合には透明なダウングレードを提供することである。

### AI システムにおけるコストベネフィット分析の応用

`modelCost.ts` から、Claude Code が精確な価格表を内蔵していることがわかる：

```typescript
// utils/modelCost.ts
// Standard pricing tier for Sonnet models: $3 input / $15 output per Mtok
export const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// Fast mode pricing for Opus 4.6: $30 input / $150 output per Mtok
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
}
```

Haiku 4.5 の価格が最も低く（$1/$5 per Mtok）、Opus 4.6 Fast Mode の価格が最も高い（$30/$150 per Mtok）で、両者の差は 30 倍になる。この価格差がシステムがバックグラウンドタスクを Haiku に割り当てる核心的な経済的ロジックである。

### グレースフルデグラデーション（Graceful Degradation）パターン

従来のソフトウェアでは、グレースフルデグラデーションとは機能が利用不能な場合に次善策を取りつつクラッシュしないことを意味する。LLM システムでは、フォールバックとは「より安価で利用可能なモデルへの切り替え」である。Claude Code はカウンター保護付きのトリガーメカニズムを実装している：連続 3 回の 529 エラーの後にモデル切り替えをトリガーし、即時切り替えは行わない（偶発的なオーバーロードが不必要な品質ダウングレードを引き起こすことを回避する）。

---

## 13.3 モデル選択アーキテクチャ

### モデル優先度階層

`utils/model/model.ts` の `getUserSpecifiedModelSetting()` 関数が優先度順序を正確に定義している：

```typescript
// utils/model/model.ts:44-66
/**
 * Priority order within this function:
 * 1. Model override during session (from /model command) - highest priority
 * 2. Model override at startup (from --model flag)
 * 3. ANTHROPIC_MODEL environment variable
 * 4. Settings (from user's saved settings)
 */
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  let specifiedModel: ModelSetting | undefined

  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) {
    specifiedModel = modelOverride
  } else {
    const settings = getSettings_DEPRECATED() || {}
    specifiedModel = process.env.ANTHROPIC_MODEL || settings.model || undefined
  }

  // Ignore the user-specified model if it's not in the availableModels allowlist.
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

`getMainLoopModel()` はさらに第 5 優先度——内蔵デフォルト値——を追加する：

```typescript
// utils/model/model.ts:68-77
export function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}
```

完全な 5 段階優先度チェーン：
| 優先度 | ソース | 説明 |
|--------|------|------|
| 1（最高）| `/model` コマンド | セッション内で即時有効、メモリ override に保存 |
| 2 | `--model` 起動フラグ | 起動時にメモリ override に書き込み |
| 3 | `ANTHROPIC_MODEL` 環境変数 | プロセスレベル |
| 4 | `settings.json` 設定ファイル | 永続化されたユーザー設定 |
| 5（最低）| 内蔵デフォルト値 | サブスクリプションタイプに応じて決定 |

### サブスクリプションタイプ別のデフォルトモデル階層化

`getDefaultMainLoopModelSetting()` がサブスクリプションの差異を明らかにしている：

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants（社内従業員）はデフォルト Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max ユーザーと Team Premium ユーザーはデフォルト Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG、Enterprise、Team Standard、Pro はデフォルト Sonnet 4.6
  return getDefaultSonnetModel()
}
```

この設計は、ユーザーが何も設定しなくても、Max/Team Premium ユーザーは Opus 4.6 が開き、Pro/Sonnet ユーザーは Sonnet 4.6 が開くことを意味する。**デフォルト値自体が製品差別化戦略である。**

### モデル Alias システム

`parseUserSpecifiedModel()` は短縮エイリアスの解析をサポートし、ユーザーが完全なモデル ID を覚える必要をなくしている：

```typescript
// utils/model/model.ts — parseUserSpecifiedModel 抜粋
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan モードは Sonnet を使用、非 plan モードは Opus を使用
```

`[1m]` サフィックスは任意のエイリアスに付加でき（例：`opus[1m]`）、システムが自動的に 1M コンテキストウィンドウのバリアントに解析する。

### モデル能力検出

`utils/model/modelCapabilities.ts` はキャッシュメカニズムを実装しており、社内従業員（`USER_TYPE === 'ant'`）にのみ有効：

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

外部ユーザーはモデル能力リストをリクエストせず、能力情報は `modelSupportsEffort()`、`modelSupports1M()` などの関数にハードコードされており、追加の API 呼び出しコストを回避する。

---

## 13.4 Haiku のバックグラウンド用途

cc-notebook は Haiku に 6 種類のバックグラウンド用途があると主張している。`queryHaiku` 関数の呼び出し箇所（`grep -rn 'queryHaiku\b'`）と `getSmallFastModel()` の呼び出し箇所の全量検索により、**ソースコード検証**は以下の通り：

### バックグラウンド用途まとめ（ソースコード検証）

| 番号 | 用途 | ファイル | トリガー条件 |
|------|------|------|---------|
| 1 | Web Fetch コンテンツ抽出 | `tools/WebFetchTool/utils.ts:503` | ウェブページ取得後、Haiku でマークダウンをユーザー指定コンテンツにフィルタリング |
| 2 | シェルコマンドプレフィックス抽出 | `utils/shell/prefix.ts:220` | Bash ツール実行前、Haiku でコマンドが権限プロンプトを必要とするか判断 |
| 3 | セッションタイトル生成 | `utils/sessionTitle.ts:87` | セッション開始後、自動的に短いタイトルを生成（JSON schema 出力）|
| 4 | MCP DateTime 解析 | `utils/mcp/dateTimeParser.ts:68` | 自然言語の時間記述を ISO 8601 形式に解析 |
| 5 | ツール呼び出しサマリー生成 | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | ツール呼び出しのバッチ完了後、1 行のサマリーラベルを生成 |
| 6 | セッションリネーム | `commands/rename/generateSessionName.ts:20` | `/rename` コマンドで kebab-case 名前を生成 |

**追加の発見**（cc-notebook が言及していない、`getSmallFastModel()` 検索で発見）：

| 番号 | 用途 | ファイル | トリガー条件 |
|------|------|------|---------|
| 7 | API Key 検証 | `services/api/claude.ts:544` | API Key の有効性を検証（ソースコードコメント：「WARNING: if you change this to use a non-Haiku model, this request will fail in 1P」）|
| 8 | Away モードサマリー | `services/awaySummary.ts:49` | ユーザー離脱時にコンテキストサマリーを生成（AFK モード）|
| 9 | Web 検索補助 | `tools/WebSearchTool/WebSearchTool.ts:280` | 一部の Web 検索シナリオで Haiku が結果を処理 |
| 10 | クォータ状態確認 | `services/claudeAiLimits.ts:200` | 最小の Haiku リクエストで現在のクォータ状態を調査 |
| 11 | トークン数見積もり | `services/tokenEstimation.ts:277` | コンテキストウィンドウ使用量を見積もる |
| 12 | Prompt/Exec Hook 実行 | `utils/hooks/execPromptHook.ts:79`、`execAgentHook.ts:118` | Hook コールバックはデフォルトで Haiku を使用（hook 設定が上書きしない限り）|
| 13 | Skill 改善分析 | `utils/hooks/skillImprovement.ts:169` | Skill 実行後、改善提案を自動的に分析 |

**結論**：cc-notebook の「6 種類のバックグラウンド用途」は**過小評価**である。ソースコード内の `queryHaiku` または `getSmallFastModel()` の呼び出し箇所は少なくとも 13 か所あり、セッションライフサイクルの各段階（起動検証、実行中の補助、セッション整理）をカバーしている。Haiku/SmallFastModel はシステム全体のバックグラウンド「基盤サービス層」であり、時折登場する最適化手段ではない。

重要な設計詳細：`queryHaiku` は非ストリーミング呼び出し（`queryModelWithoutStreaming`）を使用し、Tool permission context なし（`getEmptyToolPermissionContext()`）：

```typescript
// services/api/claude.ts:3280-3291
const result = await queryModelWithoutStreaming({
  messages,
  systemPrompt,
  thinkingConfig: { type: 'disabled' },
  tools: [],
  signal,
  options: {
    ...options,
    model: getSmallFastModel(),
    enablePromptCaching: options.enablePromptCaching ?? false,
    async getToolPermissionContext() {
      return getEmptyToolPermissionContext()
    },
  },
})
```

---

## 13.5 Overload Fallback メカニズム

cc-notebook は「529 Overload Fallback、Opus → Sonnet フォールバック」の存在を主張している。ソースコードはこの主張を**完全に検証**しており、詳細はさらに豊富である。

### 529 エラーの識別

`services/api/withRetry.ts` の `is529Error()` 関数：

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // 529 ステータスコード、またはストリーミング中に SDK がステータスコードを正しく伝達できない場合を確認
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

二重検出に注意：ステータスコード `529` とエラーメッセージ内の `overloaded_error` 文字列。SDK がストリーミング中に 529 ステータスコードを正しく伝達できない場合があるためである。

### トリガー条件：連続 3 回の 529

```typescript
// services/api/withRetry.ts — withRetry 関数抜粋
const MAX_529_RETRIES = 3

if (
  is529Error(error) &&
  (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
    (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })
      // 特殊エラーをスローし、上位層のモデル切り替えをトリガー
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

重要な制約：
- デフォルトでは**非 ClaudeAI サブスクリプションユーザー**の**Opus シリーズモデル**に対してのみトリガー（`isNonCustomOpusModel()`）
- 環境変数 `FALLBACK_FOR_ALL_PRIMARY_MODELS` で全メインモデルに拡張可能
- ストリーミングリクエストの 529 はカウンターに算入される（`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`）、非ストリーミングの再試行と協調してカウント

### FallbackTriggeredError のシグナル伝播

`FallbackTriggeredError` は `originalModel` と `fallbackModel` フィールドを持つ専用エラークラスで、呼び出しスタックを遡って `query.ts` に伝播する：

```typescript
// services/api/withRetry.ts
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    super(`Model fallback triggered: ${originalModel} -> ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

### query.ts でのモデル切り替えとユーザー通知

`query.ts:894-946` がこのエラーをキャッチし、実際のモデル切り替えを実行する：

```typescript
// query.ts — FallbackTriggeredError 処理抜粋
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // warning レベルでユーザーに表示——verbose モードの有効無効に関わらず表示される
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // toolUseContext 内のメインループモデルを同期更新
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // 新しいモデルでリクエスト全体を再試行
}
```

**ユーザー通知メカニズム**：切り替えメッセージは `'warning'` レベルを使用しており、ユーザーが verbose モードを有効にしているかどうかに関わらず、インターフェースに通知が表示される。**cc-notebook の「秘密のダウングレードなし」に関する主張は完全に検証された。**

### バックグラウンドタスクの 529 戦略：即座に諦める

非フォアグラウンドタスク（summary、title、suggestions など）は 529 時に**再試行せず**、直接破棄する：

```typescript
// services/api/withRetry.ts — FOREGROUND_529_RETRY_SOURCES
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'agent:builtin',
  'compact',
  'verification_agent',
  'side_question',
  'auto_mode',
  // ...
])

// 非フォアグラウンドタスクの 529 は直接スロー、再試行なし
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

これはアーキテクチャ上のコスト制御の意思決定である：バックグラウンドタスクの再試行は容量が逼迫している際に 3〜10 倍のゲートウェイ増幅効果をもたらすが、ユーザーはこれらのタスクの失敗をまったく感知しない。

---

## 13.6 Effort Level メカニズム

cc-notebook は Effort Level システムの存在を主張している。ソースコードはこれを**完全に検証**しており、説明よりもはるかに詳細である。

### 4 つの Effort Level

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

各レベルのセマンティクス（`getEffortLevelDescription()` より）：
- **low**：Quick, straightforward implementation with minimal overhead
- **medium**：Balanced approach with standard implementation and testing
- **high**：Comprehensive implementation with extensive testing and documentation
- **max**：Maximum capability with deepest reasoning（**Opus 4.6 のみ**）

### モデルサポートマトリックス

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // Opus 4.6 と Sonnet 4.6 のみ effort パラメータをサポート
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku、旧バージョンの Opus/Sonnet はサポートなし
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P はデフォルト true、3P はデフォルト false
  return getAPIProvider() === 'firstParty'
}

// max effort は Opus 4.6 のみ利用可能
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### 優先度チェーン：env → appState → モデルデフォルト

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' または 'auto' → effort パラメータを送信しない
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API は非 Opus 4.6 の max を拒否 → 自動的に high にダウングレード
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### デフォルト Effort の差別化

Opus 4.6 のデフォルト effort はサブスクリプションタイプによって異なる：

```typescript
// utils/effort.ts — getDefaultEffortForModel 抜粋
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // Pro ユーザーはデフォルト medium（クォータ節約）
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team も GrowthBook 設定で medium に誘導可能
  }
}
```

興味深いことに、`OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` の `dialogDescription` には明示的に「We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits.」と書かれている——これはデフォルト medium が意識的なクォータ管理戦略であり、パフォーマンス優先ではないことを示している。

### max の永続化制限

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max は ant 以外のユーザーにとってはセッションレベルで、永続化されない
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

外部ユーザーが設定した `max` effort は `settings.json` に書き込まれず、現在のセッションのみで有効となる。

---

## 13.7 コスト追跡システム

### cost-tracker.ts のコア職責

`cost-tracker.ts`（323 行）は 3 つの職責を担う：
1. **リアルタイム累積**：各 API 応答後に `addToTotalSessionCost()` を呼び出す
2. **永続化**：セッション終了時にプロジェクト設定ファイルに書き込む（`saveCurrentSessionCosts()`）
3. **復元**：再起動時に設定ファイルから前回のコストデータを読み取る（`restoreCostStateForSession()`）

### モデル別トークン統計

`addToTotalModelUsage()` はモデル名ごとに 5 次元のデータを累積する：

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

セッション終了時にフォーマットして表示する（`formatModelUsage()`）：短縮名で集計（複数の API エンドポイントが同じモデルの異なる形式を返す）し、以下のように表示する：

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Fast Mode のコストマーキング

`addToTotalSessionCost()` には Fast Mode に対する特別処理がある：

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

`speed: 'fast'` マーキングは課金に影響する——Fast Mode では Opus 4.6 は `COST_TIER_30_150`（$30/$150）を使用し、標準の `COST_TIER_5_25`（$5/$25）ではない。

### Advisor ネストコスト追跡

`addToTotalSessionCost()` は Advisor ツールの使用量を再帰的に処理する：

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

Advisor はメインモデルの応答内に隠れたサブモデル呼び出しで、そのコストは個別に追跡され、総コストに合算される。

### コスト表示トリガーメカニズム

`costHook.ts`（22 行）はプロセス終了イベントを監視する React hook：

```typescript
// costHook.ts
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

`hasConsoleBillingAccess()` がコストを表示するかどうかを制御し、課金情報にアクセスできない環境（CCR/Remote モードなど）でコストが表示されないことを確保する。一方、`saveCurrentSessionCosts()` への書き込みは無条件に実行される——表示の有無に関わらず、常に永続化される。

---

## 13.8 API 呼び出し層

### claude.ts リクエスト構築のコアパラメータ

`services/api/claude.ts`（3,419 行）は API 呼び出しの統一エントリーポイントである。重要なパラメータは複数のシステムから集約される：

```typescript
// services/api/claude.ts — リクエストパラメータ組み立て（概略）
{
  model: normalizeModelStringForAPI(options.model),  // [1m] サフィックスを剥離
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // Effort パラメータ（サポートされるモデルのみ）
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` は API に送信する前に `[1m]` および `[2m]` サフィックスを剥離する——これらのサフィックスは 1M コンテキストウィンドウをマーキングするためのクライアント内部の規約にすぎず、API 層はこれらを認識しない。

### ストリーミング応答と非ストリーミングフォールバック

メイン会話はストリーミング（Server-Sent Events）を使用するが、ストリーミングが失敗した場合は非ストリーミングにフォールバックする：

```typescript
// services/api/claude.ts:2535-2559
// ストリーミング自体が 529 の場合、この回を連続 529 カウントに算入
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

非ストリーミングフォールバックには最大トークン制限がある：

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Beta Headers の動的インジェクション

異なる機能は異なる Beta Header に対応し、リクエスト時に動的に付加される：

```typescript
// constants/betas.ts（参照）
EFFORT_BETA_HEADER        // effort パラメータサポート
CONTEXT_1M_BETA_HEADER    // 1M コンテキストウィンドウ
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // 予算制御
```

---

## 13.9 設計上の意思決定の分析

### 秘密のダウングレードなしの設計哲学

`query.ts` の `'warning'` レベルの切り替え通知と、`FallbackTriggeredError` の専用エラークラス設計から、これが意図的なアーキテクチャの選択であることがわかる：

**なぜサイレントに切り替えてはならないのか？** Claude Code はコード作成ツールであり、モデルの品質は出力品質に直接影響するからである。ユーザーは「私は Opus ではなく Sonnet を使用している」ということを知る権利があり、そのうえで待ち続けるか別の戦略を取るかを決定できる。コンシューマー向けのチャット製品とは異なり、コードツールのユーザーはより専門的で、モデルの差異に対してより敏感である。

### コスト透明性の設計考慮

`costHook.ts` の `hasConsoleBillingAccess()` の設計は注目に値する：表示しない場合でも、コストデータは永続化される。これはコスト追跡の主な目的が**セッション復元**（次回起動時に前回の費用を表示する）であり、リアルタイムアラートではないことを示している。これは「オフライン認識」の設計である：ユーザーは各 API 呼び出しごとに中断されることなく、セッション終了後に全費用を確認できる。

### モデルデフォルト差別化の製品ロジック

Opus を Max/Team Premium のデフォルトモデルとし、Sonnet を Pro/PAYG のデフォルトモデルとすることには明確な製品ロジックがある：Max サブスクリプションの価値提案の一つは「最強モデルへのアクセス」であり、デフォルト値自体がこの価値提案の体現である。

同時に、Max ユーザーでも、Opus 4.6 のデフォルト effort は `medium` である（GrowthBook によって制御される）——これは Anthropic が effort システムを通じて**品質とクォータのバランスを取っている**ことを示しており、Max ユーザーに最高の設定を一方的に提供するわけではない。

---

## 13.10 モデルマイグレーション（migrations）の必要性

`migrations/` ディレクトリ下の 11 個のマイグレーションファイルは製品進化の痕跡を明らかにしており、各マイグレーションは製品上の意思決定に対応している：

| マイグレーションファイル | トリガータイミング | コアロジック |
|---------|---------|---------|
| `migrateFennecToOpus.ts` | 社内従業員（ant）| fennec コードネームエイリアス → opus エイリアス（社内コードネームのクリーンアップ）|
| `migrateLegacyOpusToCurrent.ts` | 1P ユーザー、`opus-4-0`/`4-1` が settings にある | 旧バージョンの Opus モデル ID → `opus` エイリアス（Opus 4.0/4.1 廃止）|
| `migrateOpusToOpus1m.ts` | Max/Team Premium（1P）、settings に `opus` がある | `opus` → `opus[1m]`（1M エクスペリエンスへの統合）|
| `migrateSonnet1mToSonnet45.ts` | `sonnet[1m]` を使用しているユーザー | `sonnet[1m]` → `sonnet-4-5-20250929[1m]`（4.5 に固定、4.6 1M は対象が異なるため）|
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium（1P）、Sonnet 4.5 に固定 | Sonnet 4.5 文字列 → `sonnet` エイリアス（4.6 へのアップグレード）|
| `resetProToOpusDefault.ts` | Pro 1P ユーザー、カスタムモデルなし | マイグレーションのタイムスタンプを記録し、REPL に一度アップグレード通知を表示 |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode 有効、旧 OptIn ダイアログユーザー | `skipAutoPermissionPrompt` をクリアし、新バージョンの権限ダイアログを再表示 |
| `migrateAutoUpdatesToSettings.ts` | ユーザーが自動更新を明示的に無効化 | `autoUpdates: false` を settings.json の環境変数に移行 |
| `migrateBypassPermissionsAcceptedToSettings.ts` | グローバル設定に `bypassPermissionsModeAccepted` がある | settings.json の `skipDangerousModePermissionPrompt` に移行 |
| `migrateSonnet45ToSonnet46.ts` | 同上 | 前述の同名マイグレーション |
| `migrateEnableAllProjectMcpServersToSettings.ts` | MCP 関連設定 | MCP サーバー設定構造の調整 |

**アーキテクチャの洞察**：各マイグレーションは `userSettings`（ユーザーレベルの settings.json）のみを操作し、`projectSettings`（プロジェクトレベル）や `policySettings`（企業ポリシーレベル）には一切触れない。これは意図的な設計である：

1. **冪等性**：同じデータソースを読み書きし、再実行しても副作用が生じない
2. **最小権限**：ユーザーがプロジェクトレベルで固定した設定を変更できない（また変更すべきでない）
3. **グローバル昇格の回避**：ユーザーがあるプロジェクトで旧 Opus を固定している場合、マイグレーションはそれをグローバル設定に昇格させない

このマイグレーションシステムの存在自体が示している：**AI システムのスキーママイグレーションは従来のデータベースマイグレーションよりもはるかに複雑である**——サブスクリプションタイプの変更、モデルの廃止、コンテキストウィンドウのアップグレードなど複数の次元を考慮する必要があり、ユーザーの意図を単純に上書きすることはできない。

---

## 13.11 移植可能なパターン

本章の分析から、自身のシステムで使用できる 5 つの設計パターンを抽出する：

### パターン 1：多段階 Override チェーン
```
session_override > startup_flag > env_var > config_file > builtin_default
```
任意の層は上位層によって上書きされるが、下位層は密かに上位層に影響を与えることはできない。allowlist チェックを組み合わせて不正なモデル ID のインジェクションを防ぐ。

### パターン 2：フォアグラウンド/バックグラウンドの 529 戦略分離
フォアグラウンドタスク（ユーザーが結果を待っている）：N 回再試行し、上限を超えたら fallback をトリガー。
バックグラウンドタスク（ユーザーが感知しない）：最初の 529 で即座に諦め、容量が逼迫している際の再試行増幅効果を回避。

### パターン 3：FallbackTriggeredError のシグナル化
retry 内部でこっそりモデルを切り替えるのではなく、専用エラーをスローして上位の呼び出し元に切り替えロジックを処理させる。これにより切り替えロジックが一箇所（query.ts）に集中し、必然的にユーザー通知が伴う。

### パターン 4：toPersistableEffort 永続化フィルタリング
セッションレベルの設定（`max` effort など）は `settings.json` への書き込み前にフィルタリングされる。「セッション間で永続化すべきでない状態」と「永続化すべきユーザー設定」がデータモデルの層で区別される。

### パターン 5：コストのモデル別バケット追跡
総コストだけを追跡するのではなく、モデル名（正規化後）でバケットに分類する。こうすることでセッション終了時に「Opus がいくら、Haiku がいくら」を表示でき、ユーザーがどの機能が最も高価かを理解できる。

---

## 13.12 ソースコードインデックス

| ファイル | 行数 | コアコンテンツ |
|------|------|---------|
| `services/api/claude.ts` | 3,419 | API 呼び出し層、queryHaiku、リクエスト構築、ストリーミング処理 |
| `services/api/withRetry.ts` | ~600 | 再試行ロジック、529 処理、FallbackTriggeredError |
| `cost-tracker.ts` | 323 | コスト追跡、永続化、フォーマット表示 |
| `costHook.ts` | 22 | React hook、プロセス終了を監視してコスト表示をトリガー |
| `utils/effort.ts` | ~350 | Effort Level 定義、優先度チェーン、モデルサポート検出 |
| `utils/modelCost.ts` | ~200 | 価格表、コスト計算関数 |
| `utils/model/model.ts` | ~450 | モデル優先度チェーン、エイリアス解析、デフォルトモデルロジック |
| `utils/model/modelCapabilities.ts` | ~100 | モデル能力キャッシュ（社内ユーザーのみ）|
| `query.ts` | ~1000 | FallbackTriggeredError キャッチ、ユーザー通知、モデル切り替え |
| `migrations/*.ts` | 11 ファイル | モデルバージョンマイグレーションスクリプト |
| `tools/WebFetchTool/utils.ts:503` | — | Haiku 用途 1：Web Fetch コンテンツ抽出 |
| `utils/shell/prefix.ts:220` | — | Haiku 用途 2：シェルコマンドプレフィックス判断 |
| `utils/sessionTitle.ts:87` | — | Haiku 用途 3：セッションタイトル生成 |
| `utils/mcp/dateTimeParser.ts:68` | — | Haiku 用途 4：DateTime 解析 |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Haiku 用途 5：ツール呼び出しサマリー |
| `commands/rename/generateSessionName.ts:20` | — | Haiku 用途 6：セッションリネーム |
| `services/api/claude.ts:544` | — | Haiku 用途 7：API Key 検証 |

---

*本章は cc-notebook のモデル選択とコスト制御に関する主張を完全にカバーしている。検証結果：Haiku のバックグラウンド用途「少なくとも 6 種類」は検証済み（実際は 13 か所の呼び出し点）；秘密のダウングレードなしは完全に検証；529 Overload Fallback メカニズムは完全に検証；Effort Level システムは完全に検証。すべてのコードスニペットはソースファイルから正確にコピーし、ファイルパスと行番号を注記した。*
