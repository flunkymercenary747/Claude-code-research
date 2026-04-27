# 第 7 章：コマンドシステム

## 7.1 概要とポジショニング

Claude Code のコマンドシステムは、ユーザーが REPL と対話するためのコアとなるエントリポイントである。ユーザーが入力欄で `/` を入力するたびにこのシステムが動作する。コマンドシステムは 3 つの役割を担う：

1. **UI 制御層**：ターミナルインターフェースの状態を直接操作し、LLM を経由しない（`/clear`、`/theme`、`/vim` など）
2. **セッション管理層**：会話履歴、コンテキスト圧縮と復元を管理する（`/compact`、`/resume`、`/branch` など）
3. **能力拡張層**：複雑なタスクをモデルに委譲して実行し、Prompt 展開メカニズムを使う（`/review`、`/skills` など）

コマンドシステムの境界設計は明確な関心の分離を体現している：コマンドが「トリガー」を担い、ツール（Tools）が「実行」を担い、LLM が「意思決定」を担う。`/review` コマンドは git を直接呼び出さず、レビューの prompt を会話フローに注入してモデルが後続のツール呼び出しチェーンを駆動する。

---

## 7.2 理論的背景

### コマンドパターン（Command Pattern）

このシステムの設計は古典的な GoF コマンドパターンと高度に一致している：

- **Command インターフェース**：`Command` ユニオン型（`PromptCommand | LocalCommand | LocalJSXCommand`）がリクエストを統一的にカプセル化
- **ConcreteCommand**：各 `commands/<name>/index.ts` ファイルが具体的なコマンド実装
- **Invoker**：REPL の `processSlashCommand` が実行をディスパッチ
- **Receiver**：`ToolUseContext`（会話状態）、`AppState`（アプリ状態）が操作対象のオブジェクト

ただし Claude Code は古典パターンに対して 2 つの重要な拡張を行っている：

**遅延読み込み**：コマンドは `load(): Promise<Module>` で遅延読み込みを実装しており、登録時に即座にインスタンス化するのではない。これにより起動コストが最初の呼び出し時に分散される。重い依存を持つコマンド（`/insights` の 113KB HTML 描画モジュールなど）にとって特に重要である。

**型付き戻り値**：コマンドは値を返さないアクション（void）ではなく、構造化された結果（`LocalCommandResult`）を返す。上位の REPL がどのように描画するかを決定し、実行と表示の分離を実現している。

### REPL コマンド処理のデザインパターン

Claude Code が採用する REPL コマンド処理は 2 つのコア原則に従う：

**Immediate vs Queued**：コマンドオブジェクトの `immediate?: boolean` フィールドがコマンドをメッセージキューを迂回して即座に実行するかどうかを決める。`/clear`、`/exit` などの UI 操作は即座に応答する必要があり、一方 `/compact` のような API 呼び出しを伴う操作はキューに入って順序通り処理される。

**Auth-gated 可用性**：実行時の feature flag（`isEnabled()`）とは異なり、`availability` フィールドはコマンドリストのフィルタリング段階で効力を発揮し、未認証ユーザーには特定コマンドの存在さえ見えないようにする（claude.ai サブスクライバー限定のコマンドなど）。

---

## 7.3 コマンド登録メカニズム

### commands.ts の登録フロー

コマンド登録のコアロジックは `commands.ts`（754 行）に集中しており、全体として 4 つの層に分かれる：

**第 1 層：静的な組み込みコマンド**

```typescript
// commands.ts:240-310（コアの抜粋）
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... 約 60 個の組み込みコマンド
])
```

`COMMANDS` 関数がモジュールレベルの定数ではなく `memoize` にラップされているのは、一部のコマンドが登録時に設定ファイルを読み込む必要があるが、設定がモジュール初期化時にはまだ使用できないためである。

**第 2 層：Feature-flag 条件コマンド**

```typescript
// commands.ts:68-112（条件インポートの抜粋）
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

これらのコマンドは `bun:bundle` の `feature()` 関数で dead code elimination を行い、ビルド時に有効でないコマンドを直接削除する。実行時の判定ではない。

**第 3 層：内部専用コマンド**

```typescript
// commands.ts:197-222（INTERNAL_ONLY_COMMANDS）
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, ultraplan, subscribePr, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
].filter(Boolean)
```

これらのコマンドは `USER_TYPE === 'ant'`（Anthropic 社内ユーザー）かつ非デモモードのときのみ登録される。内部ツールとデバッグコマンドを隔離するメカニズムである。

**第 4 層：動的読み込みコマンド**

```typescript
// commands.ts:360-395（loadAllCommands）
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

