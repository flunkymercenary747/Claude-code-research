# 第1章：アーキテクチャ概要と起動フロー

> データソース：Claude Code TypeScript ソースコードスナップショット（2026-03-31）
> ソースコードパス（mini）：`~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 概要とポジショニング

**Claude Codeとは何か：** Claude Codeはターミナルで動作するAIプログラミングアシスタントで、React/InkでインタラクティブなTUI（Terminal User Interface）をレンダリングし、REPLループでClaude APIを駆動してコード編集、コマンド実行、ファイル操作等の開発タスクを実行する。

### 技術スタック概要

| 層 | 技術 | 用途 |
|------|------|------|
| ランタイム | Bun（主要）/ Node.js 18+（互換） | JavaScript実行環境 |
| 言語 | TypeScript | プロジェクト全体で厳格な型付け |
| UIフレームワーク | React + Ink | ターミナルTUIレンダリング |
| CLIフレームワーク | Commander.js（`@commander-js/extra-typings`） | コマンドライン引数解析 |
| APIクライアント | `@anthropic-ai/sdk` | Claude API呼び出し |
| MCP統合 | `@modelcontextprotocol/sdk` | MCPサーバープロトコル |
| Feature Flag | GrowthBook + `bun:bundle` feature flags | A/Bテストとデッドコード除去 |
| テレメトリー | OpenTelemetry（遅延読み込み ~400KB） | メトリクス/ログ/トレース |
| バリデーション | Zod v4 | 実行時スキーマ検証 |

### コード規模統計

- **総行数**：512,664行（`.ts` + `.tsx`ファイル）
- **ファイル数**：1,884個のTypeScriptファイル
- **トップレベルディレクトリ数**：35個

主要ディレクトリのLOC占有率：

```
utils/       180,472 行  (35.2%)  — ユーティリティ関数、権限、認証、設定等
components/   81,546 行  (15.9%)  — React UIコンポーネント
services/     53,680 行  (10.5%)  — API、MCP、分析、メモリ等のサービス
tools/        50,828 行  (9.9%)   — 30個のツール実装（Bash/File/Agent等）
commands/     26,428 行  (5.2%)   — スラッシュコマンドの実装
screens/       5,977 行  (1.2%)   — REPLなどのトップレベルスクリーン
bootstrap/     ~5,000 行  (1.0%)  — グローバル状態（state.ts 1,758行）
entrypoints/   ~3,000 行  (0.6%)  — CLI/SDK/MCPエントリーポイント
main.tsx       4,683 行  (0.9%)   — メインエントリーコーディネーター
setup.ts         477 行  (0.1%)   — 初期化設定
```

---

## 1.2 理論的基礎

### コマンドラインアプリのアーキテクチャパターン

Claude Codeは2つの古典的なCLIアーキテクチャパターンを融合している：

**REPL Loop（Read-Eval-Print Loop）**
従来のREPLは同期ループでの入力読み取り、評価、出力印刷を行う。Claude Codeはこれを非同期イベント駆動REPLにアップグレード：入力はReactコンポーネントでキャプチャ、「評価」は1回のClaude API round-trip（複数ラウンドのツール呼び出しを含む）、出力はReact/Ink reconcilerでターミナルにレンダリングする。

**Event-Driven Architecture**
起動時にすべての初期化完了を待ってブロックしない——MDM読み取り、Keychainプリフェッチ、MCP接続、Plugin hookの読み込みはすべてfire-and-forget方式で並列トリガーされる（詳細は1.4節参照）。これによりTTFR（Time To First Render）が最小化され、WebアプリのCritical Rendering Path最適化の考え方と一致する。

### ターミナルUIフレームワークの設計哲学：React in Terminal

InkはReactのコンポーネントモデル、宣言的状態、reconciliationメカニズムをターミナルに移植する。コアコンセプト：

- **Virtual DOM → 仮想ターミナルバッファ**：state変化のたびにdiffが発生し、変化した文字行のみを再描画、ちらつきを防止
- **Flexbox → ターミナルレイアウト**：CSS Yogaエンジンで列幅と改行を計算し、ターミナルUIをJSXで宣言的に記述できる
- **コンポーネント再利用**：Loading spinner、確認ダイアログ、Diff表示等のUIロジックをテスト可能なReactコンポーネントとしてカプセル化

これによりClaude CodeのUIコードはWebフロントエンドコードと認知フレームワークを共有し、`components/`ディレクトリの81,546行のコードを馴染みのあるReactパターンで理解できる。

### プラグインアーキテクチャの理論的基礎

Claude Codeのプラグインシステムは「Capability Registration Pattern（能力登録パターン）」に基づく：

- ツール（Tools）、コマンド（Commands）、Hooksはすべて起動時にグローバルレジストリに登録される
- プラグインはファイルシステムの規約（`~/.claude/plugins/`）でツール/コマンドリストを拡張
- `bun:bundle`の`feature()`関数がコンパイル時にDead Code Elimination（DCE）を実施し、実験的機能が外部ビルド成果物に現れないようにする

---

## 1.3 全体アーキテクチャ図

### 分層アーキテクチャ（ASCII）

```
┌─────────────────────────────────────────────────────────┐
│                    エントリー層 (Entry Layer)              │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (CLI交互)    (Commander.jsルーティング) (MCPサーバーモード) │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  ブートストラップ層 (Bootstrap Layer)       │
│    setup.ts      │    entrypoints/init.ts                 │
│  (セッション初期化)  │    bootstrap/state.ts                 │
│  (worktree/tmux)  │    (グローバル状態シングルトン)          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  UI層 (Ink/React TUI)                     │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (メインインタラクション) │  components/ (81K LOC)           │
│  replLauncher.tsx  │  (入力/出力/ダイアログ/待機アニメ)      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  エンジン層 (Engine Layer)                 │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (セッションライフサイクル) │  (API呼び出し) │  (React状態ツリー) │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  ツール層 (Tool Layer)                     │
│  tools/ (30個のツール、50K LOC)                           │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  サービス層 (Service Layer)                │
│  services/ (53K LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (MCPクライアント) (GrowthBook/OTel)         │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (言語サーバー) (セッションメモリ) (企業管理設定)             │
└─────────────────────────────────────────────────────────┘
```

### モジュール依存関係の概要

```
main.tsx
  ├── entrypoints/init.ts       (memoized、一度だけ初期化)
  ├── entrypoints/cli.tsx       (Commanderサブコマンドルーティング)
  ├── bootstrap/state.ts        (グローバル状態、循環依存厳禁)
  ├── setup.ts                  (セッションごとに呼び出し)
  ├── QueryEngine.ts            (headless/SDKパス)
  ├── replLauncher.tsx          (interactiveパス)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (MCPツール/リソース読み込み)
```

**bootstrap/state.tsの特別な位置づけ**：コードには明示的なコメント`// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`があり、ESLintルール`custom-rules/bootstrap-isolation`がこのファイルを非リーフモジュールからインポートすることを防止し、循環依存を避ける。

### 3種のエントリーポイントの比較

| エントリー | ファイル | トリガー方法 | 特徴 |
|------|------|----------|------|
| CLIインタラクション | `entrypoints/cli.tsx` | `claude`コマンド | 完全なREPL + React TUI |
| SDKヘッドレス | `QueryEngine.ts` | `-p`フラグ / SDK API | UIなし、単一またはストリーミング出力 |
| MCPサーバー | `entrypoints/mcp.ts` | `claude --mcp` | ツールセットをMCP serverとして公開 |

---

## 1.4 起動フロー詳解

### main.tsx の完全起動シーケンス

`main.tsx`の4,683行は順次実行ではない——ファイル冒頭のimportの副作用は精密に設計された並列ウォームアップシーケンス。

**フェーズ0：モジュール読み込み期（importの副作用、~135ms）**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. パフォーマンス基準点

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. 並列：MDMサブプロセス（plutil/reg query）

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. 並列：macOS Keychainプリフェッチ（OAuth + APIキー）

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // すべてのimport完了
```

コメントはこれら3つの並列操作のメリットを正確に説明：MDM読み取りは~135msのモジュール評価時間を節約し、Keychainプリフェッチは~65msの順次sync spawnを節約。これがClaude Codeの起動最適化のコアテクニック：**ESモジュールの静的解析特性を利用し、モジュールグラフ評価中にI/O集中型操作を先行実行する**。

**フェーズ1：Commanderルーティング（同期）**

`entrypoints/cli.tsx`でCommander.jsがargvを解析し、サブコマンド（`chat`、`api`、`mcp`、`resume`等）またはフラグに基づいて異なる実行パスに分配：

```typescript
// entrypoints/cli.tsx（簡略化構造）
async function main(): Promise<void> {
  // 高速パス：--version はゼロimport
  // 通常パス：await init() → setup() → 分岐実行
}
```

**フェーズ2：init()初期化（memoized、一度だけ実行）**

`entrypoints/init.ts`の`init`関数は`memoize`でラップされ、複数回呼び出しても一度だけ初期化することを保証：

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // 設定システムの有効化
  applySafeConfigEnvironmentVariables()  // 信頼会話前の安全なenv vars
  applyExtraCACertsFromConfig()     // TLS接続よりも早くCA証明書を設定
  setupGracefulShutdown()           // 終了クリーンアップフックを登録
  // 遅延読み込み：OpenTelemetry（~400KB）+ gRPC（~700KB）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // 非同期キャッシュ
  detectCurrentRepository()          // GitHub repoの検出
  preconnectAnthropicApi()           // TCP+TLSプリコネクト（~100-200msオーバーラップ）
  configureGlobalMTLS()
  configureGlobalAgents()            // proxy設定
})
```

**フェーズ3：setup()セッション初期化（セッションごとに呼び出し）**

```typescript
// setup.ts — 重要ステップのシーケンス
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. UDSメッセージングサーバー（swarm/antモード）
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. ターミナルバックアップチェック（iTerm2/Terminal.app）
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — cwdに依存するすべてのコードより前に必須
  setCwd(cwd)
  // 4. Hooksの設定スナップショット（setCwd()の後に必須）
  captureHooksConfigSnapshot()
  // 5. Worktreeの作成（--worktreeの場合）
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. バックグラウンドタスク登録（SessionMemory、context collapse）
  if (!isBareMode()) initSessionMemory()
  // 7. Pluginプリフェッチ（並列、ノンブロッキング）
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. 分析シンクの有効化 + 最初のテレメトリーイベント
  initSinks()
  logEvent('tengu_started', {})
  // 9. リリースノートのチェック（インタラクティブモード）
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**フェーズ4：REPLレンダリング**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // UIの遅延読み込み
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

最終的にInkがターミナルを引き継ぎ、Reactコンポーネントツリーのレンダリングが開始され、REPLが準備完了。

### 並列プリフェッチ戦略

Claude Codeの起動最適化は「**早くトリガーするほど、待つのが遅くなる**」原則に従う：

| 操作 | トリガータイミング | 待機タイミング |
|------|----------|----------|
| MDMサブプロセス (`plutil/reg query`) | `main.tsx`の1行目のimport副作用 | `applySafeConfigEnvironmentVariables()`呼び出し前 |
| Keychainプリフェッチ (OAuth + APIキー) | `main.tsx`の3行目のimport副作用 | `ensureKeychainPrefetchCompleted()` |
| Claude API TCPプリコネクト | `init()`内の`preconnectAnthropicApi()` | 最初のAPIリクエスト時に自動で接続再利用 |
| Plugin hooksの読み込み | `setup()`内でfire-and-forget | `processSessionStartHooks()`レンダリング前 |
| MCP configs読み取り | `getClaudeCodeMcpConfigs()`のキックオフ | インタラクティブモードの`getMcpToolsCommandsAndResources()` |

### 遅延読み込みメカニズム

Claude Codeは起動のクリティカルパス上の大規模モジュールに対して明示的な遅延読み込みを実施：

```typescript
// entrypoints/init.ts — OpenTelemetry遅延読み込みのコメント
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

また、`replLauncher.tsx`は最後の瞬間までAppとREPLコンポーネントを`import`せず、CommanderルーティングがReactツリーの評価を完了する前に開始しないようにする。

`bun:bundle`の`feature()`関数がコンパイル時のDCEを実装：

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

外部ビルドではこれらのコードが完全に削除され、バンドルサイズを削減する。

### setup.ts 初期化ステップ詳解

`setup.ts`の477行は以下の重要な制約を中心に展開される：

1. **`setCwd()`を最初に呼び出す必須**：後続のすべての操作（hooks、settings、plugin読み込み）が正しいcwdに依存する
2. **Hooksスナップショットは`setCwd()`の後**：正しいディレクトリから`.claude/settings.json`を読み取ることを保証
3. **Worktree作成は`getCommands()`の前**：そうでないと`/eject`コマンドが使用不可
4. **`initSinks()`はすべてのバックグラウンドタスク登録の後**：分析イベントキューが準備完了していることを保証

`--bare`モード（scripted/SDKヘッドレス呼び出し）は多くのインタラクティブ機能をスキップ：ターミナルバックアップチェック、plugin hookプリフェッチ、commit attribution、team memory watcher等、スクリプト呼び出しの起動オーバーヘッドを最小化する。

### bootstrap/state.ts の状態構築

`state.ts`（1,758行）はセッション全体のグローバルシングルトン状態を維持する。コアの`State`型がカバーする：

```typescript
// bootstrap/state.ts（State型定義、一部）
type State = {
  originalCwd: string
  projectRoot: string          // 安定したプロジェクトルートディレクトリ、worktreeは変更しない
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // テレメトリーカウンター
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // ログ/トレースプロバイダー
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... 合計約60フィールド
}
```

**設計上の制約**：コメント`// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`はアーキテクチャの番人。ESLintルール`custom-rules/bootstrap-isolation`がstate.tsを循環依存を引き起こすモジュールからインポートすることを防止する。すべての状態はsetter/getter関数を通じてアクセスされ、可変オブジェクトを直接公開しない。

---

## 1.5 エントリーポイント分析

### CLIエントリー（インタラクティブモード）

`entrypoints/cli.tsx`は最も複雑なエントリーポイントで、すべてのユーザー向け機能ルーティングを担当：

**起動パス**：
1. Commander.jsがargvを解析 → サブコマンドまたはフラグを識別
2. `await init()`初期化（memoized）
3. MCP configs、企業ポリシー、Chrome統合を処理
4. `await setup(cwd, permissionMode, ...)`セッション初期化
5. モードに応じて分岐：
   - **インタラクティブモード**：`showSetupScreens()` → `launchRepl()` → React TUI
   - **Printモード（`-p`）**：`runHeadless()` → `QueryEngine` → stdout
   - **Resumeモード**：`loadConversationForResume()` → 履歴セッションを復元
   - **Teleportモード**：リモートセッションの引き継ぎ

**重要なCLIオプション**（一部）：

| フラグ | 機能 |
|------|------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | 動的MCPサーバー設定 |
| `--worktree` | git worktreeを作成して隔離 |
| `--tmux` | tmuxセッション内で実行 |
| `--model` | メインループモデルをオーバーライド |
| `--resume` | 履歴セッションを復元 |

### SDKエントリー（プログラマティックAPI）

`-p`フラグまたはSDKプログラマティックAPI経由で呼び出す場合、React TUIをバイパスし直接`QueryEngine.ts`に入る：

- `isNonInteractiveSession = true`
- すべてのUIレンダリング（Ink）をスキップ
- `SDKMessage`型のストリーミング出力をstdoutに
- `SDKStatus`、`SDKPermissionDenial`、`SDKCompactBoundaryMessage`等の構造化出力をサポート

SDKモードには専用のbeta featuresもある：`entrypoints/sdk/coreSchemas.ts`が構造化JSON入出力スキーマを定義し、`entrypoints/agentSdkTypes.ts`が`HookEvent`、`ModelUsage`等のSDK専用型を定義する。

### MCPエントリー（MCPサーバーモード）

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools：Claude Codeのすべてのツールをシステムツールとして公開
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool：対応するTool実装にプロキシ実行
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

MCPモードはClaude Codeのツールセット全体（BashTool、FileReadTool、GrepTool等）を外部MCPクライアントに逆公開し、「Claude Code as MCP server」を実現する。

### 3種のエントリーの共有ロジック

どのエントリーを使っても共有する：
- `bootstrap/state.ts` グローバル状態
- `entrypoints/init.ts` 初期化（memoizedで一度だけ実行を保証）
- `Tool.ts` ツールレジストリ
- `services/`下のすべてのサービス（APIクライアント、権限システム等）
- Hooksライフサイクルシステム

違いはReact TUIをレンダリングするかどうかと出力形式（インタラクティブテキスト vs. 構造化JSON）。

---

## 1.6 設計上の意思決定分析

### なぜNode.jsではなくBunを選んだか

コードからBunの使用特性が観察できる：

1. **`bun:bundle`の`feature()`関数**：これはBun固有のコンパイル時Feature Flagメカニズムで、Dead Code Eliminationをサポート。`main.tsx`で大量に使用（COORDINATOR_MODE、KAIROS、CHICAGO_MCP、UDS_INBOX等）、外部ビルドではこれらの実験的コードが完全に削除される。

2. **BunのWebView API**（条件付き参照）：`typeof Bun !== 'undefined' && 'WebView' in Bun`、一部の機能がBun固有のAPIに依存することを示す。

3. **Bunのsingle-file executable**：コメントに`Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`とあり、リリース成果物がBunでコンパイルされた単一ファイル実行ファイルであることを示す。

4. **パフォーマンス**：Bunの起動速度とモジュール読み込み速度はNode.jsを大幅に上回り、CLIツールのTTFRに重要。

Node.js 18+の互換性を維持している（`setup.ts`にNodeバージョンチェックがある）のは、Bun以外の環境（CI、企業管理マシン）をサポートするため。

### なぜReact/InkをターミナルUIに使うか

`components/`ディレクトリの81,546行のコードはUIの複雑度が非常に高いことを示す。RAW ANSIエスケープコードで手書きしたとしたら、メンテナンスコストが制御不能になる。React/Inkの選択がもたらすもの：

1. **宣言的UI**：ストリーミング出力、ツール実行状態、権限確認ダイアログ等はすべてReactのstateで駆動でき、命令的なカーソル制御が不要
2. **コンポーネント隔離**：`screens/REPL.tsx`は全体レイアウトだけを扱い、各サブ機能（入力ボックス、メッセージリスト、ツール進捗）はそれぞれカプセル化
3. **ホットリロード対応**：開発時に標準のReact DevToolsの考え方でデバッグできる
4. **テスト可能性**：Reactコンポーネントは`@testing-library/react`でユニットテストを書けて、実際のターミナルに依存しない

### 並列プリフェッチのパフォーマンス最適化の考え方

Claude Codeの起動最適化には明確な優先度モデルがある：**TTFR（Time To First Render）が最優先であり、「すべての初期化完了」ではない**。

具体的に：
- Keychain読み取り（~65ms）は最初のimport副作用でトリガーされ、APIキーが必要な時まで待たない
- MCPサーバーの接続はバックグラウンドで並列に行われ、REPLレンダリングは待機しない（ユーザーがインターフェースを見てからMCPが接続完了する）
- リリースノート、GrowthBook設定、plugin hooksはすべてfire-and-forget

代償として「プリフェッチ完了前に消費される」というレースコンディションを慎重に管理する必要があり、`ensureKeychainPrefetchCompleted()`等のawaitポイントで精確に制御する。

### 遅延読み込み vs. プリロードのトレードオフ

| 戦略 | 対象 | 理由 |
|------|------|------|
| プリロード（import副作用） | MDMサブプロセス、Keychain | I/O集中型、早いほど良い |
| 遅延読み込み（`await import()`） | OpenTelemetry（~400KB）、gRPC（~700KB）、React TUIコンポーネント | モジュール評価が高コスト、クリティカルパスにない |
| 条件付き読み込み（`feature()`のDCE） | COORDINATOR_MODE、KAIROS、CHICAGO_MCP | 実験的機能、外部ユーザーには不要 |
| `setImmediate()`による遅延 | commit attribution hook | setup()のマイクロタスクウィンドウでイベントループをブロックしないため |

この多層戦略によりClaude Codeは起動時に「インターフェース表示に必要な最小限の作業だけ」を行う。

---

## 1.7 移植可能なパターン

### 起動最適化の汎用パターン

Claude Codeの起動シーケンスは再利用可能な「**並列ウォームアップ + 遅延読み込み + DCE**」三層最適化フレームワークを示している：

**Pattern 1：ESモジュールの副作用を使ったI/Oウォームアップ**
```typescript
// import文の間にfire-and-forget I/Oを挿入
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // 即座にトリガー、awaitしない
import { SomethingElse } from './other.js'  // 並列読み込み
```
適用場面：「必ず読む必要があるが読み取りが遅い」初期化データがある任意のアプリ（設定ファイル、クレデンシャル、ネットワークプリコネクト）。

**Pattern 2：memoizeによる単一初期化**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
適用場面：複数のエントリーポイントが共有する初期化ロジック、重複実行を防止。

**Pattern 3：`--bare`モードの分層**
scripted/API呼び出しはUI、ターミナルチェック、analytics等を必要とせず、`isBareMode()`で素早くスキップし、ヘッドレス呼び出しの低オーバーヘッドを維持。

**Pattern 4：状態の分離**
`bootstrap/state.ts`を厳格なリーフモジュール（循環依存なし）として、setter/getterでアクセスし、ESLintルールで強制実施。これにより状態モジュールをどこからでも安全にimportできる。

### Doramagic CLIが参考にできること

上記の分析に基づき、Doramagic CLIのアーキテクチャ設計で以下のパターンが採用可能：

1. **起動クリティカルパスの分離**：「レンダリング前に完了必須」と「レンダリング後に完了可能」な初期化を厳格に分け、コメントで理由を注記（Claude Codeの`// ~65ms on every macOS startup`コメントスタイルを参考に）

2. **グローバル状態シングルトン + アクセサーパターン**：`bootstrap/state.ts`を参考に、厳格なリーフモジュールでセッション状態を維持し、状態が散在するのを避ける

3. **`memoize`初期化関数**：どのエントリーポイントからの呼び出しでも、初期化が一度だけ実行されることを保証

4. **3つのモードの分離**：interactive（React TUI）/ headless（-pフラグ）/ server（MCP）が、基盤となるツールとサービス層を共有

5. **feature flag + DCE**：実験的機能をfeature flagでラップし、リリース時に自動的に削除

---

## 1.8 ソースコードインデックス

| ファイル | 行数 | 重要な内容 |
|------|------|----------|
| `main.tsx` | 4,683 | メインエントリー、Commanderルーティング、状態初期化、インタラクティブ/ヘッドレス分岐 |
| `setup.ts` | 477 | セッション初期化：cwd、hooks、worktree、pluginプリフェッチ |
| `bootstrap/state.ts` | 1,758 | グローバル状態シングルトン、`State`型定義、すべてのgetter/setter |
| `entrypoints/init.ts` | ~400 | memoizedグローバル初期化：config、mTLS、proxy、OTel遅延読み込み |
| `entrypoints/cli.tsx` | ~2,000 | Commander.jsルーティング、インタラクティブ/print/resume/teleport分岐 |
| `entrypoints/mcp.ts` | ~200 | MCPサーバーモード、ツールセットを公開 |
| `entrypoints/sdk/coreSchemas.ts` | - | SDKモード構造化入出力スキーマ |
| `entrypoints/agentSdkTypes.ts` | - | SDK専用型（HookEvent、ModelUsage等） |
| `replLauncher.tsx` | ~30 | App + REPLの遅延読み込み、React TUIの起動 |
| `QueryEngine.ts` | ~1,500 | セッションライフサイクル管理、ヘッドレスパスのコア |
| `Tool.ts` | - | ツールインターフェース定義（inputSchema、call、prompt等） |
| `tools/` | 50,828 | 30個のツール実装（BashTool/FileEditTool/AgentTool等）|
| `services/api/` | - | Claude API呼び出し、リトライ、usage統計 |
| `services/mcp/client.ts` | - | MCPクライアント接続管理 |
| `utils/startupProfiler.ts` | - | `profileCheckpoint()`パフォーマンス計測 |
| `utils/secureStorage/keychainPrefetch.ts` | - | macOS Keychain並列プリフェッチ |
| `utils/settings/mdm/rawRead.ts` | - | MDM設定並列読み取り |

### 重要なコードの所在

- **並列ウォームアップ起点**：`main.tsx:12-20`（3つのimport副作用）
- **memoized初期化**：`entrypoints/init.ts:57`（`export const init = memoize(...)`）
- **グローバル状態型**：`bootstrap/state.ts:30-200`（`type State = {...}`）
- **MCPサーバー定義**：`entrypoints/mcp.ts:42`（`startMCPServer`）
- **REPLレンダリングエントリー**：`replLauncher.tsx:14`（`launchRepl`）
- **ツールインターフェース**：`Tool.ts:1-30`（`ToolInputJSONSchema`、`ToolUseContext`）
- **setupの重要な順序**：`setup.ts:77-230`（setCwd → captureHooksConfigSnapshot → worktree → バックグラウンドジョブ）

---

*章の文字数：約9,800文字 | ソースコードスナップショット日：2026-03-31*
