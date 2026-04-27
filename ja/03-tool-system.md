# 第3章：ツールシステム

## 3.1 概要とポジショニング

Claude Codeのツールシステムはプロダクト全体の実行層。LLMが推論と意思決定を担当するが、実際の副作用——ファイルの読み取り、コマンドの実行、コードの検索、ネットワークアクセス——はすべてツールシステムを通じて完成される。ツールシステムはLLMの意図と現実世界の間の唯一のチャネル。

規模から見ると、これはかなり大規模なサブシステム：
- ソースコードスナップショット中に見えるツールディレクトリは**40+個のサブディレクトリ**、ファイル操作、コード実行、Agent調整、MCP統合、タスク管理等のカテゴリをカバー
- コア抽象ファイル`Tool.ts`は792行、ツール登録ファイル`tools.ts`は389行、ツール実行エンジン`services/tools/toolExecution.ts`は1,745行
- ツール結果ストレージモジュール`utils/toolResultStorage.ts`は1,040行、tokenの予算問題を独立して処理

この規模は一つの事実を示す：**ツールシステムはClaude Codeのアクセサリーではなく、コアエンジニアリング資産**。プロダクト全体の信頼性、安全性、拡張性はツールシステムの設計品質によって大部分が決まる。

競合他社の分析（cc-notebook）にはツールシステムの独立した章がなく、これは明らかな分析の盲点——本章がこの空白を埋める。

---

## 3.2 理論的基礎

### Self-Describing Tools（自己記述型ツール）パターン

従来のAPI呼び出しでは、呼び出し元がインターフェース仕様を事前に知っている必要がある。Claude Codeのツールシステムは異なる設計哲学を採用：**各ツールが自身の能力、入力形式、使用上の制約を自己記述する**。

これはToolタイプのいくつかのコアフィールドに体現されている：

```typescript
// Tool.ts:300-310（簡略化）
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // 3-10語の能力サマリー、ToolSearchのキーワードマッチング用
  description(input, options): Promise<string>   // 動的に説明を生成
  prompt(options): Promise<string>               // ツールの完全なシステムプロンプト
  inputSchema: Input           // Zodスキーマ、ドキュメントであり検証器でもある
  outputSchema?: z.ZodType
  // ...
}
```

`description()`と`prompt()`は非同期メソッドで、ツールの自己記述が**動的に生成される**ことを意味する——現在の権限コンテキスト、インストール済みのツール、環境状態に応じてプロンプト内容を調整できる。静的なドキュメントではなく、実行時に生成されるコンテキスト対応の説明。

### プラグインアーキテクチャと依存性注入

ツールシステムは本質的にプラグインアーキテクチャ。各ツールは`buildTool()`ファクトリー関数で構築され、統一された`Tool`インターフェースを実装するが、互いに完全に独立している。新しいツールの追加に必要なのは：

1. ツールディレクトリを作成（例：`tools/MyTool/`）
2. `ToolDef`インターフェースを実装
3. `tools.ts`の`getAllBaseTools()`に登録

ツール自体は互いに依存しない（循環依存はlazy requireで解消）が、すべてが`ToolUseContext`に依存する——これは権限状態、メッセージ履歴、アプリ状態等を含む実行チェーン全体を貫くコンテキストオブジェクト。

```typescript
// Tool.ts:167-172（簡略化）
export type ToolUseContext = {
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  contentReplacementState?: ContentReplacementState
  // ...
}
```

`ToolUseContext`の設計は典型的な依存性注入：ツール実行に必要なすべての外部依存はcontextを通じて渡され、ツール自体はステートレスな純関数型コンポーネント。これによりテスト、隔離、サブエージェント実行が可能になる。

### LLMでのFunction Callingの役割

Claude CodeはAnthropicのAPIのFunction Callingプロトコルに従う。LLMは推論中に`tool_use`ブロックを出力し、呼び出すツールの名前と引数を指定できる。実行結果は`tool_result`ブロックの形式でLLMに返され、次のラウンドの推論の入力となる。