Skills、プラグインコマンド、ワークフローコマンドの 3 者が並列で非同期に読み込まれ、優先度順に並べられる：bundled skills が最高優先度で、組み込みコマンドが最低優先度。これにより、ユーザーが定義したコマンドが同名の組み込みコマンドを上書き（shadow）できることが保証される。

### コマンドの型定義

`types/command.ts` では 3 種類の相互排他的なコマンドタイプが定義されており、`Command` ユニオン型を構成する：

```typescript
// types/command.ts（コアのユニオン型）
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| 型 | 説明 | 代表的なコマンド |
|------|------|----------|
| `PromptCommand` | prompt として展開して会話フローに注入し、モデルが実行 | `/review`, `/skills`, すべての Skill |
| `LocalCommand` | 純粋にローカルで同期実行し、テキスト結果を返す | `/compact`, `/context` |
| `LocalJSXCommand` | Ink React UI コンポーネントを描画 | `/model`, `/resume`, `/config` |

`CommandBase` は 3 者が共有する基本フィールドの集合であり、以下を含む：

```typescript
// types/command.ts（CommandBase コアフィールド）
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // 実行時 feature flag チェック
  isHidden?: boolean                      // typeahead に非表示
  argumentHint?: string                   // 引数ヒントテキスト
  whenToUse?: string                      // モデル呼び出しシナリオの説明
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // キューを迂回して即座に実行
  isSensitive?: boolean                   // 履歴から引数をマスク
}
```

### コマンドの分類（組み込み vs プラグイン vs ユーザー定義）

```
コマンドソースの階層（優先度降順）
├── bundledSkills        # Claude Code にバンドルされた組み込み Skills
├── builtinPluginSkills  # 有効な組み込みプラグインが提供する Skills
├── skillDirCommands     # ユーザーの .claude/skills/ ディレクトリ内の Skills
├── workflowCommands     # ワークフロースクリプトコマンド（feature: WORKFLOW_SCRIPTS）
├── pluginCommands       # サードパーティプラグインが登録したコマンド
├── pluginSkills         # サードパーティプラグインが提供する Skills
└── COMMANDS()           # ハードコードされた組み込みコマンド（最低優先度）
```

---

## 7.4 コマンド分類完全リスト

以下は `commands.ts` の `ls` 出力と登録リストに基づいて整理したものである。

### セッション管理類

| コマンド | 説明 |
|------|------|
| `/compact [instructions]` | 会話履歴を圧縮し、コンテキストウィンドウを解放 |
| `/resume` | 過去のセッション一覧から選択して会話を再開 |
| `/branch [title]` | 現在の会話から新しいセッションに分岐 |
| `/rewind` | 会話内の特定の履歴ノードに戻る |
| `/clear` | 現在の会話記録を消去 |
| `/session` | 現在のセッション情報を表示 |
| `/rename` | 現在のセッションをリネーム |
| `/summary` | 現在の会話のサマリーを生成（内部コマンド） |
| `/export` | 会話内容をエクスポート |
| `/copy` | 最後のメッセージをクリップボードにコピー |

### 開発ツール類

| コマンド | 説明 |
|------|------|
| `/review [PR#]` | ローカルコードレビュー（`gh pr diff` を呼び出す） |
| `/ultrareview [PR#]` | クラウドでの深いコードレビュー（10〜20 分、bughunter 駆動） |
| `/commit` | コード変更をコミット（内部コマンド） |
| `/commit-push-pr` | コミット + Push + PR 作成（内部コマンド） |
| `/diff` | 現在の git diff を表示 |
| `/init` | プロジェクトを初期化（CLAUDE.md を生成） |
| `/add-dir` | 追加の作業ディレクトリを追加 |
| `/hooks` | イベントフック設定を管理 |
| `/files` | セッションでトラッキングされているファイルを一覧表示 |
| `/pr_comments` | PR のコメントを表示 |
| `/issue` | GitHub Issue を作成/表示（内部コマンド） |
| `/autofix-pr` | PR の問題を自動修正（内部コマンド） |

### 設定類

| コマンド | 説明 |
|------|------|
| `/model [name]` | 会話モデルを切り替え（インタラクティブな選択器付き） |
| `/config` | 設定項目を表示/変更 |
| `/theme` | ターミナルテーマを切り替え |
| `/vim` | vim 入力モードを切り替え |
| `/keybindings` | ショートカットキーのバインドを管理 |
| `/permissions` | ツール権限を表示/変更 |
| `/privacy-settings` | プライバシー設定を管理 |
| `/output-style` | 出力フォーマットの好みを設定 |
| `/effort` | レスポンスのエフォートレベルを設定 |
| `/fast` | Fast モードに切り替え |
| `/plan` | Plan モードに切り替え（計画するだけで実行しない） |
| `/sandbox-toggle` | サンドボックスモードを切り替え |

### デバッグと診断類

| コマンド | 説明 |
|------|------|
| `/doctor` | 設定と環境の問題を診断 |
| `/cost` | 現在のセッションのトークン消費とコストを表示 |
| `/context` | コンテキストウィンドウの使用状況詳細を表示（カテゴリ別テーブル） |
| `/stats` | 使用統計データを表示 |
| `/usage` | API 使用量情報を表示 |
| `/insights` | 過去のセッションの使用分析レポートを生成（遅延読み込み 113KB モジュール） |
| `/heapdump` | メモリヒープスナップショットを生成（デバッグ用） |
| `/debug-tool-call` | ツール呼び出しをデバッグ（内部コマンド） |
| `/perf-issue` | パフォーマンス問題を記録（内部コマンド） |
| `/ant-trace` | Anthropic 内部トレース（内部コマンド） |

### 認証とサービス類

| コマンド | 説明 |
|------|------|
| `/login` | Claude.ai アカウントにログイン |
| `/logout` | ログアウト |
| `/upgrade` | より高いプランにアップグレード |
| `/install-github-app` | GitHub App をインストール |
| `/install-slack-app` | Slack App をインストール |
| `/ide` | IDE 統合管理 |
| `/terminalSetup` | ターミナル統合の設定 |
| `/mobile` | モバイル接続 QR コードを表示 |
| `/chrome` | Chrome 拡張機能管理 |
| `/desktop` | デスクトップアプリ管理 |

### 高度な機能類

| コマンド | 説明 |
|------|------|
| `/mcp` | MCP サーバー管理（一覧/起動/再起動） |
| `/skills` | Skills 管理（一覧/インストール/更新） |
| `/tasks` | バックグラウンドタスク管理 |
| `/agents` | サブエージェント管理 |
| `/memory` | プロジェクトメモリファイル管理（CLAUDE.md） |
| `/plan` | 計画モードに入る |
| `/thinkback` | モデルの思考プロセスを遡って表示 |
| `/thinkback-play` | 思考の遡及アニメーションを再生 |
| `/advisor` | AI アドバイザーモード |
| `/plugin` | プラグイン管理 |
| `/reload-plugins` | プラグインを再読み込み |
| `/passes` | 複数ラウンドのレビュー passes 管理 |
| `/feedback` | Anthropic にフィードバックを送信 |
| `/btw` | 付記メッセージを追加 |
| `/tag` | 会話にタグを付ける |
| `/stickers` | ステッカーを表示（イースターエッグ機能） |

Feature-flag 条件コマンド（デフォルトでは非表示）：

| コマンド | Feature Flag | 説明 |
|------|-------------|------|
| `/ultraplan` | `ULTRAPLAN` | クラウドでの超大型計画（長時間非同期） |
| `/voice` | `VOICE_MODE` | 音声入力モード |
| `/bridge` | `BRIDGE_MODE` | リモートコントロールブリッジモード |
| `/workflows` | `WORKFLOW_SCRIPTS` | スクリプトワークフローコマンド |
| `/peers` | `UDS_INBOX` | ピアセッション間通信 |
| `/fork` | `FORK_SUBAGENT` | サブエージェントを明示的に作成 |
| `/buddy` | `BUDDY` | Buddy コラボレーションモード |

---

## 7.5 コマンド実行フロー

### ユーザーが "/" を入力してからコマンドが実行されるまでの完全なパス

```
ユーザーが "/compact some instructions" を入力
        │
        ▼
    REPL 入力ハンドラー
    "/" プレフィックスを検出
        │
        ▼
    getCommands(cwd)                    ← すべてのソースからコマンドリストを集約
    findCommand("compact", commands)     ← name / aliases でコマンドを検索
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← auth タイプのゲートチェック
    isCommandEnabled(cmd)               ← feature flag / isEnabled() チェック
        │
        ├── cmd.immediate を確認        ← true: キューを迂回して即座に実行
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        LocalCommandResult を返す
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        React コンポーネントをターミナルに描画
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   ContentBlockParam[] を返す
                                   会話フローに注入 → モデルの推論をトリガー
```

### コマンド引数の解析

コマンドシステムには統一された引数解析フレームワークが組み込まれていない——これは意図的な設計選択である。各コマンドは自身の `args: string` 引数を自由に処理でき、極大の柔軟性を保っている：

- `/compact` は `args.trim()` をそのままカスタム圧縮指令として使用
- `/review` は `/^\d+$/.test(prNumber)` で PR 番号かどうかを判定
- `/model` は args がある場合 `SetModelAndClose` で直接設定し、args がない場合はインタラクティブな `ModelPickerWrapper` を描画
- `/resume` はセッション ID（UUID）、カスタムタイトル、または引数なしの場合はリスト選択器を開く

この設計は統一解析層の複雑さを回避している。その代償として各コマンドがエッジケースを自ら処理する必要がある。

### コマンド出力の描画

`LocalCommandResult` の 3 種類はそれぞれ異なる描画パスに対応する：

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // テキストメッセージとして描画
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // コンテキスト置換ロジックをトリガー
  | { type: 'skip' }                      // 何も描画しない
```

`LocalJSXCommand` は `onDone()` コールバックで結果を REPL に渡す：

```typescript
// types/command.ts（LocalJSXCommandOnDone）
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // メッセージの表示方法
    shouldQuery?: boolean                   // 直ちにモデルクエリをトリガーするか
    metaMessages?: string[]                 // モデルには見えるがユーザーには見えないメッセージを挿入
    nextInput?: string                      // 次の入力を自動入力
    submitNextInput?: boolean               // 自動的に送信するか
  },
) => void
```

`display: 'system'` はシステムメッセージスタイルで表示（グレーのイタリック）、`display: 'user'` は通常のユーザーメッセージとして表示、`display: 'skip'` は完全に非表示。

---

## 7.6 代表的なコマンドの深度分析

### /compact コマンドの実装詳細

`/compact` はコマンドシステムの中で最もロジックが複雑なコマンドの一つであり、会話履歴圧縮のコアとなる責任を担う。

**実行デシジョンツリー**（`commands/compact/compact.ts`）：

```
/compact [instructions]
    │
    ├── カスタム指令はあるか？
    │   └── 指令なし → trySessionMemoryCompaction()   ← Session Memory 圧縮を優先試行
    │                  成功した場合は直接返す。最速パス
    │
    ├── isReactiveOnlyMode() ？
    │   └── はい → compactViaReactive()               ← Reactive 圧縮パス（新アーキテクチャ）
    │               並列実行: executePreCompactHooks + getCacheSharingParams
    │               呼び出し: reactiveCompactOnPromptTooLong()
    │
    └── いいえ → 従来の圧縮パス
              microcompactMessages()                  ← まずマイクロ圧縮でトークンを削減
              compactConversation()                   ← 主圧縮（サマリー生成）
              setLastSummarizedMessageId(undefined)   ← トラッキングポインタをリセット
