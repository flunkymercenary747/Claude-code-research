# 第4章：Agent編成とマルチエージェントアーキテクチャ

## 4.1 概要とポジショニング

Claude Codeのマルチエージェントシステムはプロダクトアーキテクチャ全体で最も複雑なサブシステムで、約8,700行のコアコードが12個の重要モジュールにまたがる。このシステムは根本的なエンジニアリング上の問題を解決する：**シングルスレッドのREPLアプリが、複数のLLM Agentの並行実行を安全かつ効率的に編成するにはどうすればよいか**。

システムは3種の段階的な協調モードを提供する：

| モード | トリガー方法 | 並行度 | 通信メカニズム | 隔離レベル |
|------|---------|--------|---------|---------|
| **Subagent（デフォルト）** | AgentTool呼び出し | 同期/非同期 | 関数の戻り値 | プロセス内AsyncGenerator |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | 完全非同期 | `<task-notification>` XML | 独立AbortController |
| **Team Mode** | `spawnTeammate()` + TeamFile | 永続化並列 | ファイルメールボックス + ポーリング | Tmux Pane / InProcess / Remote |

これら3つのモードは独立した実装ではなく、同一の`runAgent()`コアエンジン（`runAgent.ts`）を共有し、パラメータの組み合わせで異なる動作特性を実現——これがシステム全体で最もエレガントな設計上の意思決定の一つ。

**ソースコード規模統計：**

| ファイル | 行数 | 責務 |
|------|------|------|
| `AgentTool.tsx` | 1,397 | 統合エントリーポイント、ルーティング決定、ライフサイクル管理 |
| `runAgent.ts` | 973 | Agent実行エンジン、query()ループ |
| `loadAgentsDir.ts` | 755 | Agent定義の解析（Markdown/JSON/Plugin） |
| `agentToolUtils.ts` | 686 | ツールフィルタリング、権限、結果のシリアライズ |
| `UI.tsx` | 871 | Agentプログレスと結果のレンダリング |
| `coordinatorMode.ts` | 369 | Coordinatorシステムプロンプトとコンテキスト |
| `SendMessageTool.ts` | 917 | 5経路メッセージルーティング |
| `spawnMultiAgent.ts` | 1,093 | Teammate生成（Tmux/InProcess） |
| `inProcessRunner.ts` | 1,552 | InProcessバックエンドの完全実装 |
| `teammateMailbox.ts` | 1,183 | ファイルメールボックスプロトコル |
| `worktree.ts` | 1,519 | Git Worktree隔離 |

## 4.2 理論的基礎

### 4.2.1 ActorモデルとAgent編成の関係

Claude Codeのマルチエージェントアーキテクチャは、LLM編成領域におけるActorモデルの実用的な変体。古典的なActorモデル（Hewitt、1973）の3つのコアプリミティブ——**メッセージの受信、新しいActorの作成、メッセージの送信**——はコード内に明確な対応がある：

| Actorプリミティブ | Claude Code実装 | ソースの位置 |
|-----------|-----------------|---------|
| メッセージの受信 | `waitForNextPromptOrShutdown()`ポーリングループ | `inProcessRunner.ts:689-868` |
| Actorの作成 | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| メッセージの送信 | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

ただし純粋なActorモデルから2つの重要な逸脱がある：

1. **非対称な階層**：LeaderはグローバルなビューがあるAppState、WorkerはToolUseContextのみ。これは対等なActorではなく、明確なLeader-Worker階層の監督ツリー（supervision tree）。
2. **共有状態チャネル**：InProcessバックエンドのTeammateは`setAppStateForTasks`を通じてルートのAppState storeを共有（`runAgent.ts:336-337`）し、純粋なメッセージパッシングではない。これはActorモデルへの実用的な妥協——シングルプロセス内では共有状態の方がシリアライズされたメッセージより効率的。

### 4.2.2 メッセージパッシング vs. 共有メモリの並行モデル

