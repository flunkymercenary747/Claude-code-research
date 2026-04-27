# 第 8 章：コンテキスト管理

## 8.1 概要と位置づけ

コンテキスト管理は Claude Code アーキテクチャの中で最も重要なサブシステムの一つである。典型的なコーディングセッションは数時間にわたり、数百回のツール呼び出しを伴い、数十万 token の会話履歴が生成される。管理がなければ、コンテキストウィンドウは 20〜30 ターンのやり取りで枯渇し、セッションが中断する。

Claude Code のコンテキスト管理システムが解決する核心的な問題は次の通りである。**限られたコンテキストウィンドウ（通常 200K token）の中で、セッションの継続性と情報の完全性を維持しながら、ユーザーが感じる情報損失を最小化するにはどうすればよいか？**

このシステムは `services/compact/` ディレクトリ以下の 11 ファイル、合計約 3,900 行の TypeScript コードと、補助的な重要ユーティリティモジュールである `utils/collapseReadSearch.ts`（1,109 行）および `utils/toolResultStorage.ts`（1,040 行）で構成される。サブシステム全体の設計は三つの核心原則を体現している：

1. **段階的劣化**（Graceful Degradation）：コスト不要のマイクロ圧縮から不可逆な全量圧縮まで、介入強度を段階的に高める
2. **キャッシュ優先**（Cache-First）：すべての圧縮判断において prompt cache の保全を最優先とする
3. **安全性保証**（Safety Invariants）：tool_use/tool_result のペアは切断不可、再帰保護、サーキットブレーカー機構

## 8.2 理論的基礎

### 8.2.1 情報理論の観点：不可逆圧縮 vs 可逆圧縮

コンテキスト管理は本質的に**情報圧縮の問題**である。Claude Code の多層システムはそれぞれ異なる圧縮戦略に対応する：

- **可逆圧縮**（Lossless）：マイクロ圧縮の `cache_edits` パス——API の cache editing 機構を通じて古いツール結果のサーバーサイドキャッシュコピーを削除するが、ローカルのメッセージ内容は変更しない。モデルには `[Old tool result content cleared]` というプレースホルダーが表示されるが、元データはディスク上に保存されている（`toolResultStorage.ts`）。情報は失われておらず、ホットストレージからコールドストレージに移動しただけである。
- **不可逆圧縮**（Lossy）：全量圧縮は Fork Agent を通じてサマリーを生成し、数万 token の会話を数千 token に圧縮する。これは不可逆な次元削減プロセスであり——コードの詳細、エラースタック、中間推論はすべて失われる可能性がある。

Rate-Distortion Theory の観点から見ると、Claude Code の設計は暗黙の**歪み計量関数**を内包している。サマリー prompt の 9 つの章（8.6 節参照）は、どの情報次元が最も歪みに耐えられないかを定義している——「user messages」（完全保持）の優先度は「key technical concepts」（概括を許容）より高い。

### 8.2.2 キャッシュ理論：時間局所性と空間局所性

マイクロ圧縮のホワイトリスト機構は、古典的キャッシュ理論における**時間局所性**（Temporal Locality）の仮定を体現している：

> 最近使用されたツール結果は後続から参照される可能性が高い。

`microCompact.ts` のホワイトリスト（`COMPACTABLE_TOOLS`）は eviction policy の体現である——特定のツール（Read、Shell、Grep、Glob、WebFetch、WebSearch、Edit、Write）の結果のみ削除可能であり、それはそれらの出力が再生成可能（ツールを再実行して取得可能）だからである。一方、ユーザーが手動で入力したテキストや画像など、再生成不可能なコンテンツは決して削除されない。

```typescript
// microCompact.ts:30-41 — 圧縮可能なツールのホワイトリスト
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

`keepRecent` パラメータ（デフォルトで最近 5 件を保持）は LRU（Least Recently Used）淘汰戦略を直接実装している。

### 8.2.3 サーキットブレーカーパターン（Circuit Breaker Pattern）

`autoCompact.ts` のサーキットブレーカー機構は、分散システムにおける古典的な Circuit Breaker Pattern を LLM アプリケーションに正確に適応させたものである。このパターンは Michael Nygard の『Release It!』に由来し、三状態モデル（Closed → Open → Half-Open）が Claude Code に実装されている：

```typescript
// autoCompact.ts:70-73 — サーキットブレーカーの閾値
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

このコメントは、サーキットブレーカー導入以前の実際の惨状データを明かしている：**1,279 セッションが 50 回以上の連続失敗ループに陥り**、最悪の単一セッションでは 3,272 回の失敗を試み、世界全体で毎日約 250K 回の API 呼び出しを浪費していた。サーキットブレーカーの導入により、最大リトライ回数は 3 回に制限された。

| 状態 | 動作 | 対応コード |
|------|------|---------|
| Closed（正常） | `consecutiveFailures < 3`、通常通り圧縮を試みる | `autoCompactIfNeeded` のデフォルトパス |
| Open（トリップ） | `consecutiveFailures >= 3`、圧縮をスキップ | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open（探索） | 圧縮成功後 `consecutiveFailures` が 0 にリセット | 成功時 `consecutiveFailures: 0` |

## 8.3 アーキテクチャ概観

### 8.3.1 多層圧縮体系の全体アーキテクチャ

Claude Code のコンテキスト管理は**5 層防衛線**設計を採用している。低干渉から高干渉の順に並べると：