```

重要な設計ポイント：圧縮前に `getMessagesAfterCompactBoundary(messages)` を呼び出して、REPL が UI スクロールバックのために保持している切り捨て済みメッセージを除外しなければならない——これらのメッセージはサマリーに含まれるべきでない。

圧縮成功後のクリーンアップシーケンスは固定されている：
1. `setLastSummarizedMessageId(undefined)` — メッセージポインタをリセット
2. `suppressCompactWarning()` — 「コンテキストが間もなく枯渇する」警告を抑制
3. `getUserContext.cache.clear?.()` — ユーザーコンテキストキャッシュをクリア
4. `runPostCompactCleanup()` — 後圧縮フックをトリガー

**Reactive Compact パス**では並列最適化を活用している：

```typescript
// compact.ts:compactViaReactive（コアの並列部分）
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // 前圧縮フックを実行（子プロセスを起動する場合あり）
  getCacheSharingParams(context, messages),  // システム prompt を構築（全ツールを走査）
])
```

2 つは互いに独立しているため、並列実行により待機時間を大幅に短縮している。

### /model コマンドのモデル切り替えロジック

`/model` は `local-jsx` タイプで、React コンポーネントでインタラクティブな選択器を描画する。

**2 つの実行パス**：

- **引数あり**（`/model claude-sonnet-4-6`）：`SetModelAndClose` コンポーネントを描画し、`useEffect` 内で非同期にモデルを検証し、`onDone()` を通じて即座に完了
- **引数なし**（`/model`）：`ModelPickerWrapper` コンポーネントを描画し、完全な `ModelPicker` インタラクティブインターフェースを表示

**モデル切り替えの状態更新**：

```typescript
// model.tsx:handleSelect（コアの状態更新）
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // セッションレベルの一時的な上書きをクリア
}))
```

**モデル検証の階層**（高速から低速へ）：
1. `isModelAllowed(model)` を確認 — 組織の制限ホワイトリスト
2. `isOpus1mUnavailable(model)` を確認 — 1M コンテキスト特権チェック
3. `isKnownAlias(model)` を確認 — 既知のエイリアスは直接通過（API 検証をスキップ）
4. `validateModel(model)` — API を呼び出してカスタムモデル名を検証

Fast Mode とモデル切り替えには連動関係がある：新しいモデルが Fast Mode をサポートしない場合は自動的にオフになる；サポートしており有効な場合は確認メッセージに "Fast mode ON" と表示される。

### /review コマンドのコードレビューフロー

`/review` は `PromptCommand` タイプの典型的な使い方を示している——シンプルな prompt テンプレートで完全なレビューフローを駆動する：

```typescript
// review.ts:LOCAL_REVIEW_PROMPT（完全な prompt テンプレート）
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