このループの重要な制約：**ツール定義（名前 + 入力スキーマ）はLLMに送信するsystem promptに含める必要があり**、貴重なコンテキストのtokenを消費する。ツール数が40+に増え、MCPのサードパーティツールも加わると、このオーバーヘッドは無視できなくなる——これが3.6節で説明するToolSearch遅延読み込みメカニズムを直接生み出した。

---

## 3.3 アーキテクチャとデータ構造

### buildTool()統一抽象

`buildTool()`はツールシステムのコアファクトリー関数で、`Tool.ts:756-769`に定義されている：

```typescript
// Tool.ts:756-769
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

一つのことをする：ユーザーが提供する`ToolDef`（オプションフィールドの省略を許可）と`TOOL_DEFAULTS`（安全なデフォルト値）をマージし、完全な`Tool`を返す。

デフォルト値（`Tool.ts:729-742`）は**fail-closed（失敗安全）**の設計哲学を体現：

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // デフォルトで並行安全でない
  isReadOnly: (_input?) => false,            // デフォルトで書き込みを想定
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // デフォルトで許可、汎用権限システムで処理
  toAutoClassifierInput: (_input?) => '',    // デフォルトでセキュリティ分類器をスキップ
  userFacingName: (_input?) => '',
}
```

`isConcurrencySafe`がデフォルト`false`であることに注目——つまりシステムは2つのツールを並列実行するリスクを冒すより、直列実行を選ぶ。`isConcurrencySafe: () => true`を明示的に宣言するツール（GrepTool、GlobTool等の読み取り専用ツール）だけが並列スケジューリングされる。

### ツールのコア型定義

`Tool`インターフェースのメソッドはいくつかの機能ドメインに分けられる（`Tool.ts:297-580`）：

**実行ドメイン**
- `call(args, context, canUseTool, parentMessage, onProgress)` — コア実行メソッド、`Promise<ToolResult<Output>>`を返す
- `validateInput(input, context)` — 実行前検証、`ValidationResult`を返す
- `checkPermissions(input, context)` — 権限チェック、汎用権限システムとは独立

**説明ドメイン**（ツールの自己記述能力）
- `description(input, options)` — 短い説明、APIのtoolsリスト用
- `prompt(options)` — 完全なシステムプロンプト、モデルにツールの使い方を伝える
- `searchHint` — 3-10語の能力サマリー、ToolSearchのキーワードマッチング専用

**レンダリングドメイン**（Reactコンポーネント、REPLモードのみ）
- `renderToolUseMessage(input, options)` — ツール呼び出し開始時のUI
- `renderToolResultMessage(content, progressMessages, options)` — ツール結果のUI
- `renderToolUseProgressMessage(progressMessages, options)` — 実行中のプログレスUI
- `renderToolUseRejectedMessage(input, options)` — 拒否された時のUI

**メタデータドメイン**
- `isConcurrencySafe(input)` — 並列実行可能かどうかを宣言
- `isReadOnly(input)` — 読み取り専用かどうかを宣言（権限判断に影響）
- `isDestructive(input)` — 非可逆かどうかを宣言（削除、上書き、送信）
- `shouldDefer` — 遅延読み込みするかどうか（ToolSearchがオンデマンドで読み込む）
- `alwaysLoad` — 常にpromptに読み込む（遅延しない）
- `maxResultSizeChars` — ツール結果をディスクに永続化するトリガーの閾値

`ToolResult<T>`の構造（`Tool.ts:289-298`）も注目に値する：