```
┌─────────────────────────────────────────────────────────────────┐
│                        ユーザーリクエスト                          │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage（予防層）                           │
│   大きなツール結果 → ディスク永続化 + 2KB プレビュー               │
│   トリガー: 結果 > 閾値（デフォルト 50K chars）                   │
│   コスト: ゼロコンテキスト（ディスクに保存、プレビューのみコンテキスト）│
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact（マイクロ圧縮）                            │
│   パス A: 時間トリガー — 古いツール結果の内容を削除                │
│   パス B: キャッシュ編集 — cache_edits API でサーバーキャッシュ削除 │
│   トリガー: API 呼び出し毎回の前                                  │
│   コスト: 極めて低（ツール結果はプレースホルダーに置換、ディスクから復元可）│
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact（自動圧縮）                                │
│   Session Memory → Fork Agent → 全量サマリー                    │
│   トリガー: token が effectiveContextWindow - 13K を超えた場合    │
│   コスト: 高（不可逆サマリー、詳細損失、API 呼び出し 1 回分）       │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact（手動圧縮）                             │
│   ユーザーが能動的に起動、Partial Compact をサポート               │
│   トリガー: ユーザーコマンド                                      │
│   コスト: 同上                                                   │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact（リアクティブ圧縮）                    │
│   API が prompt_too_long を返す → 切り捨てて再試行                │
│   トリガー: 413 エラー                                           │
│   コスト: 最高（緊急切り捨て + サマリー、情報損失が最大）           │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 各層のトリガー条件、コスト、情報損失の比較

| 層 | トリガー条件 | タイミング | レイテンシ | 情報損失 | API コスト |
|------|---------|------|------|---------|---------|
| L0: Tool Result Storage | 単一ツール結果 > 閾値 | ツール実行後 | ディスク I/O | ゼロ（原文をディスクに保存） | ゼロ |
| L1a: Time-based MC | 前回 assistant から 60 分以上経過 | API 呼び出し前 | ゼロ（ローカル操作） | 低（古い結果を削除） | ゼロ |
| L1b: Cached MC | 圧縮可能なツール数が閾値超 | API 呼び出し前 | ゼロ（cache_edits） | 低（同上） | ゼロ |
| L2: Auto-Compact | token > threshold | ターン間 | 5〜15 秒（API 呼び出し） | 高（不可逆サマリー） | API 呼び出し 1 回 |
| L3: Manual Compact | ユーザー /compact | ユーザートリガー | 同上 | 中〜高（ユーザーが指示可能） | API 呼び出し 1 回 |
| L4: Reactive Compact | prompt_too_long 413 | API 失敗後 | 10〜30 秒（再試行） | 最高（切り捨て + サマリー） | 1〜4 回の API 呼び出し |

### 8.3.3 データフロー

```
メッセージ配列 (Message[])
    │
    ▼
microcompactMessages()  ──→ [時間トリガー?] ──Y──→ 内容削除 → 返す
    │ N                      │
    │                  [キャッシュ編集?] ──Y──→ pendingCacheEdits → 返す
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     圧縮なし、そのまま返す
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [session memory あり?]
    │ N                              │ Y
    ▼                                ▼
compactConversation()           calculateMessagesToKeepIndex()
    │                                │
    ▼                                ▼
streamCompactSummary()          buildPostCompactMessages()
    │ (Fork Agent)
    ▼
formatCompactSummary()
    │
    ▼
新メッセージ配列: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 第 1 層：マイクロ圧縮（Microcompact）

マイクロ圧縮はコンテキスト管理の第一防衛線である。**API 呼び出しの直前**に実行され（`microcompactMessages` エントリーポイント）、最小コストでコンテキスト空間を解放することを目標とする。

### 8.4.1 圧縮可能なツールのホワイトリスト