コマンド自体の重要なコードはわずか 4 行で、残りはすべてモデルが行う——これが `PromptCommand` の設計哲学である：**コマンドが WHAT を定義し、モデルが HOW を決める**。

対照的なのが `/ultrareview`（`local-jsx` タイプ）で、まったく異なるパスを実行する：

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← 無料枠 / Extra Usage 残高を確認
    │   ├── Team/Enterprise → 直接通過
    │   ├── 無料回数あり → 通過、注意表示
    │   └── 残高なし → 超過確認ダイアログを表示
    │
    └── launchRemoteReview()
        ├── PR モード → teleportToRemote(branchName: "refs/pull/N/head")
        └── ブランチモード → git merge-base → git diff チェック → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → タスク URL を返し、モデルがユーザーに通知
```

`/ultrareview` はコードレビュータスクをクラウドに「テレポート」して実行し、ローカルで `RemoteAgentTask` を登録して即座に返る。ポーリングメカニズムで結果を受け取る——これはローカルコマンドの同期実行モデルとは完全に異なる非同期タスク委譲パターンである。

---

## 7.7 コマンドと Skill の境界

### 両者の異同

| 次元 | コマンド（Command） | Skill |
|------|---------------|-------|
| 定義方法 | TypeScript コード、ハードコードされたロジック | Markdown ファイル、frontmatter + prompt コンテンツ |
| 読み込みタイミング | 起動時に静的登録（組み込み）または非同期読み込み（プラグイン） | 実行時にファイルシステムからスキャン |
| 実行タイプ | `local` / `local-jsx` / `prompt` | `prompt` のみ（prompt として展開） |
| モデルからの呼び出し | ほとんどの組み込みコマンドはモデル呼び出しを禁止（`source: 'builtin'`） | SkillTool 経由でモデルが呼び出せるよう設計 |
| ユーザーへの可視性 | すべてのコマンドが `/` typeahead に表示 | `userInvocable` と `hasUserSpecifiedDescription` に依存 |
| コンテキスト認識 | `ToolUseContext` で完全なアプリ状態にアクセス | prompt コンテンツのみ使用可能、状態への直接アクセスなし |
| ソース識別子 | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### 設計選択の背景にある考慮

**なぜ組み込みコマンドは Markdown Skill を使わないのか？**

組み込みコマンドはアプリケーション状態（`AppState`）へのアクセス、Node.js API の呼び出し（ファイルシステム、暗号化）、React コンポーネントの描画が必要——これらの能力は prompt テンプレートで表現できる範囲をはるかに超えている。`/compact` は 4 つの異なる圧縮戦略を呼び出す必要があり；`/model` はインタラクティブな UI を描画する必要があり；`/resume` はセッションファイルの読み書きが必要である。これらはすべてコードでなければならない。

**SkillTool のフィルタリングロジック**が境界の正確な画定を明らかにしている：

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← 組み込みコマンドは除外される
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`** がコアルールである：組み込みコマンドはモデルが呼び出せるリストから明示的に除外される。これによりモデルが SkillTool を通じて権限チェックを迂回してセッション状態を直接操作することを防ぐ。