```typescript
// Tool.ts:289-298
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

`contextModifier`はツール実行後にコンテキストを変更できるが、コメントに明示されている：**並行安全でないツールのみが`contextModifier`を実行する**——これは重要な並行安全上の制約。

### ツールの登録と発見メカニズム

`tools.ts`の`getAllBaseTools()`がツール登録の単一真実源（`tools.ts:108-186`）。この関数は現在の環境で利用可能なすべての内蔵ツールを返し、複数の条件でツールの可用性を制御する：

**環境条件**（process.env）：
- `USER_TYPE === 'ant'` — Anthropic内部ツール（ConfigTool、TungstenTool、REPLTool）
- `NODE_ENV === 'test'` — テストツール（TestingPermissionTool）
- `ENABLE_LSP_TOOL` — LSP統合ツール
- `CLAUDE_CODE_VERIFY_PLAN` — プラン検証ツール

**Feature Flag条件**（`bun:bundle`の`feature()`）：
- `PROACTIVE` / `KAIROS` — SleepTool（プロアクティブな動作）
- `AGENT_TRIGGERS` — ScheduleCronTool等の定時タスクツール
- `COORDINATOR_MODE` — コーディネーターモード関連ツール
- `WEB_BROWSER_TOOL` — ブラウザツール
- `WORKFLOW_SCRIPTS` — ワークフローツール

**実行時条件**：
- `isToolSearchEnabledOptimistic()` — ToolSearchToolを追加するかどうか
- `isTodoV2Enabled()` — タスク管理ツールセットを追加するかどうか
- `isAgentSwarmsEnabled()` — チーム協調ツールを追加するかどうか
- `hasEmbeddedSearchTools()` — bfs/ugrepが内蔵されていればGlobTool/GrepToolを追加しない

ツールの**重複排除と並び替え**（`assembleToolPool()`、`tools.ts:218-248`）は精巧に設計された戦略を採用：内蔵ツールとMCPツールをそれぞれ並び替えた後に連結し、内蔵ツールを接頭辞として、MCPツールを後に追加する。これはシステムプロンプトの安定性（prompt cache stability）を維持するため——Anthropicのサーバーは固定位置にキャッシュブレークポイントを設定し、内蔵ツールとMCPツールが混合ソートされると、MCPツールが追加されるたびにキャッシュが壊れる。

---

## 3.4 ツール分類一覧

`tools/`ディレクトリ構造と`tools.ts`の登録ロジックに基づいて、完全なツール一覧が整理できる：

### ファイル操作（File Operations）

| ツール | ディレクトリ | 機能 | 並行安全 |
|------|------|------|---------|
| FileReadTool | `FileReadTool/` | ファイル読み取り、PDF/画像/Notebookをサポート、ページ分割読み取り | はい |
| FileEditTool | `FileEditTool/` | 精確な文字列置換、replace_allをサポート | いいえ |
| FileWriteTool | `FileWriteTool/` | ファイルの書き込み/作成 | いいえ |
| GlobTool | `GlobTool/` | globパターンでファイルを検索 | はい |
| GrepTool | `GrepTool/` | ripgrepの正規表現コンテンツ検索 | はい |
| NotebookEditTool | `NotebookEditTool/` | Jupyter Notebookのセル編集 | いいえ |

### コード実行（Execution）

| ツール | ディレクトリ | 機能 | 備考 |
|------|------|------|------|
| BashTool | `BashTool/` | Shellコマンド実行、バックグラウンドタスク、サンドボックスをサポート | コアツール |
| PowerShellTool | `PowerShellTool/` | Windows PowerShell実行 | 条件付き有効化 |
| REPLTool | `REPLTool/` | 隔離されたVM環境でのREPL実行 | Ant内部 |

### Agent調整（Agent Orchestration）

| ツール | ディレクトリ | 機能 |
|------|------|------|
| AgentTool | `AgentTool/` | サブエージェント（subagent）を起動、並列実行をサポート |
| SendMessageTool | `SendMessageTool/` | 他のAgentにメッセージを送信 |
| TeamCreateTool | `TeamCreateTool/` | Agentチームを作成 |
| TeamDeleteTool | `TeamDeleteTool/` | Agentチームを削除 |
| TaskCreateTool | `TaskCreateTool/` | バックグラウンドタスクを作成 |
| TaskGetTool | `TaskGetTool/` | タスクの状態を取得 |
| TaskUpdateTool | `TaskUpdateTool/` | タスクの状態を更新 |
| TaskListTool | `TaskListTool/` | すべてのタスクを一覧 |
| TaskStopTool | `TaskStopTool/` | タスクを停止 |
| TaskOutputTool | `TaskOutputTool/` | タスクの出力を取得 |

### コンテキストとツール発見（Context & Discovery）

| ツール | ディレクトリ | 機能 |
|------|------|------|
| SkillTool | `SkillTool/` | Skill（~/.claude/skills/）を読み込んで実行 |
| ToolSearchTool | `ToolSearchTool/` | 遅延読み込みツールを検索 |
| MCPTool（動的生成）| `MCPTool/` | MCPサーバーツール（実行時に動的に登録） |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | MCPリソースを一覧 |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | MCPリソースを読み取り |
| LSPTool | `LSPTool/` | LSP言語サーバー統合 |

### 計画と状態（Planning & State）

| ツール | ディレクトリ | 機能 |
|------|------|------|
| EnterPlanModeTool | `EnterPlanModeTool/` | プランモードに入る（読み取り専用、実行しない） |
| ExitPlanModeTool | `ExitPlanModeTool/` | プランモードを終了 |
| EnterWorktreeTool | `EnterWorktreeTool/` | git worktree隔離環境に入る |
| ExitWorktreeTool | `ExitWorktreeTool/` | worktree環境を終了 |
| TodoWriteTool | `TodoWriteTool/` | Todoリストを書き込む（サイドバーに表示） |
| BriefTool | `BriefTool/` | セッションサマリーを生成 |

### ネットワークアクセス（Network）

| ツール | ディレクトリ | 機能 |
|------|------|------|
| WebFetchTool | `WebFetchTool/` | HTTPフェッチ、HTML→Markdown変換、ドメイン安全チェック |
| WebSearchTool | `WebSearchTool/` | Web検索 |

### システムとスケジューリング（System & Scheduling）

| ツール | ディレクトリ | 機能 | 条件 |
|------|------|------|------|
| ConfigTool | `ConfigTool/` | 設定の読み書き | Ant内部 |
| SleepTool | `SleepTool/` | 待機（プロアクティブモード） | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | 合成出力（特殊用途） | — |
| ScheduleCronTool | `ScheduleCronTool/` | 定時タスクの作成/削除/一覧 | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | リモートトリガー | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | ユーザーへの質問（インタラクティブ） | — |

---

## 3.5 ツール実行フロー

### LLMのtool_useからツール実行までの完全フロー

ツール実行のエントリーポイントは`services/tools/toolExecution.ts`の`runToolUse()`関数（`toolExecution.ts:298-428`）で、これはAsync Generatorである：

```
LLM が tool_use ブロックを出力
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — ツールを検索、エイリアスをサポート（改名されたツールの後方互換性）
    ↓