マイクロ圧縮は特定ツールの出力にのみ作用する。ホワイトリストの背後にある設計原則は：**再生成可能なコンテンツのみ削除する**。

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // ファイル読み取り — 再読み取り可
  ...SHELL_TOOL_NAMES,     // Shell コマンド — 再実行可
  GREP_TOOL_NAME,          // 検索 — 再検索可
  GLOB_TOOL_NAME,          // ファイルマッチ — 再マッチ可
  WEB_SEARCH_TOOL_NAME,    // Web 検索 — 再検索可
  WEB_FETCH_TOOL_NAME,     // Web フェッチ — 再フェッチ可
  FILE_EDIT_TOOL_NAME,     // ファイル編集 — 結果はディスクに保存済み
  FILE_WRITE_TOOL_NAME,    // ファイル書き込み — 同上
])
```

`apiMicrocompact.ts` にはさらに細粒度の区別が定義されている：

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // tool_result の内容を削除
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // tool_use の入力を削除
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

この区別は巧妙である。Read/Grep/Shell については**出力**（tool_result）を削除し、Edit/Write については**入力**（tool_use input）を削除する。編集操作の入力（差分内容）は大きいが結果はすでにディスクに永続化されているからだ。

### 8.4.2 二つのサブパスの詳細

マイクロ圧縮には相互排他的な二つの実行パスがあり、`microcompactMessages()` 関数が統一的にディスパッチする：

```typescript
// microCompact.ts:287-317 — ディスパッチロジック
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // パス A: 時間トリガー — 最高優先度、後続パスをショートサーキット
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // パス B: キャッシュ編集 — メインスレッドのみ、特定モデルのみサポート
  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    const model = toolUseContext?.options.mainLoopModel ?? getMainLoopModel()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  return { messages }
}
```

**パス A: Time-based Microcompact（時間トリガー）**

ユーザーがセッションを離れて設定された時間閾値（デフォルト 60 分）を超えた後に戻ってきたときにトリガーされる。設計根拠は `timeBasedMCConfig.ts` に明記されている：

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

サーバーサイドの prompt cache の TTL は 1 時間である。ユーザーが 1 時間以上離れていた場合、**キャッシュは必ず失効**しており、プロンプトプレフィックス全体を再書き込みする必要がある。この時点で古いツール結果を削除することは「無料」——追加のキャッシュ失効コストが発生しないためだ。

時間トリガーの主要ロジック：

```typescript
// microCompact.ts:381-389 — 時間トリガーの評価
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) {
    return null
  }
  const gapMinutes =
    (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  return { gapMinutes, config }
}
```

時間トリガー後の削除戦略も LRU（`keepRecent` デフォルト 5）を使用するが、境界保護がある：

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

この `Math.max(1, ...)` は、`keepRecent=0` の場合に `slice(-0)` が全配列を返す JavaScript の落とし穴を防いでいる——「防御的プログラミングによるセマンティクスの曖昧さの回避」の典型例だ。

時間トリガー後にはキャッシュ編集状態もリセットする必要がある：

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**パス B: Cached Microcompact（キャッシュ編集）**

これは Anthropic 内部の高度な最適化パス（`feature('CACHED_MICROCOMPACT')`）であり、API の `cache_edits` 機構を利用して、**ローカルのメッセージ内容を変更せずに**サーバーサイドキャッシュ内のツール結果を削除する。

```typescript
// microCompact.ts:327-370 — キャッシュ編集パスの核心
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // ツール結果を登録
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && 
            compactableToolIds.has(block.tool_use_id) &&
            !state.registeredTools.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  const toolsToDelete = mod.getToolResultsToDelete(state)
  if (toolsToDelete.length > 0) {
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits
    }
    // ...
    return {
      messages,  // メッセージは変更なし!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

重要な設計決定：**メッセージ配列は変更されない**——`return { messages }` は元の参照を返す。キャッシュ編集は API 層で発生し（`cache_edits` パラメーター経由）、ローカル状態は完全なまま保たれる。これにより、API 呼び出しが失敗またはリトライされても、ローカルへの副作用は一切ない。

### 8.4.3 キャッシュ編集の状態管理

キャッシュ編集パスは三つの主要な状態を維持する：

```typescript
// microCompact.ts:43-49 — モジュールレベルの状態
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

これらの状態のライフサイクル管理は繊細である：

- `pendingCacheEdits` は使い捨て——`consumePendingCacheEdits()` で読み取った後にクリアされ（`microCompact.ts:80-84`）、呼び出し元は API リクエストで送信後にピン留めしなければならない。
- `pinnedCacheEdits` は累積的——成功したキャッシュ編集はそれぞれ特定の user message の位置にピン留めされ、後続のリクエストでは同じ位置に再送して確実にキャッシュヒットさせる。
- `cachedMCState` は圧縮後（`resetMicrocompactState()`）または時間トリガー後にリセットされる。

```typescript
// microCompact.ts:78-105 — 状態の消費とピン留め
export function consumePendingCacheEdits() {
  const edits = pendingCacheEdits
  pendingCacheEdits = null
  return edits
}

export function getPinnedCacheEdits() {
  if (!cachedMCState) return []
  return cachedMCState.pinnedEdits
}

export function pinCacheEdits(
  userMessageIndex: number,
  block: import('./cachedMicrocompact.js').CacheEditsBlock,
): void {
  if (cachedMCState) {
    cachedMCState.pinnedEdits.push({ userMessageIndex, block })
  }
}
```

### 8.4.4 Token 推定補助関数

マイクロ圧縮モジュールはシステム全体で共有される token 推定関数を提供する：

```typescript
// microCompact.ts:155-194 — estimateMessageTokens
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue
    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 固定 2000
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'tool_use') {
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {}),
        )
      }
      // ...
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // 4/3 の保守的パディング
}
```

`roughTokenCountEstimation` の核心的な計算式は極めてシンプルである：`Math.round(content.length / 4)`（`tokenEstimation.ts:203-207`）。最終的に `estimateMessageTokens` はこの結果に 4/3 の保守的係数をかけており、`text.length / 3` に相当する。この二重保守戦略により、過小評価の確率は極めて低くなっている。

## 8.5 第 2 層：自動圧縮（Auto-Compact）

### 8.5.1 閾値計算式

自動圧縮のトリガー閾値は以下の式で計算される：

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

具体的な数値の導出（Claude Opus 200K を例に）：

```
contextWindow = 200,000
maxOutputTokens = 16,384 (またはモデル固有の値)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (p99.99 = 17,387 に基づく)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — 主要な定数
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

`AUTOCOMPACT_BUFFER_TOKENS = 13,000` の選択はエンジニアリングのトレードオフである。小さすぎると圧縮が頻発し（毎回 5〜15 秒と API 呼び出し 1 回を消費）、大きすぎると利用可能なコンテキストが無駄になる。13K は通常の 3〜5 ターンの会話に相当する空間である。

### 8.5.2 shouldAutoCompact の決定木

```typescript
// autoCompact.ts:127-178 — 完全な決定チェーン
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. 再帰保護：session_memory と compact のクエリソースはトリガーしない
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. コンテキスト折りたたみ保護：marble_origami（ctx-agent）はトリガーしない
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. 設定チェック：ユーザーが有効にしているか
  if (!isAutoCompactEnabled()) return false

  // 4. リアクティブモード：有効な場合は積極的な圧縮を抑制する
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. コンテキスト折りたたみモード：折りたたみはコンテキスト管理であり、圧縮は干渉すべきでない
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Token カウント + 閾値比較
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

この決定木は Claude Code が並行して実験している複数のコンテキスト管理戦略を露呈している：
- **Reactive Compact**（`tengu_cobalt_raccoon`）：積極的に圧縮せず、API が prompt_too_long を報告するまで待つ
- **Context Collapse**（`CONTEXT_COLLAPSE`）：90% コミット / 95% ブロックのストリーミング方式でコンテキストを管理する
- **Auto Compact**（現在のデフォルト）：閾値に達したときに積極的に圧縮する

三つは相互排他的で、feature flag で制御される。

### 8.5.3 サーキットブレーカー機構

```typescript
// autoCompact.ts:219-272 — サーキットブレーカーを含む autoCompactIfNeeded
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // サーキットブレーカーチェック
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // トリップ状態、直接スキップ
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // Session Memory 圧縮を優先して試みる
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 従来の圧縮
  try {
    const compactionResult = await compactConversation(...)
    runPostCompactCleanup(querySource)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const nextFailures = (tracking?.consecutiveFailures ?? 0) + 1
    if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
      logForDebugging(
        `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`,
        { level: 'warn' })
    }
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

### 8.5.4 autoCompactIfNeeded の実行フロー

完全な実行順序：