**リモート安全コマンドセット（REMOTE_SAFE_COMMANDS）**はさらにこの境界を細かく定義している：

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

`--remote` モードでは 20 個のコマンドのみ利用可能——これらのコマンドはローカルファイルシステム、git、IDE に依存せず、純粋な TUI の状態操作であり、リモートブリッジセッションで安全に実行できる。

---

## 7.8 設計決定の分析

**決定 1：統一インターフェースではなく 3 種類のコマンドタイプ**

`local` / `local-jsx` / `prompt` の 3 分類は複雑さを増すように見えるが、各タイプが異なるコアの問題を解決している：
- `local` は副作用があるが UI のない操作を処理（構造化データを返す必要あり）
- `local-jsx` はインタラクティブなインターフェースが必要な操作を処理（Ink 描画ツリーに依存）
- `prompt` はモデルに委譲できる操作を処理（最低の結合度）

単一インターフェースに統一した場合、すべてのコマンドが React 描画を処理する必要が生じる（不必要な依存）か、型安全性が失われるかのどちらかになる。

**決定 2：グローバルシングルトンではなく cwd でメモ化**

`loadAllCommands = memoize(async (cwd: string) => ...)` は作業ディレクトリをキャッシュキーとして使用するため、異なるディレクトリの Claude Code インスタンスは独立したコマンドキャッシュを持つ。これはモノリポや複数プロジェクトのシナリオで各ディレクトリが独立した Skills セットを持つという要件をサポートする。