abortController.signal.aborted? → CANCEL_MESSAGEを返す
    ↓
streamedCheckPermissionsAndCallTool() [AsyncIterableを返す]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — Zod型検証
  2. tool.validateInput(input, context)  — ツール独自の検証
  3. runPreToolUseHooks()                — PreToolUse hooksを実行
  4. canUseTool()                        — 権限チェック（UI確認ダイアログが出る場合あり）
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — 大きな結果を永続化
  7. runPostToolUseHooks()               — PostToolUse hooksを実行
    ↓
MessageUpdateLazyをyield（tool_resultを含む）
    ↓
次のラウンドのLLM推論
```

重要な後方互換性設計（`toolExecution.ts:350-360`）：ツールが改名された時、古い名前が`aliases`として保持される。`options.tools`でツールが見つからない場合、システムは`getAllBaseTools()`でエイリアスマッチングを検索——古いtranscriptの古いツール名が今でも実行できることを保証。

### ストリーミングツール実行（Streaming Tool Execution）

ツール実行は`Stream<MessageUpdateLazy>`を通じてストリーミング化される（`toolExecution.ts:500-535`）：

```typescript
// toolExecution.ts:500-535（簡略化）
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // プログレスメッセージ
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // 最終結果
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

ストリーミング設計の意義：UIはツールが実行中にリアルタイムでプログレスを表示できる（例えばBashToolのリアルタイム出力、AgentToolのサブエージェントプログレス）。プログレスメッセージと最終結果が同じ`Stream`パイプラインを通じて転送され、コンシューマー側のコードが単純化される。

### 並行ツール実行

Claude CodeはLLMが単一のレスポンスで複数の`tool_use`ブロックを出力し、並列実行することをサポートする。並行の前提：**すべてのツールが`isConcurrencySafe: () => true`を宣言している**こと。

並行実行時、`contextModifier`は実行されない（`ToolResult`のコメントに「contextModifier is only honored for tools that aren't concurrency safe」とある）。これは重要な安全上の制約：グローバルコンテキストを変更する操作は並行環境で実行できない。