1. **環境変数チェック**：`DISABLE_COMPACT` → グローバル無効化
2. **サーキットブレーカーチェック**：`consecutiveFailures >= 3` → スキップ
3. **閾値チェック**：`shouldAutoCompact()` → 多層ゲート
4. **Session Memory 圧縮**（優先パス）：既存の session memory を活用し API 呼び出しを代替
5. **従来の Fork Agent 圧縮**（フォールバックパス）：完全な API 駆動サマリー生成
6. **失敗処理**：サーキットブレーカーカウンターをインクリメントし、次のターンに引き継ぐ

## 8.6 第 3 層：従来の圧縮（Full Compact）

### 8.6.1 Fork Agent 機構

従来の圧縮の核心は Fork Agent を通じた会話サマリーの生成である。`streamCompactSummary()` 関数（`compact.ts:1136-1396`）は二段階フォールバック戦略を実装している：

**第一段：Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — キャッシュ共有 fork
if (promptCacheSharingEnabled) {
  try {
    const result = await runForkedAgent({
      promptMessages: [summaryRequest],
      cacheSafeParams,
      canUseTool: createCompactCanUseTool(),
      querySource: 'compact',
      forkLabel: 'compact',
      maxTurns: 1,
      skipCacheWrite: true,
      overrides: { abortController: context.abortController },
    })
    // ...
  }
}
```

Fork Agent はメイン会話の完全な prompt cache（system prompt + tools + context messages）を再利用し、サマリーリクエストを 1 件追加するだけである。主要な設計点：

1. `maxTurns: 1` — 複数ターンのやり取りを許可しない
2. `canUseTool: createCompactCanUseTool()` — すべてのツール呼び出しを拒否
3. `skipCacheWrite: true` — キャッシュに書き込まない（一時的な分岐）
4. **maxOutputTokens を設定しない** — コメントによれば：設定すると thinking config が変わり、cache key が不一致になるためだ

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**第二段：Streaming Fallback**

Fork Agent が失敗した場合、直接のストリーミング API 呼び出しにフォールバックし、このときは `maxOutputTokensOverride` を**設定できる**：

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

ストリーミングフォールバックは設定駆動のリトライもサポートしており（`tengu_compact_streaming_retry`）、最大 `MAX_COMPACT_STREAMING_RETRIES = 2` 回。

### 8.6.2 前処理パイプライン

圧縮前のメッセージは三段階の前処理を経る：

```typescript
// compact.ts:1293-1300 — 前処理チェーン
normalizeMessagesForAPI(
  stripImagesFromMessages(
    stripReinjectedAttachments([
      ...getMessagesAfterCompactBoundary(messages),
      summaryRequest,
    ]),
  ),
  context.options.tools,
)
```

1. `getMessagesAfterCompactBoundary` — 最後の圧縮以降のメッセージのみ取得
2. `stripReinjectedAttachments` — `skill_discovery` / `skill_listing` 添付ファイルを除去（圧縮後に再注入される）
3. `stripImagesFromMessages` — 画像ブロックを `[image]` テキストマーカーに置換（`compact.ts:144-199`）

`stripImagesFromMessages` の存在理由は実務的である：

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

CCD（Claude Code Desktop）ユーザーはスクリーンショットを頻繁に添付するが、画像を除去しなければ、圧縮 API 呼び出し自体がプロンプト過長で失敗する可能性がある。

### 8.6.3 サマリー出力の 9 章構成フォーマット

`prompt.ts` はサマリーが従うべき 9 つの構造化チャプターを定義している：

```
1. Primary Request and Intent    — ユーザーの意図
2. Key Technical Concepts        — 技術的概念
3. Files and Code Sections       — ファイルとコードスニペット
4. Errors and fixes              — エラーと修正
5. Problem Solving               — 問題解決
6. All user messages             — すべてのユーザーメッセージ（tool result 以外）
7. Pending Tasks                 — 保留中のタスク
8. Current Work                  — 現在の作業
9. Optional Next Step            — 次のステップ（オプション）
```

チャプター 6 の設計は特に重要である——「List ALL user messages that are not tool results」。これにより、会話が圧縮されてもユーザーの元の表現が完全に保持される。これが**ユーザーフィードバック情報ゼロ損失**の保証である。

チャプター 9 には丁寧に設計された制約がある：

```
// prompt.ts — チャプター 9 の制約
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

これにより、圧縮後のモデルが「勝手に行動する」のを防いでいる——ユーザーが明示的に要求した次のステップのみが記録される。

### 8.6.4 NO_TOOLS_PREAMBLE による回避防止設計

Fork Agent はメイン会話の完全なツールセットを引き継ぐ（cache key の一致のため）が、圧縮エージェントはどのツールも使用すべきでない。これはジレンマを生む：ツールはスキーマ上存在するが、呼び出されるべきでない。

解決策は**三層のツール拒否**である：

```typescript
// prompt.ts:16-24 — 第一層：プロンプト冒頭の強力な宣言
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — 第二層：プロンプト末尾の繰り返しリマインダー
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — 第三層：コードレベルの拒否
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

コメントは三層設計の実際の理由を明かしている：

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

Sonnet 4.6 では、プロンプト指示だけでは 2.79% の確率でツール呼び出しを試みる（4.5 では 0.01% のみ）。`createCompactCanUseTool` は最後のコードレベルの安全網である。

### 8.6.5 後処理（formatCompactSummary）

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // <analysis> 草稿エリアを除去 — サマリー品質向上のための中間推論、保持不要
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // <summary> の内容を抽出
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${summaryMatch[1].trim()}`)
  }
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')
  return formattedSummary.trim()
}
```

`<analysis>` タグの設計は Chain-of-Thought のテクニックである。モデルに分析エリアで「下書き」させた後、`<summary>` に最終出力させる。分析エリアの存在はサマリーの品質を向上させるが、最終出力では除去される——冗長な中間推論が後続ターンのコンテキスト空間を浪費するためだ。

### 8.6.6 圧縮後のメッセージシーケンスと添付ファイルの再注入

圧縮完了後、新しいメッセージシーケンスは `buildPostCompactMessages()` で構築される：

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // システムメッセージ：圧縮境界をマーク
    ...result.summaryMessages,    // ユーザーメッセージ：サマリー内容
    ...(result.messagesToKeep ?? []),  // 保持された元のメッセージ
    ...result.attachments,        // ファイル添付 + スキル + プラン
    ...result.hookResults,        // SessionStart フックの結果
  ]
}
```