**決定 3：統一引数解析を行わない**

これは意図的な「緩い設計」である。統一解析フレームワーク（commander.js など）はすべてのコマンドに完全な引数 schema の宣言を強制するが、「自由テキスト指令」型のコマンド `/compact` にはまったく意味がない。生の文字列を保持してコマンドが自分で解析方法を決めることで、一貫性を犠牲にして柔軟性を得ている。

**決定 4：Availability と isEnabled の 2 層ゲート**

2 層ゲートは異なるライフサイクルの可視性問題を解決している：
- `availability` はコマンドリスト構築時にフィルタリングし、結果がキャッシュされる。静的な auth タイプチェックに適している
- `isEnabled()` は `getCommands()` の呼び出しごとに再評価される（キャッシュなし）。動的な feature flag チェックに適している

コメントには `isEnabled()` がメモ化されない理由が特に説明されている：`/login` 実行後に auth 状態が変化し、即座にコマンドリストに反映される必要があるためである。

**決定 5：内部コマンドを別パッケージで管理しない**

`INTERNAL_ONLY_COMMANDS` は `USER_TYPE === 'ant'` 環境変数によって可視性を制御し、別の npm パッケージを使わない。これによりビルドの複雑さが軽減される。代償として外部ビルド時に dead code elimination でこの部分のコードを削除する必要がある（`filter(Boolean)` は `null` の条件コマンドにも同様に有効である）。

---

## 7.9 移植可能なパターン

### パターン 1：コマンドタイプの 3 分類

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**適用シナリオ**：「純ロジック」、「UI インタラクション」、「LLM への委譲」を同時にサポートする必要があるコマンドシステム。3 者の境界は非常に明確で、他の REPL/CLI フレームワークに直接移植できる。

**コアな価値**：型システムが責任の分離を強制し、実行時の instanceof チェックが不要になる。

### パターン 2：遅延読み込み + cwd によるメモ化

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

**適用シナリオ**：コマンド数が多い（>50）、または一部のコマンドが重いモジュールに依存する CLI ツール。

**実装のポイント**：メモ化のキーにはコマンドセットに影響するすべての要因を含める必要がある（ここでは cwd）。キャッシュの無効化タイミングは実際の状態変化に対応させる（ここでは `clearCommandsCache()`）。

### パターン 3：マルチソースコマンド集約 + 優先度順ソート