典型的な並行安全ツール：GrepTool、GlobTool、FileReadTool（すべて`isConcurrencySafe: () => true`を宣言）。

---

## 3.6 ToolSearch — 遅延読み込みメカニズム

### なぜToolSearchが必要か（プロンプト肥大化問題）

各ツールの定義（名前 + JSON Schema + 説明）はLLMに送信する際にtokenを消費する。ツール数が一定の閾値（実験では約40-60個のツール）を超えると問題が発生：

1. **tokenコストの上昇**：各API呼び出しに大量のツール定義が含まれる
2. **アテンションの希薄化**：LLMが数十個のツールに直面すると、各ツールへのアテンションが低下する可能性がある
3. **prompt cacheの失効リスク**：ツールリストの変化（MCPツールの動的追加等）がキャッシュを失効させる

ToolSearchの解決策は**オンデマンド読み込み**：ほとんどのツールは`shouldDefer: true`でマークされ、初期プロンプトには完全なスキーマが送信されず、検索によって発見された後にのみ読み込まれる。

### deferredツールの登録と発見

ツールは`shouldDefer`フィールドで遅延読み込みするかどうかを宣言（`Tool.ts:456-462`）：

```typescript
// Tool.ts:456-462
readonly shouldDefer?: boolean

/**
 * When true, this tool is never deferred — its full schema appears in the
 * initial prompt even when ToolSearch is enabled. For MCP tools, set via
 * `_meta['anthropic/alwaysLoad']`. Use for tools the model must see on
 * turn 1 without a ToolSearch round-trip.
 */
readonly alwaysLoad?: boolean
```

`isDeferredTool()`関数（`tools/ToolSearchTool/prompt.ts`に定義）はツールが遅延されるべきかどうかを判断：`shouldDefer: true`が設定されており`alwaysLoad: true`がないツールがdeferredとしてマークされる。

ToolSearchTool自体は**決して遅延されない**——最初のラウンドで利用可能でなければ、他のツールが発見できなくなる。

### オンデマンド読み込みの実装

ToolSearchToolの`call()`メソッド（`ToolSearchTool.ts:221-302`）は2種のクエリモードをサポート：

**直接選択モード**（`select:`プレフィックス）：
```
query: "select:NotebookEdit"     → NotebookEditToolを直接返す
query: "select:Read,Edit,Grep"   → 複数のツールをバッチ選択
```

**キーワード検索モード**：
```
query: "jupyter notebook"        → キーワードマッチング、NotebookEditTool等を返す
query: "mcp__github"             → MCPサーバープレフィックスマッチング
```

検索スコアリングアルゴリズム（`ToolSearchTool.ts:155-198`）：

```
ツール名の完全一致部分（MCP）: +12点
ツール名の完全一致部分（通常）: +10点
ツール名の部分的なキーワード含有（MCP）: +6点
ツール名の部分的なキーワード含有（通常）: +5点
ツール名の完全一致フォールバック: +3点
searchHintの語境界マッチング: +4点（精密に設計された能力サマリー、シグナルが強い）
説明テキストの語境界マッチング: +2点
```

`searchHint`フィールドの重み（+4点）が説明テキスト（+2点）より高く、ツール開発者に精確な能力サマリーを提供するよう促す。例えばGrepToolの`searchHint: 'search file contents with regex (ripgrep)'`、FileEditToolの`searchHint: 'modify file contents in place'`。

検索結果は`tool_reference`ブロックとしてLLMに返される（`ToolSearchTool.ts:330-352`）。これはAnthropicのAPIの特殊な拡張で、サーバーに「これらのツールの完全なスキーマを現在の会話のツールリストに注入してください」と伝える。

---

## 3.7 ツール結果ストレージ

### ディスクストレージ戦略

ツール実行の結果は非常に大きくなる可能性がある（例えば10MBのログファイルの読み取り、大量の出力を生成するコマンドの実行）。大きな結果をメッセージ履歴に直接入れると、tokenが浪費され、後続のリクエストのcontextが膨張する。

`utils/toolResultStorage.ts`は**オンデマンド永続化**戦略を実装：