添付ファイルの再注入は複雑なプロセスであり（`compact.ts:532-585`）、以下を含む：

1. **ファイル添付**：最近アクセスした上位 5 ファイル、50K token 予算制約下で各ファイル最大 5K token
2. **プランファイル**：アクティブなプランがある場合
3. **プランモード指示**：プランモードの場合
4. **スキルコンテンツ**：呼び出されたスキルの内容、最近使用順で各スキル最大 5K token、合計 25K token の予算
5. **Deferred Tools Delta**：遅延読み込みツールのスキーマを再宣言
6. **Agent Listing Delta**：エージェントリストを再宣言
7. **MCP Instructions Delta**：MCP サーバー指示を再宣言

### 8.6.7 PTL リトライ機構（Prompt-Too-Long Recovery）

圧縮 API 呼び出し自体がプロンプト過長で失敗した場合、システムは段階的な切り捨てによりリトライする：

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 以前のリトライで残ったマーカーメッセージを先にクリア
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // 精密切り捨て：API が返した token 差分に基づく
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 大まかな切り捨て：20% のメッセージグループを削除
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // 少なくとも 1 グループを保持
  if (dropCount < 1) return null

  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

リトライの上限は `MAX_PTL_RETRIES = 3` である。切り捨て戦略には二つのパスがある：
- **精密パス**：API エラーに token 差分が含まれる場合 → 差分に基づいてグループを削除
- **大まかなパス**（Vertex/Bedrock など非標準エラー形式）：毎回 20% を削除

283 行目の境界処理に注意：グループ 0 を削除した後、メッセージシーケンスが assistant メッセージで始まる可能性があり、API 制約（最初のメッセージは user でなければならない）に違反する。システムは合成された user マーカーメッセージを挿入して修正する。

### 8.6.8 部分圧縮（Partial Compact）の二方向

`partialCompactConversation()`（`compact.ts:772-1106`）は二方向をサポートする：

```
Direction 'from': 
  [圧縮後に保持] | pivot | [サマリー化されたメッセージ]
  → prompt cache を保持（保持部分が先頭、cache prefix は変わらない）

Direction 'up_to':
  [サマリー化されたメッセージ] | pivot | [圧縮後に保持]
  → prompt cache が無効化（サマリーが先頭、プレフィックスが変わる）
```

`up_to` 方向には追加のクリーンアップロジックがある——保持されたメッセージから古い compact boundary と summary を除去しなければならない：

```typescript
// compact.ts:791-799
const messagesToKeep =
  direction === 'up_to'
    ? allMessages.slice(pivotIndex)
        .filter(m =>
          m.type !== 'progress' &&
          !isCompactBoundaryMessage(m) &&
          !(m.type === 'user' && m.isCompactSummary))
    : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

コメントがその理由を説明している：`up_to` モードではサマリーが保持メッセージの前にあるため、古い boundary が `findLastCompactBoundaryIndex` の逆方向スキャンを誤誘導する。

## 8.7 第 4 層：Session Memory 圧縮

### 8.7.1 核心的なアイデアと利点

Session Memory 圧縮（`sessionMemoryCompact.ts`）は従来の圧縮の最適化された代替手段である。核心的なアイデア：バックグラウンドで継続的に抽出された session memory（会話のインクリメンタルサマリー）を活用して、リアルタイムで生成する Fork Agent サマリーを置き換える。

利点：
- **追加 API 呼び出しなし**：session memory はバックグラウンドエージェントが継続的に維持し、圧縮時に直接使用
- **より低レイテンシ**：5〜15 秒の API レスポンスを待つ必要がない
- **より細粒度の保持**：最近のメッセージをいくつ保持するかを正確に計算できる

### 8.7.2 calculateMessagesToKeepIndex アルゴリズムの詳細

これは Session Memory 圧縮の核心アルゴリズムであり（`sessionMemoryCompact.ts:262-327`）、圧縮後に保持するメッセージ数を決定する：

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // lastSummarizedIndex + 1 から開始（session memory がそれ以前をカバー済み）
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // 現在の保持範囲の token 数とテキストメッセージ数を計算
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // 上限に達した → 拡張しない
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 二つの最低要件を満たした → 拡張しない
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 前方に拡張するが、最後の compact boundary は超えない
  const idx = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  const floor = idx === -1 ? 0 : idx + 1

  for (let i = startIndex - 1; i >= floor; i--) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
    startIndex = i
    if (totalTokens >= config.maxTokens) break
    if (totalTokens >= config.minTokens && 
        textBlockMessageCount >= config.minTextBlockMessages) break
  }

  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

設定パラメータ（GrowthBook リモート設定で上書き可能）：

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // 最低 10K token を保持
  minTextBlockMessages: 5,     // 最低 5 件のテキストメッセージを保持
  maxTokens: 40_000,           // 最大 40K token を保持
}
```

アルゴリズムの二重制約設計（`minTokens` AND `minTextBlockMessages`）により：
- 少数の超大型メッセージで token は満たしても拡張を停止しない（token は満たしてもメッセージが少なすぎる）
- 小さなメッセージが多すぎても実際の token が不足しない

**Floor 機構**：前方への拡張は最後の compact boundary を超えてはならない（`floor = lastBoundaryIndex + 1`）。コメントが理由を説明している：

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

ディスクストレージ層のメッセージチェーンは compact boundary で不連続になっており、それを超えるとローダーの逆方向走査が保持されたメッセージを飛ばしてしまう。

### 8.7.3 adjustIndexToPreserveAPIInvariants のバグ修正

この関数（`sessionMemoryCompact.ts:172-260`）は圧縮システム全体の中で最も精巧なコードの一つであり、二つの API 不変条件の問題を解決している：

**バグシナリオ 1：孤立した tool_result**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