システムは2つの並行モデルを同時に使用し、隔離レベルに応じて選択する：

**メッセージパッシングモデル**（Team Mode - Tmux Paneバックエンド）：
```
Leader → writeToMailbox("worker-1", {...}) → ファイルシステム → readMailbox() → Worker
```
通信はJSONファイル + ファイルロックで実装され、`teammateMailbox.ts`の`LOCK_OPTIONS`が指数バックオフリトライ（10回リトライ、5-100ms）を設定して並行書き込みをシリアライズする：

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**共有メモリモデル**（InProcessバックエンド）：
```
Leader → setAppState(prev => {...}) → 同一AppState store ← getAppState() ← Worker
```
InProcess Teammateは`toolUseContext.setAppStateForTasks`を通じてルートstoreを直接読み書きする。レースコンディションはReactスタイルの`setAppState(prev => {...})`関数型更新セマンティクスで回避される（底層はReactではないが、同様のCASパターンを採用）。

### 4.2.3 分散システムのCoordinatorパターン

Coordinator Modeの設計は分散システムの古典的なCoordinatorパターン（Master-Workerとも呼ばれる）をマッピングしているが、独自の制約を追加：**Coordinator自体がLLM Agentで、その「調整ロジック」はハードコードされておらず、system promptでプログラムされる**。

`coordinatorMode.ts:126-369`の`getCoordinatorSystemPrompt()`関数は約5,000文字の構造化promptを返し、完全なWorkerスケジューリング戦略を含む：