```typescript
return [
  ...bundledSkills,       // 最高優先度（同名の組み込みコマンドを上書き可）
  ...pluginCommands,
  ...COMMANDS(),          // 最低優先度（上書きされる可能性あり）
]
```

**適用シナリオ**：プラグインエコシステムをサポートする CLI ツールで、サードパーティ拡張が組み込みの動作を上書き（override）できるようにしたい場合。

**注意事項**：`findCommand` はリスト内の最初のマッチを返すため、配列の順序が優先度の順序となる。設計時に明確に記録しておく必要がある。

### パターン 4：Auth-gated コマンド可視性

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai': if (isClaudeAISubscriber()) return true; break
      case 'console':   if (!isUsing3PServices() && ...) return true; break
    }
  }
  return false
}
```

**適用シナリオ**：SaaS 製品で異なるサブスクリプション層のユーザーに異なる機能セットを表示する必要がある場合。

**重要な設計**：実行段階でエラーを返すのではなく、コマンドリストのフィルタリング段階でインターセプトする——ユーザーは使えないコマンドを見ることがなく、認知負担を軽減する。

### パターン 5：Bridge Safe / Remote Safe ホワイトリスト

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([session, exit, clear, ...])
```

**適用シナリオ**：制限された環境（リモートセッション、サンドボックス、モバイルブリッジ）で実行する必要があるコマンドシステム。

**実装の考え方**：ブラックリストよりも安全——新しいコマンドはデフォルトで制限環境では使用不可であり、明示的にホワイトリストに追加する必要がある。これにより不注意から機密コマンドが誤った環境で露出するリスクを回避できる。

---

## 7.10 ソースコードインデックス

| ファイルパス | 行数 | 内容 |
|---------|------|------|
| `src/commands.ts` | 754 | コマンド登録、集約、フィルタリング、検索の全エントリポイントロジック |
| `src/types/command.ts` | 約 250 | Command ユニオン型定義、CommandBase、各サブタイプの詳細宣言 |
| `src/commands/compact/compact.ts` | 287 | /compact の 3 パス実装（session memory / reactive / traditional） |
| `src/commands/model/model.tsx` | 296 | /model インタラクティブ選択器 + 直接設定の 2 パス、React Compiler コンパイル出力 |
| `src/commands/review.ts` | 約 50 | /review（prompt タイプ）と /ultrareview（local-jsx タイプ）のエントリポイント |
| `src/commands/review/reviewRemote.ts` | 316 | /ultrareview のリモート起動ロジック：teleport、overage gate、タスク登録 |
| `src/commands/resume/resume.tsx` | 274 | /resume セッションリスト選択器 UI |
| `src/commands/branch/branch.ts` | 296 | /branch 会話の分岐：JSONL コピー、sessionId 書き換え、競合処理 |
| `src/commands/context/context-noninteractive.ts` | 325 | /context 非インタラクティブパス：カテゴリ別トークン統計、Markdown テーブル描画 |
| `src/skills/loadSkillsDir.ts` | — | Skills ディレクトリスキャンと動的読み込みロジック |
| `src/skills/bundledSkills.ts` | — | 製品にバンドルされた組み込み Skills の登録 |
| `src/plugins/builtinPlugins.ts` | — | 組み込みプラグインの Skill コマンド抽出 |
| `src/utils/plugins/loadPluginCommands.ts` | — | サードパーティプラグインコマンドの読み込みとキャッシュ |

**主要関数インデックス**：

| 関数 | ファイル | 用途 |
|------|------|------|
| `getCommands(cwd)` | commands.ts | 現在のユーザーが使用できるすべてのコマンドを返す（メインエントリポイント） |
| `findCommand(name, commands)` | commands.ts | 名前/エイリアスでコマンドを検索 |
| `meetsAvailabilityRequirement(cmd)` | commands.ts | auth タイプのゲートチェック |
| `getSkillToolCommands(cwd)` | commands.ts | モデルが呼び出せる Skill コマンドセットを返す |
| `getSlashCommandToolSkills(cwd)` | commands.ts | ユーザーが / でトリガーできる Skill セットを返す |
| `isBridgeSafeCommand(cmd)` | commands.ts | コマンドが bridge モードで実行可能か判定 |
| `formatDescriptionWithSource(cmd)` | commands.ts | ユーザーインターフェースでソース注釈付きの説明フォーマット |