startIndex = N+2 の場合:
  旧コードはメッセージ N+2 の tool_results のみチェック → 見つからない → N+2 を返す
  normalizeMessagesForAPI が message.id でマージ後:
    msg[1]: assistant with [tool_use: VALID_ID]  (ORPHAN tool_use が除外される!)
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → API エラー：orphan tool_result が存在しない tool_use を参照している
```

**バグシナリオ 2：thinking ブロックの消失**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

startIndex = N+1 の場合:
  thinking ブロックが N で除外される
  normalizeMessagesForAPI はマージできない（同じ ID のメッセージがない）
  → thinking ブロックが永久に消失する
```

修正コードは二段階の調整を実行する：

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Step 1: tool_use/tool_result のペアを処理
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... 保持範囲内に既にある tool_use ID を収集
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // 不足している tool_use を前方に検索
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 見つかった ID を削除
      }
    }
  }

  // Step 2: message.id を共有する thinking ブロックを処理
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id)
      messageIdsInKeptRange.add(messages[i]!.message.id)
  }
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id &&
        messageIdsInKeptRange.has(messages[i]!.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

このコードの重要な洞察は：Claude API のストリーミングレスポンスは同一の API 返答を複数の assistant メッセージに分割する（`message.id` を共有するが UUID は異なる）。thinking ブロックと tool_use ブロックは別々になっている。`normalizeMessagesForAPI` はこれらのメッセージを `message.id` でマージする——圧縮が同じ ID のメッセージグループを切断すると、マージ後に不整合が発生する。

### 8.7.4 trySessionMemoryCompaction の完全フロー

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. ゲートチェック
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. リモート設定を初期化（初回のみ）
  await initSessionMemoryCompactConfig()

  // 3. 実行中の session memory 抽出が完了するのを待つ
  await waitForSessionMemoryExtraction()

  // 4. session memory コンテンツを取得
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. 境界を決定
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID が存在しない → フォールバック
  } else {
    // 再開されたセッション: 境界なし → 末尾から開始
    lastSummarizedIndex = messages.length - 1
  }

  // 6. 保持範囲を計算
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // 古い boundary をフィルタリング

  // 7. session start フックを実行
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. 結果を構築
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. 閾値チェック（autocompact のみ）
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // 圧縮後も閾値超 → 従来の圧縮にフォールバック
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 設定パラメータ（GrowthBook リモート設定）

Session Memory 圧縮のすべての主要パラメータは GrowthBook リモート設定で上書き可能である：

```typescript
// sessionMemoryCompact.ts:91-109 — リモート設定の初期化
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // 防御的コーディング：正の値のみ使用、0 は無視
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

機能ゲートは二つの独立した feature flag で制御される：

```typescript
// sessionMemoryCompact.ts:333-349
export function shouldUseSessionMemoryCompaction(): boolean {
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) return true
  if (isEnvTruthy(process.env.DISABLE_CLAUDE_CODE_SM_COMPACT)) return false
  
  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory', false)
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact', false)
  return sessionMemoryFlag && smCompactFlag
}
```

## 8.8 コンテキスト折りたたみとツール結果ストレージ

### 8.8.1 collapseReadSearch 機構

`utils/collapseReadSearch.ts`（1,109 行）は UI 層のメッセージ折りたたみを実装している——連続する検索/読み取り操作を単一行のサマリーに折りたたむ（例：「Read 5 files, searched 3 patterns」）。

核心的な分類ロジック（`getToolSearchOrReadInfo`、`collapseReadSearch.ts:142-237`）はツール呼び出しを以下に分類する：

| カテゴリ | 折りたたみ可 | 折りたたみ動作 |
|------|-------|---------|
| Read (file_path) | はい | "Read N files" |
| Search (Grep/Glob) | はい | "Searched N patterns" |
| Shell (Bash) | フルスクリーンモードでははい | "Ran N bash commands" |
| REPL | はい（サイレント吸収） | 内部ツールは個別にカウント |
| Memory Write | はい | 特殊マーク |
| ToolSearch | はい（サイレント吸収） | カウントは増加しない |
| Edit/Write (memory 以外) | いいえ | 折りたたみグループを分断 |

「サイレント吸収」（`isAbsorbedSilently`）は精巧な設計である。REPL と ToolSearch はカウンターを増加させないが、現在の折りたたみグループも分断しない。つまり `[Read, ToolSearch, Read]` は「Read 2 files」に折りたたまれ、ToolSearch で二つのグループに分けられることはない。

折りたたみは**UI 層のみ**の最適化であり——API に送信されるメッセージ内容を変更せず、端末表示にのみ影響する。

### 8.8.2 toolResultStorage のディスクストレージ戦略

`utils/toolResultStorage.ts`（1,040 行）はコンテキスト管理の「第零層」——ツール結果が会話履歴に入る前に超大型結果を処理する。

**永続化閾値の解析**：

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Read ツールは特殊：Infinity → 永続化しない（Read 自体に maxTokens 制限がある）
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // GrowthBook 上書き（tengu_satin_quoll）
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // デフォルト：min(ツール宣言値, グローバル 50K デフォルト)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**重複排除の最適化**：`tool_use_id` は一意であり、`flag: 'wx'`（exclusive write）を使って重複書き込みを防ぐ：

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: 以前のターンで既に永続化済み、スキップ
}
```

**空の結果の処理**：

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

この修正はモデルの動作バグを解決している：空の tool_result が原因で一部のモデルが `\n\nHuman:` パターンを会話終了として解釈してしまう。

**Per-Message Aggregate Budget**（`enforceToolResultBudget`、`toolResultStorage.ts:769-908`）：

これは `toolResultStorage.ts` の中で最も複雑な機能であり、API レベルの user メッセージ（`normalizeMessagesForAPI` によるマージ後）に対して、ツール結果の総サイズ予算を強制する。