```typescript
// coordinatorMode.ts:161-167 — 重要なスケジューリングルール
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

「promptで調整ロジックをプログラムする」このパターンは、Coordinatorの動作をpromptの修正で調整できることを意味する——研究→統合→実装→検証の四段階ワークフローはコードで強制されるのではなく、LLMの命令遵守能力で実現される。これは従来の分散Coordinatorのハードコードされたスケジューリングロジックとは鮮明な対比をなす。

## 4.3 アーキテクチャとデータ構造

### 4.3.1 全体アーキテクチャ図（Leader-Worker）

```
                    ┌─────────────────────────────────────────┐
                    │           Human User (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ ユーザー入力
                    ┌──────────────▼──────────────────────────┐
                    │         Main REPL (query() loop)         │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — ルーティング   │     │
                    │    └──┬─────────┬─────────┬───────┘     │
                    │       │         │         │              │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐     │
                    │  │ Sync   │ │ Async  │ │Teammate │     │
                    │  │Agent   │ │Agent   │ │Spawn    │     │
                    │  │(block) │ │(fire&  │ │         │     │
                    │  │        │ │forget) │ │         │     │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘     │
                    │      │          │         │              │
                    │      └────┬─────┘    ┌────▼──────────┐  │
                    │           │          │  spawnMulti-   │  │
                    │      ┌────▼────┐     │  Agent.ts      │  │
                    │      │runAgent │     └────┬───────────┘  │
                    │      │  .ts    │          │              │
                    │      │         │     ┌────▼──────────┐  │
                    │      │ query() │     │  3バックエンド: │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    通信層：
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → 親が収集             │
    │  Async Agent:   <task-notification> XML → ユーザーメッセージ│
    │  Teammate:      ファイルメールボックス (.claude/teams/*/inboxes/)│
    │  InProcess:     AppState共有 + mailbox fallback      │
    │  Remote (ant):  teleportToRemote() → CCRセッション   │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 AgentDefinition型システム

Agent定義は三層のユニオン型設計を採用：

```typescript
// loadAgentsDir.ts — コア型階層

// 基本型：すべてのAgentが共有するフィールド
type BaseAgentDefinition = {
  agentType: string              // ルーティングキー（例："Explore"、"worker"）
  whenToUse: string              // LLMがAgentを選択する根拠
  tools?: string[]               // ホワイトリスト（undefined = すべて）
  disallowedTools?: string[]     // ブラックリスト
  model?: string                 // 'inherit' | 具体的なモデル名
  effort?: EffortValue           // 推論の努力レベル
  permissionMode?: PermissionMode // 権限の継承戦略
  maxTurns?: number              // 最大会話ターン数
  background?: boolean           // 常にバックグラウンドで実行
  isolation?: 'worktree' | 'remote' // 隔離モード
  memory?: AgentMemoryScope      // 永続メモリ
  omitClaudeMd?: boolean         // CLAUDE.mdを省略（週~5-15 Gtokを節約）
  // ...
}

// Built-in Agent：動的prompt、静的なsystemPromptなし
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent：Markdown/JSONから読み込む
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent：プラグインシステムから
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// 最終ユニオン型
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

この設計の巧妙さは`getSystemPrompt`メソッドにある：Built-in Agentは`toolUseContext`パラメータを受け取り（現在のツールセットに基づいて動的にpromptを調整できる）、Custom/Plugin AgentはMarkdownファイルの内容をクロージャで取得する。つまり：

- **Built-in AgentのpromptはDynamic**：各呼び出しで異なる可能性がある
- **Custom AgentのpromptはStatic**：Markdownファイルで定義されるが、`memory`が有効な場合は実行時にメモリ内容が追加される（`loadAgentsDir.ts:335-340`）

Agent定義の読み込み優先度はオーバーライドチェーンに従う：`builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`、`getActiveAgentsFromList()`でMapを使って後から読み込んだものが上書き（`loadAgentsDir.ts:169-186`）。

### 4.3.3 3種の実行バックエンドの統一抽象

3種のバックエンドは同一の`runAgent()` AsyncGeneratorインターフェースを共有するが、プロセスモデルと通信メカニズムは全く異なる：

| 次元 | Tmux Pane | InProcess | Remote（ant専用） |
|------|-----------|-----------|-------------------|
| **プロセスモデル** | 独立Claude CLIプロセス | 同プロセスのAsyncLocalStorage隔離 | CCRリモートセッション |
| **起動方法** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **通信** | ファイルメールボックスポーリング（500ms） | 共有AppState + ファイルメールボックスフォールバック | HTTP API |
| **権限** | 独立権限コンテキスト | Leader UIキューブリッジ | リモートで独立 |
| **リソースコスト** | 高（完全なプロセス） | 低（共有V8ヒープ） | 極めて高（リモートインスタンス） |
| **生存期** | Leaderとは独立 | Leaderプロセスに束縛 | 独立 |
| **検出ロジック** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

バックエンドの検出とフォールバックは`spawnMultiAgent.ts:339-375`にエレガントなフォールバックチェーンが実装されている：

```
iTerm2 (it2バックエンド) → Tmux → InProcessフォールバック
```

iTerm2が検出されているがit2 CLIがインストールされていない場合、システムはインタラクティブなsetupプロンプト（`It2SetupPrompt`）を表示し、ユーザーにit2のインストールまたはTmuxへのフォールバックを選択させる。

### 4.3.4 通信プロトコルのデータ構造

**ファイルメールボックスのメッセージ形式**（`teammateMailbox.ts:42-49`）：

```typescript
type TeammateMessage = {
  from: string       // 送信者名
  text: string       // メッセージ内容（プレーンテキストまたはJSONの構造化メッセージ）
  timestamp: string  // ISOタイムスタンプ
  read: boolean      // 既読フラグ
  color?: string     // 送信者の色識別子
  summary?: string   // UIプレビューサマリー（5-10語）
}
```

メールボックスのパスは固定形式：`~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**構造化メッセージタイプ**（`text`フィールドのJSONエンコードで伝達）：

| メッセージタイプ | 方向 | 用途 |
|---------|------|------|
| `shutdown_request` | Leader → Worker | シャットダウンのリクエスト |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | シャットダウンの応答 |
| `idle_notification` | Worker → Leader | アイドル通知 |
| `permission_request` | Worker → Leader | 権限リクエスト |
| `permission_response` | Leader → Worker | 権限応答 |
| `plan_approval_request` | Worker → Leader | Plan Modeの承認リクエスト |
| `plan_approval_response` | Leader → Worker | 承認応答 |
| `sandbox_permission_request` / `_response` | 双方向 | ネットワークサンドボックスの権限 |
| `task_assignment` | Leader → Worker | タスクの割り当て |
| `team_permission_update` | Leader → Workers | 権限のブロードキャスト |

## 4.4 コアアルゴリズムとフロー

### 4.4.1 AgentToolルーティング決定ツリー（完全版）

`AgentTool.call()`はシステムの統合エントリーポイントで、ルーティングロジックは`AgentTool.tsx:238-764`に実装されている。完全な決定ツリー：

```
AgentTool.call(input) エントリー
│
├─ [1] team nameとnameパラメータの両方が存在？
│   ├─ YES: TeammateがネストされたTeammateを生成しようとしているか？
│   │   ├─ YES: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NO: → spawnTeammate() (return teammate_spawned)
│   └─ NO: 続行
│
├─ [2] effectiveType (subagent_type)を解析
│   ├─ 明示的に指定 → 指定値を使用
│   ├─ 未指定 + Fork Gate ON → undefined (Forkパス)
│   └─ 未指定 + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (Forkパス)
│   ├─ YES: 再帰的なForkチェック
│   │   ├─ Forkサブプロセス内にある → throw
│   │   └─ 通過 → selectedAgent = FORK_AGENT
│   └─ NO: activeAgentsから検索
│       ├─ 見つかる → selectedAgent = found
│       ├─ permission denyされている → throw（deny ruleの情報付き）
│       └─ 存在しない → throw（利用可能なagentsを列挙）
│
├─ [4] effectiveIsolationを解析
│   ├─ 'remote'（ant専用） → teleportToRemote() → return remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → 後続ステップでworktreePathを使用
│
├─ [5] system promptとpromptメッセージを構築
│   ├─ Forkパス: 親promptを継承 + buildForkedMessages()
│   └─ 通常: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] shouldRunAsync判定
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — ただしisBackgroundTasksDisabledの場合は除外
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → return { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → while(true)ループに入る
│       ├─ Race: nextMessage vs backgroundSignal
│       │   ├─ backgroundが勝つ → 非同期実行に切り替え（wasBackgrounded=true）
│       │   └─ messageが勝つ → messageをyield、プログレスを追跡
│       └─ ループ終了 → finalizeAgentTool() → return AgentToolResult
```

### 4.4.2 runAgent() AsyncGenerator実行フロー

`runAgent()`はマルチエージェントシステム全体のコアエンジン（`runAgent.ts:247-860`）で、`AsyncGenerator<Message, void>`——各Messageをyieldするたびに呼び出し元がそれを処理できる（記録、表示、またはバックグラウンドキューに追加）。

**実行フローの重要なフェーズ：**

1. **ツール解析**：`resolveAgentTools()`がAgent定義の`tools`ホワイトリストを実際のToolオブジェクトに解析し、同時に`disallowedTools`ブラックリストを適用（`runAgent.ts:500-502`）

2. **System Prompt構築**：`override?.systemPrompt`または`getAgentSystemPrompt()`で構築。Explore/Plan Agentは`claudeMd`と`gitStatus`をスキップし、フリート全体で週~5-15 Gtokを節約（`runAgent.ts:389-409`）

3. **AbortController戦略**（`runAgent.ts:524-528`）：
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // 外部制御（asyncパス）
     : isAsync
       ? new AbortController()      // 非同期：独立controller
       : toolUseContext.abortController  // 同期：親controllerを共有
   ```

4. **権限のオーバーライド**（`runAgent.ts:414-497`）：Agentの`permissionMode`は親のモードをオーバーライドするが、`bypassPermissions`、`acceptEdits`、`auto`の3つの親モードは常に優先——管理者が設定したセキュリティポリシーがサブAgentによってダウングレードされないことを保証。

5. **コアループ**——`query()`を直接呼び出してyield（`runAgent.ts:748-806`）：
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... stream_event、attachment、記録可能なメッセージを処理
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **クリーンアップのfinallyブロック**（`runAgent.ts:816-858`）：MCPクリーンアップ、セッションhooksクリーンアップ、prompt cacheトラッキング、ファイル状態キャッシュ解放、Perfetto登録解除、AppState todosクリーンアップ、バックグラウンドbashタスクのkill——合計9個のクリーンアップ操作、リソースリークを防止。

### 4.4.3 非同期Agentのライフサイクル（fire-and-forget）

非同期Agentの完全なライフサイクルは`runAsyncAgentLifecycle()`（`agentToolUtils.ts:322-497`）によって駆動される：

```
registerAsyncAgent() → AppStateにタスクを登録
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — すべてのメッセージを収集
   │   ├─ agentMessages.push(message)
   │   ├─ task.retainがtrueの場合 → AppState.tasks[taskId].messagesに追加
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — SDKプログレスイベント
   │
   ├─ finalizeAgentTool() — 最終結果を抽出
   │
   ├─ completeAsyncAgent() — 完了としてマーク（最初に、遅い操作の前に）
   │   │                      ↑ 重要な設計：gh-20236修正
   │   │                        classifyHandoffとworktree cleanupがhangする可能性
   │   │                        状態遷移をブロックできない
   │
   ├─ classifyHandoffIfNeeded() — セキュリティ分類器チェック（オプション）
   │
   ├─ getWorktreeResult() — worktreeクリーンアップ
   │
   └─ enqueueAgentNotification() — <task-notification> XMLで親に通知
```

**gh-20236修正**は記録に値する設計上の決定：`completeAsyncAgent()`は`classifyHandoffIfNeeded()`と`getWorktreeResult()`の前に呼び出される。コメントが明確に理由を説明：

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 ツールフィルタリングと権限の継承

ツールフィルタリングは三層フィルタリングチェーン（`agentToolUtils.ts:66-115`）：

```
Layer 1: ALL_AGENT_DISALLOWED_TOOLS — すべてのAgentが使用禁止のツール
Layer 2: CUSTOM_AGENT_DISALLOWED_TOOLS — Custom Agentのみ追加禁止のツール
Layer 3: ASYNC_AGENT_ALLOWED_TOOLS — 非同期Agentのホワイトリスト（反転ロジック）
```

特例：
- MCPツール（`mcp__`プレフィックス）は常に許可
- `ExitPlanMode`はPlan Mode下で常に許可
- InProcess TeammateはAgent Swarmsモードで`AgentTool`（同期サブAgentを生成）とTaskツール（共有タスクリストで調整）を使用できる

ツール解析はワイルドカード（`'*'`または`undefined` = 全ツール）とAgent限定制限（`AgentTool(worker, researcher)`構文、`agentToolUtils.ts:165-172`）もサポートする。

### 4.4.5 Coordinator Modeの四段階ワークフロー

Coordinator Modeのコアロジックは`coordinatorMode.ts:126-369`の`getCoordinatorSystemPrompt()`でpromptとして定義されている。すべてのタスクを四段階に分解する：

**Phase 1: Research**（Workerが並列実行）
- 複数のWorkerが同時にコードベースを探索
- 重要なprompt指令：*"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Phase 2: Synthesis**（Coordinator自身が実施）
- これが最も重要なフェーズ——Coordinatorは研究結果を自分で読んで理解しなければならない
- 明示的に禁止されるアンチパターン：*"Never write 'based on your findings'"*
- 具体的なファイルパス、行番号、変更内容を含む実装仕様の産出が必要

**Phase 3: Implementation**（Workerが実行）
- Coordinatorはcontinueするかどうかを決定（`SendMessageTool`）またはfresh spawn（`AgentTool`）
- 決定基準はコンテキストの重複度（promptに完全な決定表がある）

**Phase 4: Verification**（独立したWorker）
- 独立した検証を明示的に要求：*"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- 検証基準：*"proving the code works, not confirming it exists"*

### 4.4.6 Team Modeの永続化協働

Team ModeはTeamFile（`.claude/teams/{team_name}/team.json`）に基づいて永続化されたチーム状態を実装する。Coordinator Modeのfire-and-forget Workerとは異なり、TeammateはLong-running processである：

1. **生成**：`spawnTeammate()`がTmux paneまたはInProcessタスクを作成
2. **実行**：Teammateがpromptを実行 → 完了 → `idle_notification`を送信 → 次のpromptを待機
3. **通信**：すべてのメッセージはファイルメールボックス経由（どのバックエンドでもファイルシステムで通信）
4. **シャットダウン**：Leaderが`shutdown_request`を送信 → TeammateのLLMがapproveまたはrejectを決定

InProcess Runnerのメインループ（`inProcessRunner.ts:883-1464`）が完全な永続化セマンティクスを実装：

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. 現在のpromptを実行（runAgent()を呼び出し）
  // 2. アイドルとしてマーク
  // 3. idle_notificationをLeaderに送信
  // 4. waitForNextPromptOrShutdown() — メールボックスをポーリング
  //    ├─ shutdown_request → LLMに決定させる
  //    ├─ new_message → 次のラウンドのpromptに設定
  //    └─ aborted → shouldExit = true
}
```

メッセージ優先度戦略（`inProcessRunner.ts:760-804`）：
1. 最高優先度：`shutdown_request`（Leaderのシャットダウン指令は埋もれない）
2. 次に：`team-lead`からのメッセージ（LeaderはユーザーIntent）
3. 最後に：FIFOキューのpeerメッセージ

### 4.4.7 ファイルメールボックス通信プロトコル

ファイルメールボックスはすべてのバックエンドの通信基盤。その設計は**パフォーマンスよりシンプルさ**を選択した：

**書き込みプロトコル**（`teammateMailbox.ts:133-191`）：
```
1. ensureInboxDir() — ディレクトリが存在することを確認
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — アトミックな作成（存在しない場合）
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — ファイルロックを取得
4. readMailbox() — ロック内で再読み取り（ダーティリードを避ける）
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — 書き戻し
7. release() — ロックを解放
```

**読み取りプロトコル**（`teammateMailbox.ts:83-107`）：
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. TeammateMessage[]を返す
```

読み取りは**ロックなし**——これは意図的な設計。読み取り側には最終一貫性で十分で、書き込み側が`lockfile`でアトミック性を保証する。

### 4.4.8 SendMessageの5経路ルーティング

`SendMessageTool.call()`は5つの独立したメッセージルーティングパスを実装（`SendMessageTool.ts`）：

```
input.toの値
│
├─ [ルート1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — クロスマシンRemote Control
│   （安全チェックが必要：クロスマシンメッセージはユーザーの明示的な同意が必要）
│
├─ [ルート2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — ローカルUnix Domain Socket
│
├─ [ルート3] agentNameRegistryまたはtoAgentIdが一致
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task stopped/evicted → resumeAgentBackground()
│       （ディスクのtranscriptから停止したAgentを自動的に復元）
│
├─ [ルート4] to === '*'
│   → handleBroadcast() — TeamFile.membersを遍歴して各メールボックスに書き込む
│
└─ [ルート5] その他
    ├─ プレーンテキスト → handleMessage() — メールボックスに書き込む
    └─ 構造化メッセージ → 対応するhandlerに分発：
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

ルート3の**自動回復**メカニズムは特に巧妙：停止したAgentにメッセージを送信すると、システムが自動的にディスクのtranscriptからそれを復元してバックグラウンドで実行する。これにより、Coordinatorは`SendMessage`を通じてかつて完了したWorkerをシームレスに継続でき、まだ実行中かどうかを気にする必要がない。

## 4.5 設計上の意思決定分析

### 4.5.1 なぜIPCではなくファイルメールボックスを選んだか

ファイルメールボックスは「原始的」な選択に見える——なぜUnix Domain Socket、Named Pipe、またはgRPCを使わないのか？

**コアの理由：バックエンド非依存性**。ファイルシステムは3種のバックエンド（Tmux、InProcess、Remote）の最大公約数：
- Tmux Paneは独立プロセスで、共有メモリがない
- InProcessは同一プロセスだがAsyncLocalStorage隔離を使用
- Remoteはネットワーク越しだが、ネットワークファイルシステムを共有できる

ファイルメールボックスの追加的な利点：
1. **可観測性**：`cat ~/.claude/teams/*/inboxes/*.json`を直接実行してデバッグできる
2. **永続性**：プロセスクラッシュ後もメッセージが失われない
3. **シンプルさ**：複雑な接続管理、ハートビート、再接続が不要
4. **並行安全性**：`proper-lockfile`が提供するファイルロックで十分な信頼性

代償は**レイテンシ**：500msのポーリング間隔は最悪の場合500msのメッセージ配信遅延を意味する。しかしLLM Agentシナリオでは、各ツール呼び出し自体が数秒かかるため、500msは無視できる。

### 4.5.2 InProcess vs. Paneバックエンドのトレードオフ

| 次元 | InProcess | Tmux Pane |
|------|-----------|-----------|
| **メモリ** | 共有V8ヒープ（低） | 独立プロセスヒープ（高） |
| **起動レイテンシ** | ~0ms | ~2-3s（CLI起動） |
| **隔離** | AsyncLocalStorage（弱） | OSプロセス（強） |
| **権限** | Leader UIブリッジ（リアルタイム） | メールボックスポーリング（遅延） |
| **デバッグ** | 共有ログ（複雑） | 独立terminal（直感的） |
| **生存期** | Leaderに束縛 | 独立 |

InProcessバックエンドの最大の利点は**権限ブリッジ**——`getLeaderToolUseConfirmQueue()`によりWorkerの権限ダイアログがLeaderのterminalに直接表示され、Worker badgeが付く。つまりユーザーはWorkerのterminalに切り替えて権限を承認する必要がない。

ただしInProcessには根本的な制限がある：**WorkerはバックグラウンドAgentを生成できない**（`AgentTool.tsx:277-278`）、なぜならそのライフサイクルがLeaderプロセスに束縛されており、バックグラウンドAgentには独立したAbortControllerが必要だから。

### 4.5.3 権限は常に人間が制御するという設計哲学

マルチエージェントシステム全体の権限設計は妥協できない原則に従う：**人間が常に最終的な権限付与者**。

この原則がコードに体現されている：
1. **サブAgentは権限をエスカレートできない**：`runAgent.ts:419` — `bypassPermissions`、`acceptEdits`、`auto`モードの親設定はサブAgentの`permissionMode`より常に優先
2. **LeaderのpermissionはWorkerに漏れない**：`runAgent.ts:467-477` — `allowedTools`が指定された場合、セッションレベルのallow rulesをクリアし、CLIアーキテクチャレベルのrulesだけを保持
3. **クロスマシンメッセージには明示的な同意が必要**：`SendMessageTool.ts:checkPermissions` — `bridge:`アドレスへの送信には`safetyCheck`が必要で、`classifierApprovable: false`（セキュリティ分類器は自動承認できない）
4. **Plan Mode承認**：Teammateに`plan_mode_required`を設定でき、この場合Leaderにplanをapproveしてもらってから実行する必要がある

### 4.5.4 query()ループ再利用の再帰設計

`runAgent()`のコアは`query()`関数の呼び出し——この`query()`はメインREPLループが使用するのと同じ関数。つまり**サブAgentとメインAgentは完全に同じAPI呼び出しとツール実行パイプラインを使用する**。

```typescript
// runAgent.ts:748-757 — Agentのquery()呼び出し
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```

この設計の深遠な影響：
- **ツールの一貫性**：AgentはユーザーとまったくAIツールを使用する（フィルタリングされているだけ）
- **再帰能力**：Agentのツールプールに`AgentTool`を含めることができ、AgentがサブAgentを生成できる（InProcess Teammateは同期サブAgentを生成することを許可されている）
- **Prompt Cacheの再利用**：Forkパスは`useExactTools`でサブAgentのAPIリクエストのプレフィックスが親Agentと一致することを保証し、prompt cacheのヒット率を最大化

しかし再帰にはリスクもある——無限再帰fork。解決策はダブルチェック（`AgentTool.tsx:331-333`）：
1. `querySource === 'agent:builtin:fork'` — コンパイル時の耐久性（context.optionsはautocompactの影響を受けない）
2. `isInForkChild(messages)` — メッセージスキャンのフォールバック

## 4.6 移植可能なパターン

### 4.6.1 Agent編成の汎用パターン

Claude Codeの実装から5つの汎用Agent編成パターンが抽出できる：

**パターン1：AsyncGeneratorをAgentインターフェースとして**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGeneratorはpull-based（引き取り型）のメッセージフローセマンティクスを提供——呼び出し元が次のメッセージをいつ消費するかを決定し、background切り替え（yield点でraceを挿入）とプログレス追跡を自然にサポートする。

**パターン2：Foreground → Backgroundのシームレスな切り替え**

Claude CodeのSync Agentは実行中にbackgroundにできる——`Promise.race([nextMessage, backgroundSignal])`で。このパターンは「長タスクを途中でバックグラウンド化できる」任意のシナリオに適用可能。重要なのは、foregroundとbackgroundの間で安定したtaskIdを持つこと。

**パターン3：ファイルシステムをAgent間通信の「最小公倍数」として**

複数のバックエンド（プロセス内/クロスプロセス/クロスマシン）が統一した通信を必要とする場合、ファイルシステムが最もシンプルな選択。JSON + ファイルロックで十分な一貫性保証が得られる。

**パターン4：Prompt-Programmed Coordination**

調整ロジックをコードではなくsystem promptに書き、調整戦略を「実装」ではなく「設定」にする。Agent編成が急速に反復する段階では特に価値がある——promptの変更コストはコードの変更より遥かに低い。

**パターン5：通知の装飾より安全な状態遷移を優先**

gh-20236の修正パターン：非同期フローでは、まずコアな状態遷移（`completeAsyncAgent`）を完了させ、その後にhangする可能性のある装飾的な操作（分類器チェック、worktreeクリーンアップ）を実行する。ブロックする可能性のある操作が重要な状態変更をgateしてはいけない。

## 4.7 ソースコードインデックス

| ファイル | パス | 重要なエクスポート |
|------|------|---------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool`（buildTool定義）、`inputSchema`、`outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | `runAgent()` AsyncGenerator、`filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition`型ユニオン、`getAgentDefinitionsWithOverrides()`、`parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`、`resolveAgentTools()`、`finalizeAgentTool()`、`runAsyncAgentLifecycle()`、`classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`、`renderToolResultMessage()`、`renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`、`getCoordinatorSystemPrompt()`、`getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool`（5経路ルーティング）、`handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`、`handleSpawnSplitPane()`、`resolveTeammateModel()`、`buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`、`createInProcessCanUseTool()`、`waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`、`writeToMailbox()`、`markMessageAsReadByIndex()`、すべての構造化メッセージ型 |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`、`createAgentWorktree()`、`removeAgentWorktree()`、`validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | `TaskState`ユニオン（7種のtask型）、`isBackgroundTask()` |

**TaskStateユニオン型**（`tasks/types.ts`）：
```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

---

*本章はClaude Code TypeScriptソースコードスナップショット（2026-03-31、~512K LOC）の分析に基づく。すべてのコード参照は具体的なファイル名と行番号範囲を注記している。*