1. 結果サイズを計算（`contentSize()`）
2. ツールの`maxResultSizeChars`閾値と比較（`getPersistenceThreshold()`で解析）
3. 閾値を超えた結果を`~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`に書き込む
4. ファイルパス + プレビューを含むメッセージに置き換え

```typescript
// toolResultStorage.ts:168-177
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  let message = `${PERSISTED_OUTPUT_TAG}\n`
  message += `Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}\n\n`
  message += `Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):\n`
  message += result.preview
  message += result.hasMore ? '\n...\n' : '\n'
  message += PERSISTED_OUTPUT_CLOSING_TAG
  return message
}
```

`PREVIEW_SIZE_BYTES = 2000`（約2KB）、プレビューは最後の改行文字まで切り取り、行の途中で切断しないようにする。

重要な**冪等性**設計（`toolResultStorage.ts:145-158`）：書き込み時に`{ flag: 'wx' }`（exclusive create）を使用し、ファイルが既に存在する場合は書き込みエラーを無視し、既存のファイルを使ってプレビューを生成する。これによりmicrocompactが履歴メッセージを再生した際に重複書き込みが発生せず、EEXISTエラーも出ない。

FileReadToolには特別な処理がある：`maxResultSizeChars: Infinity`——読み取りツールの結果はディスクに永続化されない。コメントに理由が説明されている："persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits"（永続化すると循環が発生：Readがファイルを読み、結果が大きいのでファイルに永続化され、モデルがそのファイルをReadで読もうとする……）。

### Token予算管理