設計のポイント：
- **状態の凍結**（`ContentReplacementState`）：ある tool_result が「閲覧済み」（モデルに送信済み）になると、その決定（置換する/しない）が凍結され、生存期間中は絶対に変更されない——これにより prompt cache の安定性が保証される
- **三区分**戦略：`mustReapply`（以前置換済み → キャッシュされた置換内容を再適用）、`frozen`（以前閲覧したが置換しなかった → 変更しない）、`fresh`（新規 → 置換の可能性あり）
- **最大優先**：置換が必要な場合、最大の fresh 結果を優先して置換する

## 8.9 5 層エラーリカバリにおける圧縮の役割

### 8.9.1 完全な 5 層エラーリカバリ機構

圧縮システムは Claude Code のエラーリカバリ機構の中で複数の役割を担う：

| 層 | トリガー条件 | 圧縮動作 | 出典 |
|------|---------|---------|------|
| L1 | API が prompt_too_long (413) を返す | Reactive Compact：切り捨て + 再サマリー | `compactMessages.ts` |
| L2 | 圧縮 API 自体が 413 を返す | PTL Retry：最古のメッセージグループを切り捨て × 3 回 | `compact.ts:truncateHeadForPTLRetry` |
| L3 | 圧縮後も閾値超 | Re-compaction：自動で再圧縮 | `autoCompact.ts:recompactionInfo` |
| L4 | 3 回連続で圧縮失敗 | Circuit Breaker：試行を停止 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent がテキスト出力なし | Streaming Fallback：直接ストリーミング API 呼び出し | `compact.ts:streamCompactSummary` |

### 8.9.2 リアクティブ圧縮 vs 積極的圧縮

二つの戦略のトレードオフ：

**積極的圧縮**（Auto-Compact、現在のデフォルト）：
- token が閾値に達したときに積極的にトリガー
- 利点：ユーザー体験がよりスムーズ、413 エラーに遭遇しない
- 欠点：早期に圧縮しすぎる可能性があり、利用可能なコンテキストを無駄にする

**リアクティブ圧縮**（Reactive Compact、`tengu_cobalt_raccoon` 実験）：
- API が prompt_too_long を報告するまで待つ
- 利点：コンテキスト使用率を最大化
- 欠点：ユーザー体験に明確な中断があり、再試行を待つ必要がある

コードには二つの相互排他的な関係が確認できる：

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // リアクティブモードでは積極的に圧縮しない
  }
}
```

## 8.10 メッセージのグループ化と Token 推定

### 8.10.1 groupMessagesByApiRound アルゴリズム

`grouping.ts`（63 行）はメッセージを API ラウンド単位でグループ化する——各グループは完全な API の往復に対応する：

```typescript
// grouping.ts:28-62
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (msg.type === 'assistant' && 
        msg.message.id !== lastAssistantId && 
        current.length > 0) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }
  if (current.length > 0) groups.push(current)
  return groups
}
```

グループの境界を判断する唯一の根拠は `message.id` の変化である——同一の API レスポンスの複数のストリーミングブロックは同じ `message.id` を共有するため、自然に同じグループに属する。

この設計は以前の「人間ターン」ベースのグループ化（実際のユーザーメッセージでのみグループ化）に取って代わった。前者は SDK/CCR/eval シナリオでの長時間の単一ターン agent セッションを処理できなかった。

### 8.10.2 roughTokenCountEstimation と保守的なパディング

token 推定は二段階の保守的戦略を採用している：

**第一段**：基本推定

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

デフォルトは 4 bytes/token。JSON ファイルは 2 bytes/token（JSON には `{`、`}`、`:`、`,` などの単一文字 token が多いため）。

**第二段**：メッセージレベルのパディング

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

組み合わせた効果：通常のテキストに対して、有効な推定は `text.length / 4 * 4/3 = text.length / 3` となる。

### 8.10.3 精確 vs 推定の混合戦略

システムは異なるシナリオで異なる精度を使用する：

| シナリオ | 精度 | 出典 | レイテンシ |
|------|------|------|------|
| shouldAutoCompact | 混合：API が返した精確な値を優先 | `tokenCountWithEstimation` | 0（キャッシュ済み） |
| estimateMessageTokens | 粗い推定（`text.length/3`） | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | 粗い推定 | `estimateMessageTokens` | 0 |
| 圧縮後の token 集計 | 精確 | `tokenCountFromLastAPIResponse` | 0（API が既に返した） |

`tokenCountWithEstimation` の混合戦略は：最近の API レスポンスの `usage.input_tokens`（精確値）を優先して使用し、利用できない場合（例：最初のリクエスト前）は推定にフォールバックする。

## 8.11 設計上の意思決定分析

### 8.11.1 段階的劣化の哲学

Claude Code のコンテキスト管理は**スキップしない**段階的劣化を採用している。各層は最小コストで問題を解決しようとし、現在の層が失敗した場合のみ次の層に進む。これにより一般的な「過剰反応」問題を回避できる——例えば、一つの大きなファイルの Read 結果だけで全量圧縮をトリガーしてしまうことなど。

業界慣行との比較：
- **ChatGPT**：古いメッセージを切り捨て（シンプルだが乱暴）
- **GitHub Copilot Chat**：固定コンテキストウィンドウ + 最近 N 件のメッセージ（圧縮なし）
- **Claude Code**：5 層の段階的手順（予防 → 微調整 → サマリー → 緊急リカバリ）

### 8.11.2 キャッシュ優先設計

Prompt cache は Claude Code の命綱である——200K token のリクエストで 180K が cache read（$0.30/M token）であれば、すべてが cache miss（$3/M token）の場合と比べてコストが 10 倍低くなる。ほぼすべての設計決定はこの経済的制約に従っている：

1. **Fork Agent がキャッシュプレフィックスを共有**：圧縮 API 呼び出しがメイン会話のキャッシュを再利用
2. **Fork で maxOutputTokens を設定しない**：thinking config の不一致によるキャッシュミスを回避
3. **Cached MC がローカルメッセージを変更しない**：prompt prefix を変更しない
4. **ContentReplacementState が閲覧済み ID を凍結**：同じ tool_result の置換決定が生存期間中変わらないことを保証
5. **sentSkillNames をリセットしない**：約 4K token の skill_listing を再注入することを回避
6. **pinnedCacheEdits を固定位置で再送**：cache edit の位置が一致することを保証

### 8.11.3 安全性保証

システムは三種の不変条件を維持する：

**ペアを切断不可**：`adjustIndexToPreserveAPIInvariants` は tool_use と tool_result が決して異なる側に分割されないことを保証する。これは機能的正確性（API がエラーを報告する）だけでなく、意味的正確性（モデルは以前呼び出したツールの結果を見る必要がある）の要件でもある。

**再帰保護**：`shouldAutoCompact` の `querySource` チェックにより、session_memory エージェント、compact エージェント、context collapse エージェントが自動圧縮をトリガーしないことが保証される——これらのエージェント自体がコンテキスト管理の一部であり、再帰的な圧縮はデッドロックを引き起こす。

**サーキットブレーカー機構**：`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` は実際のデータ（1,279 セッションの失敗ループ）に基づいて設定されており、無限リトライを有限リトライ + サーキットブレーカーに変えた。

### 8.11.4 API ネイティブのコンテキスト管理との比較

`apiMicrocompact.ts` は Claude Code が一部のコンテキスト管理を API 層にオフロードする方向を模索していることを明らかにしている：

```typescript
// apiMicrocompact.ts:37-47
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

これらの `context_management.edits` 戦略は API リクエストで直接宣言され、サーバーサイドで実行される。利点はレイテンシが低く（クライアントサイドの処理が不要）、サーバーサイドの token カウントと精確に一致することである。現在、ツール削除戦略は内部ユーザー（`USER_TYPE === 'ant'`）のみに開放されており、外部ユーザーは thinking 削除のみを使用する。

## 8.12 移植可能なパターン

### 8.12.1 多層圧縮体系の汎用設計パターン

Claude Code のコンテキスト管理から抽出できる移植可能な汎用パターン：

**パターン 1：階層的淘汰（Tiered Eviction）**
- 異なる種類のコンテンツに異なる淘汰戦略を適用する
- 再生成可能なコンテンツ（ツール出力）を優先的に淘汰し、再生成不可能なコンテンツ（ユーザー入力）は最後に淘汰する
- 実装方法：ホワイトリスト + 優先度ソート

**パターン 2：推定と精確の混合（Hybrid Estimation）**
- 迅速な判断には粗い推定（`text.length / 3`）、精確な計算には API の返した値を使用する
- 粗い推定は常に保守的（高く見積もって早期圧縮を引き起こしても、低く見積もって API エラーになるよりよい）

**パターン 3：凍結と再生（Freeze-Replay）**
- コンテンツがモデルに「閲覧」されたら、その処理決定を凍結する
- 後続ターンでは凍結されたコンテンツに「再生」（cached replacement を再適用）のみ行い、新しい決定は行わない
- ビット単位の prompt prefix の安定性を保証 → キャッシュヒット

**パターン 4：境界を意識した切り捨て（Boundary-Aware Truncation）**
- 意味的単位の途中で切り捨てない（tool_use/tool_result のペア、同 ID のメッセージグループ）
- 切り捨て後に積極的に修正する（合成メッセージの挿入、インデックスの調整）

**パターン 5：サーキットブレーカー保護（Circuit Breaker Protection）**
- 無限リトライの可能性がある操作に失敗カウンターを設ける
- 直感ではなく実際の運用データに基づいて閾値を設定する

### 8.12.2 Doramagic から借鑑できる点

Doramagic の Soul Extractor パイプラインでは、抽出プロセスで大量の中間結果（コードスニペット、API ドキュメント、コミュニティ議論）が生成される可能性がある。借鑑できるパターン：

1. **階層的抽出キャッシュ**：microcompact のホワイトリスト機構と同様に、中間 API レスポンスとコード分析結果を再生成可能性で分類し、再取得可能なコンテンツを優先的に淘汰する
2. **インクリメンタルサマリー**：Session Memory Compact と同様に、完全な履歴ではなく抽出された知識のインクリメンタルサマリーを維持する
3. **決定の凍結**：知識ブロックが「価値あり」または「価値なし」と確認された後は、その決定を不可逆にする——異なる抽出ラウンド間で何度も再評価することを避ける

## 8.13 ソースコードインデックス

| ファイル | 行数 | 核心的な責任 |
|------|------|---------|
| `services/compact/compact.ts` | ~1,705 | 従来の圧縮メインロジック：Fork Agent、PTL リトライ、添付ファイル再注入、部分圧縮 |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Session Memory 圧縮：calculateMessagesToKeepIndex、adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | マイクロ圧縮：時間トリガー、キャッシュ編集、token 推定 |
| `services/compact/prompt.ts` | ~374 | 圧縮プロンプト：9 章テンプレート、NO_TOOLS_PREAMBLE、formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | 自動圧縮：閾値計算、shouldAutoCompact 決定チェーン、サーキットブレーカー |
| `services/compact/apiMicrocompact.ts` | ~153 | API ネイティブのコンテキスト管理：clear_tool_uses、clear_thinking |
| `services/compact/grouping.ts` | ~63 | メッセージのグループ化：groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | 圧縮後のクリーンアップ：キャッシュのリセット、モジュール状態、分類器 |
| `services/compact/timeBasedMCConfig.ts` | ~43 | 時間トリガー設定：GrowthBook リモート設定 |
| `services/compact/compactWarningHook.ts` | ~16 | React hook：compact 警告抑制状態の購読 |
| `services/compact/compactWarningState.ts` | ~18 | 状態ストレージ：compact 警告抑制フラグ |
| `services/cost-tracker.ts` | ~323 | コスト追跡：token 課金、モデル使用統計 |
| `utils/collapseReadSearch.ts` | ~1,109 | コンテキスト折りたたみ：UI 層のメッセージグループ化と折りたたみ |
| `utils/toolResultStorage.ts` | ~1,040 | ツール結果ストレージ：ディスク永続化、per-message 予算、ContentReplacementState |