`toolResultStorage.ts`はより巨視的な**メッセージレベルのツール結果予算（per-message aggregate budget）**も実装している。これは`ContentReplacementState`メカニズムによって駆動される（`toolResultStorage.ts:395-440`）：

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // 既に予算チェックを受けたtool_use_id（結果が固定）
  replacements: Map<string, string>  // 置換されたID → 置換後のコンテンツ文字列
}
```

コア制約：**結果が一度判断（置換または非置換）されたら、永遠に変わらない**（`seenIds`セットで保証）。これはprompt cacheの安定性のため——同じtool_use_idの処理結果はセッション全体を通じて一貫していなければならず、そうでないとコンテンツの変化でキャッシュが失効する。

予算制限はGrowthBook feature flagの`tengu_hawthorn_window`で動的に制御され、1つのメッセージのツール結果の総量が上限を超えた場合、システムは最も大きなツール結果をディスク永続化バージョンに置き換え、総量が予算内に収まるまで続ける。

---

## 3.8 設計上の意思決定分析

### 自己記述 vs. 外部登録のトレードオフ

Claude Codeは**自己記述**パターンを選択（各ツールが自身のスキーマ、説明、プロンプト、レンダリングロジックを持つ）、これらの情報を中央のレジストリに集中させない。

利点：
- **ツールが完全に自己完結**：新しいツールの追加に必要なのは1つのディレクトリで、中央レジストリのロジックを変更する必要がない
- **説明が動的に生成できる**：`description()`と`prompt()`は非同期関数で、環境、権限、インストール状態に基づいて動的に内容を調整できる
- **レンダリングロジックとツールが共存**：Reactレンダリングコンポーネントはツールファイルのすぐ隣にあり、ツールの動作を変更することとUIを変更することが同じPRになる

欠点：
- **ツールインターフェースの肥大化**：`Tool`型には40+のメソッド/フィールドがあり、新しいツール作者は多くのインターフェースの詳細を理解する必要がある
- **コードの重複**：各ツールが`renderToolUseMessage`、`renderToolResultMessage`等のレンダリングメソッドを持ち、パターンが非常に似ている
- **`buildTool()`では完全には解消できない**：デフォルト値を提供するが、多くのメソッドは各ツールが自分で実装する必要がある

実際、Claude Codeは**共有UIコンポーネント**（`tools/shared/`等）と**パターン抽出**（`lazySchema()`等）でコードの重複を軽減しているが、根本的なインターフェースの複雑さは依然として存在する。

### なぜ一部のツールが遅延読み込みなのか

ToolSearchの遅延読み込みの決定は一つの原則に従う：**最初のラウンドの会話で不要かもしれないツールはすべて遅延すべき**。

`alwaysLoad`のツール（絶対に遅延しない）は、モデルが最初のラウンドからその存在を知る必要があるものでなければならない。典型的な例はAgentTool、BashTool、FileReadTool——これらはどんなプログラミングタスクの基礎ツール。

`shouldDefer`のツール（遅延読み込み）は通常：特定のシナリオでのみ必要なツール（NotebookEditToolはJupyterタスクのみ必要）、大量のMCPツール（ユーザーが数十個のMCPサーバーをインストールしているが、各会話でそのうち少数しか使用しない）。

MCPツールはデフォルトでツール数に基づいてToolSearchメカニズムをトリガーするが、ツールのmetadataに`_meta['anthropic/alwaysLoad']`を設定することで強制的に遅延しないようにできる。

### ツール権限の分層設計

ツール権限は**三層防御**設計を採用：

1. **Zod型検証**（`checkPermissionsAndCallTool`の第一歩）：ツールのinputSchemaがパラメータ型を厳格に検証し、LLMが誤った型のパラメータを生成するとエラーメッセージを返して拒否
2. **ツール独自の検証**（`validateInput()`）：ツールが自身のビジネスロジック検証を実装。例えばFileEditToolはold_stringとnew_stringが異なることを確認し、ファイルサイズが1GiBを超えていないかをチェック
3. **汎用権限システム**（`canUseTool()` + `checkPermissions()`）：ユーザーが設定したallow/denyルール、ツールが読み取り専用かどうか、破壊的な操作かどうか等に基づいて最終判断し、インタラクティブな確認ダイアログが表示される場合がある

これら3層は順次実行され、どこかの層が失敗すると短絡して次の層には入らない。

---

## 3.9 移植可能なパターン

### 自己記述型ツールパターンの汎用設計

Claude Codeのツールシステムから抽出した最も移植価値の高いパターンは：**ツール = 自己完結したプラグイン**。

コア原則：
1. **スキーマ = ドキュメント = 検証器**：Zodスキーマで入力を定義し、LLM用のJSON Schemaを自動生成し、実行時にLLMの出力を検証
2. **ファクトリー関数 + 安全なデフォルト値**：`buildTool()`がフェイルセーフなデフォルト動作（デフォルトで並行安全でない、デフォルトで読み取り専用でない）を提供し、ツール開発者は自分の例外だけを宣言
3. **searchHintの精簡サマリー**：3-10語の能力説明、キーワード検索専用に最適化され、完全な説明とは分離
4. **能力宣言 > 実行時判断**：`isReadOnly()`、`isConcurrencySafe()`、`isDestructive()`により、スケジューラーはツールを実行せずにスケジューリング決定ができる

### DoramagicのツールシステムへのBrickシステムの参考

DoramagicのBrickシステム（278+積木）はClaude Codeのツールシステムと深い類似性を持つが、本質的な違いもある：

**類似点**：
- どちらも「プラグイン式」アーキテクチャ：各Brick/Toolは自己完結した機能ユニット
- どちらも説明メカニズムが必要：LLMがいつどのツール/積木を使うかを知るため
- どちらも分類体系がある：機能ドメイン別に整理

**参考にできる具体的なパターン**：

1. **`searchHint` ≈ Brickのtags**：Claude Codeは各ツールに3-10語の精簡な能力説明を提供し、検索マッチング専用。Doramagicの積木は現在tagsとcategoriesで整理されているが、`hint`フィールドを追加してモデルの積木発見効率を最適化できる。

2. **遅延読み込み → 積木のオンデマンドアクティベーション**：Claude Codeのdeferred toolsメカニズムは「すべての積木をシステムプロンプトに詰め込むのは良い考えではない」ことを示す。Doramagicは`shouldDefer`の設計を参考に、あまり使われない積木（ドメイン専用積木）を遅延読み込みに設定し、モデルが明示的に必要とした時だけアクティベートできる。

3. **`maxResultSizeChars` → 積木出力予算**：各ツールが自身の結果の最大token予算を宣言し、超えれば圧縮する。Doramagicの積木出力（抽出された知識JSON）も非常に大きくなる可能性があり、このメカニズムを参考に「サマリー優先、詳細はオンデマンド」の出力戦略を実装できる。

4. **`isConcurrencySafe` → 積木の並行宣言**：Doramagicの知識抽出パイプラインでは、複数の積木が同じコードベースで同時に作業する可能性がある。積木の並行安全性を明示的に宣言することで、スケジューラーがどの積木を並列実行できるかを自動的に決定できる。

5. **三層権限防御 → 積木実行の安全**：DoramagicがOpenClaw Skillとして実行するシナリオでは、Brick実行の合法性検証がこの三層設計を参考にできる：スキーマ検証 → ビジネス検証 → プラットフォーム権限チェック。

**本質的な違い**：Claude Codeのツールは主に**確定的な操作**（ファイルの読み取り、コマンドの実行）を対象とし、出力は精確に定義可能。Doramagicの積木は**知識抽出**を対象とし、出力はセマンティック。つまりDoramagicはClaude Codeのように積木の出力をZodスキーマで厳格に検証できない——これがDoramagicの「コードは事実を言い、AIはストーリーを語る」アーキテクチャ原則の意義：確定的なスケルトン（facts抽出）はスキーマで制約できるが、非確定的な解釈（stories生成）は制約する必要がない。

---

## 3.10 ソースコードインデックス

| ファイル | 行数 | 重要な内容 |
|------|------|---------|
| `src/Tool.ts` | 792 | `Tool`型定義、`buildTool()`、`ToolUseContext`、`ToolResult`、`TOOL_DEFAULTS` |
| `src/tools.ts` | 389 | `getAllBaseTools()`、`getTools()`、`assembleToolPool()`、`getMergedTools()`、`filterToolsByDenyRules()` |
| `src/services/tools/toolExecution.ts` | 1,745 | `runToolUse()`、`checkPermissionsAndCallTool()`、`streamedCheckPermissionsAndCallTool()`、`buildSchemaNotSentHint()` |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 471 | `searchToolsWithKeywords()`、`parseToolName()`、キーワードスコアリングアルゴリズム、`select:`プレフィックスによる直接選択 |
| `src/utils/toolResultStorage.ts` | 1,040 | `persistToolResult()`、`buildLargeToolResultMessage()`、`ContentReplacementState`、`enforceToolResultBudget()` |
| `src/tools/BashTool/BashTool.tsx` | ~1,800+ | `isSearchOrReadBashCommand()`、サンドボックス、バックグラウンドタスク、プログレス表示 |
| `src/tools/FileEditTool/FileEditTool.ts` | ~500+ | 文字列置換、大ファイル保護（1GiB制限）、シークレット検出 |
| `src/tools/FileReadTool/FileReadTool.ts` | ~600+ | マルチフォーマットサポート（PDF/画像/Notebook）、tokenカウント、`maxResultSizeChars: Infinity` |
| `src/tools/GrepTool/GrepTool.ts` | ~400+ | ripgrep統合、head_limit/offsetページ分割、`DEFAULT_HEAD_LIMIT = 250` |
| `src/tools/WebFetchTool/utils.ts` | ~450+ | ドメインブラックリストチェック、LRUキャッシュ（50MB/15分）、HTML→Markdown変換 |
| `src/tools/MCPTool/classifyForCollapse.ts` | ~350 | MCPツールのsearch/read分類（Slack/GitHub/Linear/Jira等20+サービス事業者の事前設定ルール） |

**重要な定数**（複数のファイルに散在）：
- `PREVIEW_SIZE_BYTES = 2000`（toolResultStorage.ts）— 大きな結果のプレビューサイズ
- `DEFAULT_HEAD_LIMIT = 250`（GrepTool.ts）— grepのデフォルト結果上限
- `MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024`（WebFetchTool/utils.ts）— ネットワークフェッチの10MB上限
- `FETCH_TIMEOUT_MS = 60_000`（WebFetchTool/utils.ts）— HTTPリクエストの60秒タイムアウト
- `CACHE_TTL_MS = 15 * 60 * 1000`（WebFetchTool/utils.ts）— URLキャッシュ15分
- `PROGRESS_THRESHOLD_MS = 2000`（BashTool.tsx）— 2秒を超えるとプログレスを表示
- `MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024`（FileEditTool.ts）— ファイル編集の1GiB制限
