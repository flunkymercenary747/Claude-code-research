# 第 14 章：Feature Flags とオブザーバビリティ

## 14.1 概要と位置づけ

Claude Code のオブザーバビリティシステムは多層的・多目標型のシステムであり、コンパイル期の機能トリミングからランタイムの動作追跡まで全プロセスをカバーしている。システム全体は 3 つの柱から構成されている：

1. **Feature Flag システム**：二軌道設計——コンパイル時の `feature()` 呼び出し（`bun:bundle` による Dead Code Elimination）とランタイムの GrowthBook 動的設定。前者は異なるユーザーグループにリリースされる機能の境界を制御し、後者は再リリースなしに機能スイッチの調整を可能にする。

2. **オブザーバビリティパイプライン**：OpenTelemetry 標準に基づき、gRPC/HTTP/Protobuf の 3 種の転送プロトコルをサポートし、Metrics、Logs、Traces の 3 種類のシグナルを統一的に収集し、Perfetto 追跡フォーマットで内部デバッグ層を提供する。

3. **Analytics 収集**：二重ルーティング——Datadog（外部モニタリング）+ First-Party 1P イベントログ（内部 BigQuery/proto）。イベント名プレフィックス `tengu_*` ですべてのビジネスイベントを識別し、GrowthBook の動的サンプリング設定でデータ量を制御する。

このシステムのコア設計原則は**階層化による隔離**である：ユーザープライバシー優先（デフォルトでコンテンツとファイルパスを記録しない）、内外部ビルドの差異化（ant-only vs external）、グレースフルデグラデーション（各層にキルスイッチがある）。

---

## 14.2 理論的基礎

### Feature Flag 駆動開発

Feature Flag（機能フラグ）により、チームは同一コードベース内で異なる段階の機能を並行開発し、必要に応じて有効化できる。Claude Code は 2 層のフラグメカニズムを採用している：

- **コンパイル時フラグ**：`bun:bundle` が提供する `feature()` 呼び出しにより、バンドル時に Dead Code Elimination を実行する。外部バージョンに存在しないコードブロック全体が完全に削除され、パッケージサイズを削減するだけでなく、内部ロジックのリバースエンジニアリングを防ぐ。
- **ランタイムフラグ**：GrowthBook SDK を通じてサーバーサイドから動的に取得し、A/B テスト、段階的リリース、緊急キルスイッチなどのシナリオをサポートする。

### オブザーバビリティの 3 つの柱

OpenTelemetry コミュニティはオブザーバビリティを 3 つのシグナル（Three Pillars of Observability）として定義している：

- **Metrics（指標）**：時系列数値データ（API レイテンシ、トークン消費量など）。Claude Code は `@opentelemetry/sdk-metrics` を使用して PeriodicExportingMetricReader で 60 秒ごとにエクスポートする。
- **Logs（ログ）**：構造化イベント記録。すべての `logEvent()` 呼び出しは最終的に OTel `LoggerProvider` + `BatchLogRecordProcessor` でバッチエクスポートされる。
- **Traces（トレース）**：分散呼び出しチェーン。Claude Code は `sessionTracing.ts` を通じて Interaction → LLM Request → Tool Call の階層化 Span ツリーを構築し、マルチエージェントシナリオでの親子関係追跡をサポートする。

### CLI ツールにおける A/B テストの応用

Web 製品とは異なり、CLI ツールの A/B テストは独自の課題に直面する：ブラウザフィンガープリントなし、マルチプラットフォーム・マルチ配布チャネル、オフライン実行シナリオ。Claude Code の対応戦略：

- ユーザー次元ターゲティング：`GrowthBookUserAttributes` は `platform`、`subscriptionType`、`rateLimitTier` などの属性を持ち、階層化された実験をサポートする。
- ローカルディスクキャッシュ：サーバーから特性値の取得に成功するたびに `~/.claude/config.json` の `cachedGrowthBookFeatures` に書き込み、オフライン時でも最後に既知の値を使用できるようにする。
- 露出の重複排除：同一セッション内の各 feature の実験露出イベントは 1 回のみ記録される（`loggedExposures` Set）。

---

## 14.3 Feature Flag システム

### GrowthBook 統合

GrowthBook はオープンソースの Feature Flag および A/B テストプラットフォームである。Claude Code は公式の `@growthbook/growthbook` SDK を通じて統合されており、ファイルは `src/services/analytics/growthbook.ts`（1155 行）にある。

**初期化フロー**：

```typescript
// growthbook.ts:529-600（簡略化）
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

重要な設計：`memoize` によりプロセスのライフサイクル全体で GrowthBook クライアントが 1 回のみ初期化される；auth の変更（ログイン/ログアウト）時は `apiHostRequestHeaders` を更新しようとするのではなく（SDK は初期化後の更新をサポートしない）、`refreshGrowthBookAfterAuthChange()` でクライアントを破棄して再構築する。

**ユーザー属性モデル**（`growthbook.ts:31-46`）：

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

**リフレッシュ戦略**：
- 外部ユーザー：6 時間ごとにリフレッシュ（`6 * 60 * 60 * 1000`）
- 社内従業員（ant）：20 分ごとにリフレッシュ

**キャッシュアーキテクチャ**（3 段階優先度）：
1. メモリ内の `remoteEvalFeatureValues` Map（プロセス内の最新値）
2. ディスクキャッシュ `~/.claude/config.json` の `cachedGrowthBookFeatures`（プロセス間永続化）
3. 旧バージョンの `cachedStatsigGates`（マイグレーション互換レイヤー、段階的に廃止中）

**API 互換 Workaround**（`growthbook.ts:320-390`）：サーバーサイドが返す remoteEval レスポンスは `value` フィールドを使用するが、SDK は `defaultValue` を期待しており、コード内に明示的なフォーマット変換ロジックがあり、サーバーサイドの修正を待つ TODO コメントが付いている。

**環境変数オーバーライド**（ant 社内ユーザーのみ）：
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### コンパイル時 Feature Flag 完全リスト（80+ 個）

`bun:bundle` の `feature()` 呼び出しによるデッドコード排除を実装。以下はソースコードから抽出したすべてのコンパイル時フラグ：

| フラグ名 | 場所 | 制御機能 |
|-----------|---------|---------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Perfetto 追跡（ant-only）|
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | 拡張テレメトリ beta |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | 自動モード/スケジュールタスクシステム |
| `KAIROS_BRIEF` | `commands.ts` | KAIROS 簡略モード |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | KAIROS チャネルサポート |
| `KAIROS_DREAM` | `commands.ts` | KAIROS ドリームモード |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | GitHub webhook サブスクリプション |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | KAIROS プッシュ通知 |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | エージェントトリガー（スケジュールタスク）|
| `AGENT_TRIGGERS_REMOTE` | — | リモートエージェントトリガー |
| `AGENT_MEMORY_SNAPSHOT` | — | エージェントメモリスナップショット |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | 会話分類器 |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | 検証エージェント |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | 内蔵探索/計画エージェント |
| `COORDINATOR_MODE` | `builtInAgents.ts` | コーディネーターモード |
| `FORK_SUBAGENT` | `commands.ts` | Fork サブエージェント |
| `BUDDY` | `commands.ts` | Buddy 機能 |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Unix Domain Socket 受信ボックス |
| `BRIDGE_MODE` | `commands.ts` | ブリッジモード（CCR）|
| `DAEMON` | `commands.ts` | デーモンモード |
| `VOICE_MODE` | `commands.ts` | 音声モード |
| `ULTRAPLAN` | `commands.ts` | UltraPlan コマンド |
| `ULTRATHINK` | — | UltraThink 機能 |
| `TORCH` | `commands.ts` | TORCH コマンド（動的ロード）|
| `MCP_SKILLS` | `commands.ts` | MCP スキルサポート |
| `CHICAGO_MCP` | `metadata.ts` | Chicago MCP 内蔵サーバー（computer-use）|
| `WORKFLOW_SCRIPTS` | `commands.ts` | ワークフロースクリプト |
| `CCR_REMOTE_SETUP` | `commands.ts` | CCR リモートセットアップコマンド |
| `CCR_AUTO_CONNECT` | — | CCR 自動接続 |
| `CCR_MIRROR` | — | CCR ミラーモード |
| `PROACTIVE` | `commands.ts` | プロアクティブモード |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | 実験的スキル検索 |
| `HISTORY_SNIP` | `commands.ts` | 履歴スニペット機能 |
| `HISTORY_PICKER` | — | 履歴ピッカー |
| `WEB_BROWSER_TOOL` | — | Web ブラウザツール |
| `QUICK_SEARCH` | — | クイック検索 |
| `MONITOR_TOOL` | — | モニタリングツール |
| `OVERFLOW_TEST_TOOL` | — | オーバーフローテストツール |
| `BREAK_CACHE_COMMAND` | — | 強制キャッシュブレークポイントコマンド |
| `TREE_SITTER_BASH` | — | Tree-sitter Bash 解析 |
| `TREE_SITTER_BASH_SHADOW` | — | Tree-sitter シャドウ比較 |
| `BASH_CLASSIFIER` | — | Bash セキュリティ分類器 |
| `TERMINAL_PANEL` | — | ターミナルパネル |
| `NATIVE_CLIPBOARD_IMAGE` | — | ネイティブクリップボード画像サポート |
| `NATIVE_CLIENT_ATTESTATION` | — | ネイティブクライアント証明 |
| `AUTO_THEME` | — | 自動テーマ |
| `POWERSHELL_AUTO_MODE` | — | PowerShell 自動モード |
| `TOKEN_BUDGET` | — | トークン予算表示 |
| `STREAMLINED_OUTPUT` | — | 簡略出力モード |
| `CONNECTOR_TEXT` | — | コネクターテキスト |
| `CONTEXT_COLLAPSE` | — | コンテキスト折りたたみ |
| `COMPACTION_REMINDERS` | — | 圧縮リマインダー |
| `CACHED_MICROCOMPACT` | — | キャッシュマイクロ圧縮 |
| `REACTIVE_COMPACT` | — | リアクティブ圧縮 |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Prompt Cache ブレークポイント検出 |
| `EXTRACT_MEMORIES` | — | 自動メモリ抽出 |
| `LODESTONE` | — | Lodestone 機能 |
| `TEAMMEM` | — | チームメモリ |
| `TEMPLATES` | — | テンプレート機能 |
| `FILE_PERSISTENCE` | — | ファイル永続化 |
| `BG_SESSIONS` | — | バックグラウンドセッション |
| `DOWNLOAD_USER_SETTINGS` | — | ユーザー設定ダウンロード |
| `UPLOAD_USER_SETTINGS` | — | ユーザー設定アップロード |
| `NEW_INIT` | — | 新バージョン初期化フロー |
| `HARD_FAIL` | — | ハードフェイルモード |
| `SLOW_OPERATION_LOGGING` | — | 遅い操作のログ |
| `SHOT_STATS` | — | リクエスト統計 |
| `MEMORY_SHAPE_TELEMETRY` | — | メモリ形状テレメトリ |
| `COWORKER_TYPE_TELEMETRY` | — | 協力者タイプテレメトリ |
| `ANTI_DISTILLATION_CC` | — | 反蒸留保護 |
| `RUN_SKILL_GENERATOR` | — | スキルジェネレーター |
| `SKILL_IMPROVEMENT` | — | スキル改善 |
| `REVIEW_ARTIFACT` | — | コードレビュー成果物 |
| `MESSAGE_ACTIONS` | — | メッセージアクション |
| `AWAY_SUMMARY` | — | 離脱サマリー |
| `COMMIT_ATTRIBUTION` | — | コミット帰属 |
| `UNATTENDED_RETRY` | — | 無人再試行 |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | libc タイプ検出（ビルド時インジェクション）|

### ランタイム Feature Flag vs コンパイル時 Feature Flag

| 次元 | コンパイル時（`feature()`） | ランタイム（GrowthBook）|
|------|----------------------|---------------------|
| 実行タイミング | バンドルフェーズ | プロセス起動後の非同期ロード |
| コード保持 | 削除されたブランチは成果物に存在しない | コードは存在するがロジックはフラグ値によって制御 |
| 更新方法 | 新バージョンのリリース | サーバーサイドプッシュ、最短 20 分で有効化 |
| 典型的な用途 | ant-only 機能、実験的ツール、プラットフォーム差異コード | A/B テスト、段階的リリース、キルスイッチ、動的設定 |
| オーバーライド方法 | ビルド変数 | `CLAUDE_INTERNAL_FC_OVERRIDES` 環境変数（ant のみ）|

### Dead Code Elimination メカニズム

`bun:bundle` の `feature()` は Bun バンドラーの特別な内蔵関数で、ビルドフェーズにビルド時の定義に基づいて `feature('X')` を直接 `true` または `false` に置き換え、定数折りたたみとデッドコード排除により常に false となるブランチを削除する。

例（`perfettoTracing.ts:216-220`）：
```typescript
// 外部ビルドでは、この if ブロック全体が完全に削除される
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... すべての Perfetto 初期化コード
}
```

このメカニズムはパッケージサイズを縮小するだけでなく、内部ツールのコードが外部成果物に公開されることを防ぐ。

### 既知の重要なランタイム Feature Flags

以下は既知の `tengu_*` GrowthBook フラグとその機能の一部：

| フラグ名 | タイプ | 機能説明 |
|-----------|------|---------|
| `tengu_auto_mode_config` | Object | 自動モード設定（enabled/opt-in）|
| `tengu_1p_event_batch_config` | Object | 1P イベントバッチエクスポート設定 |
| `tengu_event_sampling_config` | Object | イベントサンプリングレート設定辞書 |
| `tengu_log_datadog_events` | Boolean | Datadog イベント上報スイッチ |
| `tengu_frond_boric` | Object | Analytics sink キルスイッチ（sink 名でこと無効化）|
| `tengu_quartz_lantern` | Boolean | FileWriteTool アトミック書き込み動作制御 |
| `tengu_hive_evidence` | Boolean | タスク更新/Todo 書き込み動作制御 |
| `tengu_plum_vx3` | Boolean | WebSearchTool で Haiku モデルを使用するスイッチ |
| `tengu_kairos_cron` | Object | KAIROS スケジュールタスク設定 |
| `tengu_kairos_cron_durable` | Boolean | 永続的スケジュールタスクサポート |
| `tengu_agent_list_attach` | Boolean | AgentTool リスト追加動作 |
| `tengu_amber_stoat` | Boolean | 内蔵エージェントの可用性制御 |
| `tengu_slim_subagent_claudemd` | Boolean | 簡略サブエージェント CLAUDE.md ロード |
| `tengu_glacier_2xr` | Boolean | ToolSearch モード決定制御 |
| `tengu_max_version_config` | Object | 最大バージョン制限（強制アップグレードキルスイッチ）|
| `tengu_prompt_cache_1h_config` | Object | Prompt Cache 1 時間設定 |
| `tengu_sm_compact_config` | Object | Session Memory 圧縮設定 |
| `tengu_ant_model_override` | String | ant 専用モデルオーバーライド |
| `enhanced_telemetry_beta` | Boolean | 拡張テレメトリ beta スイッチ |

---

## 14.4 オブザーバビリティシステム

### OpenTelemetry 統合

Claude Code は OpenTelemetry の 3 シグナルサポートを完全実装しており、コアエントリーポイントは `src/utils/telemetry/instrumentation.ts`（825 行）にある。

**初期化ブートストラップ**（`instrumentation.ts:bootstrapTelemetry()`）：
ant ビルドでは `ANT_OTEL_*` プレフィックスの変数から設定を読み取り、標準の `OTEL_*` 変数にマッピングする。外部ユーザーに対しては標準の OTel 環境変数設定仕様に従い、デフォルトの temporality は `delta`（累積ではなく増分）に設定される。

**3 シグナルエクスポーター設定**（遅延ロード設計）：

```typescript
// instrumentation.ts:169-190（簡略化）
// OTLP/Prometheus エクスポーターは動的 import 遅延ロードを使用
// 不要時に @grpc/grpc-js（約 700KB）がロードされることを回避
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

3 種類の転送プロトコルすべてをサポート：`grpc`、`http/json`、`http/protobuf`。`OTEL_EXPORTER_OTLP_PROTOCOL` 環境変数で選択する。

**リソース属性**：サービス名は `claude-code` で、プラットフォームアーキテクチャ、WSL バージョン、サブスクリプションタイプ、サービスバージョンなどの属性を持ち、`envDetector`、`hostDetector`、`osDetector` で自動的に埋められる。

### gRPC データ転送

gRPC は企業シナリオで推奨される転送プロトコルで、双方向ストリーミングと強く型付けされた protobuf エンコーディングを提供する。Claude Code では：

- gRPC エクスポーター（`@opentelemetry/exporter-metrics-otlp-grpc`）は遅延ロード依存として、起動時間への影響を回避する
- mTLS 設定は `getMTLSConfig()` でサポートされ、企業内部ネットワークシナリオで自己署名証明書を使用可能
- プロキシサポートは `getProxyUrl()` + `HttpsProxyAgent` で透過的に処理

子プロセスは OTEL 関連の環境変数を継承しない（`subprocessEnv.ts`）：
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Perfetto 追跡

Perfetto は Google が開発した高性能システムレベルの追跡フレームワークで、Claude Code はその Chrome Trace Event フォーマットの互換レイヤーを実装している（`src/utils/telemetry/perfettoTracing.ts`、1120 行、ant-only）。

**有効化方法**：
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # ~/.claude/traces/trace-<session-id>.json に書き込み
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # 指定パスに書き込み
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # 60 秒ごとに定期書き込み
```

**追跡される Span タイプ**：

| Span 名 | 分類 | 保持情報 |
|-----------|------|---------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | attempt 番号 |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**メモリ管理**（イベント上限 100,000 件）：
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// 上限に達したら最も古い半分を削除し、O(1) コストを償却
// trace_truncated マーカーを挿入して ui.perfetto.dev でギャップが見えるようにする
```

**マルチエージェント階層追跡**：各エージェント（サブエージェントを含む）は独立したプロセス ID にマッピングされ、`parent_agent` metadata イベントで階層関係を記録し、Perfetto UI では独立したスイムレーンとして表示される。

**書き込み戦略**（三重保障）：
1. `cleanup registry` 非同期コールバック（正常終了）
2. `process.on('beforeExit')` ハンドラー（バックアップ）
3. `process.on('exit')` 同期書き込み（最後の防衛線、この時点では async 不可）

### OpenTelemetry Session Tracing

`src/utils/telemetry/sessionTracing.ts`（927 行）は外部ユーザー向けの拡張テレメトリエントリーポイントで、Perfetto フォーマットではなく標準 OTel Span に基づいている。

**有効化条件**（`sessionTracing.ts:170-185`）：
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

**AsyncLocalStorage コンテキスト伝播**：各 Interaction と Tool Call は独立した ALS ストレージで SpanContext を保持し、マルチエージェント並行シナリオで Span が混在しないようにする。WeakRef による弱参照で長期存続する span のメモリリークを防ぎ、60 秒間隔で 30 分以上経過した孤立 Span をクリーンアップする。

**logEvent イベント体系**

すべてのビジネスイベントは `src/services/analytics/index.ts` の `logEvent()` 関数を通じて統一ディスパッチされる：

```typescript
// index.ts（簡略化）
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // boolean | number | undefined のみ許可
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

重要な設計：metadata タイプは意図的に `string` を除外しており、開発者に `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 型変換の使用を強制する。これにより型レベルでコンテンツやファイルパスの誤記録を防ぐ。

---

## 14.5 Analytics 収集

### 二重ルーティングアーキテクチャ

すべてのイベントは `sink.ts` を経由して 2 つのバックエンドにルーティングされる：

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog（tengu_log_datadog_events gate が開いている場合）
    │     DATADOG_ALLOWED_EVENTS ホワイトリスト内のイベントのみ転送
    │     _PROTO_* キー（PII マーカーフィールド）を削除
    └─→ 1P First-Party Logger（OpenTelemetry BatchLogRecordProcessor）
          /api/event_logging/batch に送信
          _PROTO_* キーを保持（BigQuery の保護列にルーティング）
```

**Datadog 統合**（`datadog.ts`）：
- エンドポイント：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- バッチ送信：100 件/バッチ、15 秒リフレッシュ間隔
- ネットワークタイムアウト：5 秒
- ホワイトリストメカニズム：約 50 個のコアイベント（`DATADOG_ALLOWED_EVENTS` Set）
- 無効化条件：Bedrock/Vertex/Foundry サードパーティクラウド、テスト環境、ユーザーが no-telemetry を選択

**1P Event Logging（FirstPartyEventLoggingExporter）**：
- OpenTelemetry 標準の `LogRecordExporter` インターフェースを使用
- バッチエクスポート：デフォルト 200 件/バッチ、5 秒スケジュール遅延
- 失敗時の再試行：指数バックオフ（基本 500ms、最大 30s、最大 8 回）
- 永続化失敗キュー：失敗したイベントを `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl` に書き込み、次回起動時に再試行
- Proto シリアライズ：生成された `ClaudeCodeInternalEvent` protobuf タイプを使用

### ユーザー行動追跡

400+ 個の `tengu_*` イベント名が完全なユーザーインタラクションライフサイクルをカバーしている。コアイベントカテゴリ：

**セッションライフサイクル**：`tengu_started`、`tengu_init`、`tengu_exit`、`tengu_cancel`

**API 呼び出し**：`tengu_api_query`、`tengu_api_success`、`tengu_api_error`、`tengu_api_retry`

**ツール使用**：`tengu_tool_use_success`、`tengu_tool_use_error`、`tengu_tool_use_granted_in_prompt_permanent`

**権限リクエスト**：`tengu_internal_bash_tool_use_permission_request`、`tengu_tool_use_show_permission_request`、`tengu_tool_use_granted_by_classifier`

**OAuth 認証**：`tengu_oauth_flow_start`、`tengu_oauth_success`、`tengu_oauth_token_refresh_*`（完全なロックステートマシン追跡）

**MCP サーバー**：`tengu_mcp_server_connection_succeeded`、`tengu_mcp_server_connection_failed`、`tengu_mcp_oauth_flow_*`

**更新メカニズム**：`tengu_binary_download_attempt`、`tengu_native_update_complete`、`tengu_binary_download_failure`

### パフォーマンス指標収集

`sessionTracing.ts` の API Call Span は以下の派生指標を計算する：

```typescript
// perfettoTracing.ts（endLLMRequestPerfettoSpan 簡略化）
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second（入力トークン処理速度）

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second（サンプリング速度）

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // キャッシュヒット率（パーセント）
```

### イベントサンプリング制御

GrowthBook の動的設定 `tengu_event_sampling_config` で各イベントのサンプリングレートを制御する：

```typescript
// firstPartyEventLogger.ts（shouldSampleEvent 簡略化）
// null を返す = 100% サンプリング（設定なし）
// 0 を返す = 完全に破棄
// rate (0-1) を返す = ランダムサンプリング、sample_rate を metadata に書き込む
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // 例：10% サンプリング
}
```

### エラー報告

多層エラーイベント体系：
- `tengu_uncaught_exception`、`tengu_unhandled_rejection`：プロセスレベルのキャッチされていないエラー
- `tengu_api_error`、`tengu_query_error`：API 呼び出しエラー
- `tengu_streaming_error`：ストリーミング応答エラー
- `tengu_atomic_write_error`：ファイル書き込みエラー
- `tengu_compact_failed`：セッション圧縮失敗

---

## 14.6 診断とデバッグ

### /doctor コマンド

`src/commands/doctor/index.ts` が `/doctor` コマンドを登録している：

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

このコマンドは `local-jsx` タイプ（REPL 内で直接 React コンポーネントをレンダリング）で実行され、インストールの整合性、MCP サーバーの接続状態、キーバインド設定の有効性、環境依存（ripgrep など）を確認する。

### 診断追跡システム

IDE 統合シナリオでは、Claude Code は Language Server Protocol を通じてコード診断情報を受け取る。ファイル保存後（`didSave` イベント）、TypeScript Server が新しい診断メッセージを送信し、システムはそれを `<new-diagnostics>` XML タグとしてモデルに渡す：

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### ヒープメモリ診断

`src/utils/heapDumpService.ts` はプロセスレベルのメモリ診断機能を提供し、ヒープダンプのトリガー時に同期的にメモリ使用量スナップショットを収集し、`~/Desktop/<session-id>-diagnostics.json` に出力する。これには `heapUsed`、`external`、`rss` と分析提案が含まれる。対応する analytics イベント：`tengu_heap_dump`。

### エラー回復ログ

`src/utils/telemetry/bigqueryExporter.ts` は BigQuery 指標エクスポーターを実装し、OTEL Metrics パイプラインと統合し、ant 内部の長期パフォーマンスモニタリングとキャパシティプランニングに使用される。`1p_failed_events` 永続化キューにより、ネットワーク障害が発生しても重要なイベントが失われないことを保証する。

---

## 14.7 設計上の意思決定の分析

### コンパイル時フラグの長所と短所

**長所**：
1. **ゼロランタイムオーバーヘッド**：削除されたコードブランチは成果物に存在せず、いかなる条件判断のオーバーヘッドもない
2. **セキュリティ隔離**：ant-only 機能コードは外部ユーザーから完全に不可視で、リバースエンジニアリング不可
3. **パッケージサイズ最適化**：大型モジュール（`@grpc/grpc-js` 約 700KB など）は必要なビルドにのみ存在する
4. **タイプセーフ**：TypeScript の型チェックはバンドル前に適用され、ランタイムに影響しない

**短所**：
1. **リリース依存**：フラグ状態の変更には新バージョンのリリースが必要で、ホットアップデート不可
2. **テストマトリックスの爆発**：N 個のコンパイル時フラグは理論上 2^N 種類のビルド組み合わせのテストが必要
3. **デバッグの複雑さ**：外部ユーザーが問題を報告したとき、一部のコードパスがそのビルドにまったく存在しない

### プライバシーとオブザーバビリティのバランス

Claude Code はプライバシー保護に複数の防御線を採用している：

1. **タイプシステム保護**：`LogEventMetadata` は `boolean | number | undefined` のみを許可し、文字列の直接上報を禁止する。文字列を記録するには `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` を明示的に宣言する必要があり、これは `never` タイプで実際に値を保持できない——開発者に型注釈を記述させ、その文字列がコードやパスを含まないことを手動で検証したことを示す。

2. **MCP ツール名のマスキング**：MCP ツール名の形式 `mcp__<server>__<tool>` はユーザーのプライベートサービス設定を漏洩する可能性があるため、デフォルトで `mcp_tool` にマスキングされる。`cowork` エントリーポイント、公式 MCP レジストリ内のサーバー、または明示的に内蔵として宣言されたサーバー名のみが元の名前を保持する。

3. **PII マーカーフィールド**：`_PROTO_*` プレフィックスの metadata キーは PII センシティブフィールドを示し、1P の保護された BigQuery 列にのみルーティングされ、`sink.ts` は Datadog へ転送する前にこれらのフィールドを除去する。

4. **サードパーティクラウドの無効化**：Bedrock/Vertex/Foundry を使用する企業顧客に対して、すべての Anthropic 側の analytics（Datadog と 1P を含む）はデフォルトで無効になる。

### テレメトリの遅延ロードの理由

OTLP 関連パッケージ（gRPC 約 700KB、proto 約 300KB）が動的 `import()` で遅延ロードされる理由：

1. **起動時間の敏感さ**：CLI ツールの主要なパフォーマンス指標は Time-to-First-Output であり、不必要な初期化はすべて遅延すべき
2. **プロトコルの相互排他性**：1 つのプロセスは 1 種類の転送プロトコルのみ使用するため、すべてのバリアント（6 つのパッケージ）を静的 import するのは純粋な無駄
3. **Bun 最適化互換性**：遅延ロードは Bun のモジュール解析最適化パターンに適合し、静的解析後に必要に応じてバンドルされる

---

## 14.8 移植可能なパターン

以下の設計パターンは他のプロジェクトにとって参考価値が高い：

### 1. タイプシステムによる PII 漏洩防止

`never` タイプのマーカー型を通じて、コンパイル期に開発者が機密情報を含まないことを明示的に確認することを強制する。コスト是ゼロ（ランタイムオーバーヘッドなし）で、保護効果は 100%（回避には明示的な型アサーションが必要）。データ上報のニーズがある任意のシステムに適用可能。

### 2. 二段階 Feature Flag アーキテクチャ

コンパイル時（コード階層化）+ ランタイム（動作制御）の二軌道制で、異なるリリース速度の要件に対応する：
- 構造的機能（モジュール全体の有無）→ コンパイル時
- 動作チューニング（パラメータ、比率、アルゴリズム選択）→ ランタイム

### 3. Sink キルスイッチパターン

`tengu_frond_boric` GrowthBook 設定により、名前（`datadog`、`firstParty`）で任意の analytics バックエンドを独立して無効化できる（新バージョンのリリース不要）。これはすべての複数のダウンストリーム sink を持つイベントシステムに適した汎用的な緊急サーキットブレーカーパターンである。

### 4. 失敗イベントの永続化再試行

1P イベントのエクスポートに失敗した場合、ローカルの JSONL ファイルに書き込み、次回起動時に再試行する。これにより、ネットワーク障害時でも重要なテレメトリデータが失われないことを保証する。特にオフラインまたは不安定なネットワーク環境で実行されるツールに適している。

### 5. 実験露出の重複排除

GrowthBook 実験露出イベント（A/B テスト結果分析に使用）はセッションレベルの Set で重複排除され、同じ feature の露出が分析側で 1 回のみ記録されることを保証し、同じフラグを複数回呼び出すことで露出カウントが過剰になることを防ぐ。

---

## 14.9 ソースコードインデックス

| ファイルパス（`src/` 相対）| 行数 | コアの役割 |
|------------------------|------|---------|
| `services/analytics/growthbook.ts` | 1155 | GrowthBook SDK 統合、Feature Flag 読み取り、A/B 露出記録 |
| `services/analytics/index.ts` | 173 | logEvent 公開 API、イベントキュー、Sink インターフェース定義 |
| `services/analytics/sink.ts` | 114 | 二重ルーティング実装（Datadog + 1P）、初期化 |
| `services/analytics/datadog.ts` | 307 | Datadog バッチログ送信、ホワイトリストフィルタリング |
| `services/analytics/firstPartyEventLogger.ts` | 449 | OpenTelemetry LoggerProvider 初期化、サンプリング制御 |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | 1P イベント HTTP エクスポート、永続化再試行、proto シリアライズ |
| `services/analytics/metadata.ts` | 973 | イベントメタデータ強化、MCP ツール名マスキング、PII 処理 |
| `services/analytics/config.ts` | 38 | isAnalyticsDisabled() 共有ロジック |
| `services/analytics/sinkKillswitch.ts` | 25 | Sink レベルキルスイッチ（tengu_frond_boric）|
| `utils/telemetry/instrumentation.ts` | 825 | OTel SDK 初期化、3 シグナル（Metrics/Logs/Traces）設定 |
| `utils/telemetry/sessionTracing.ts` | 927 | OTel Span 管理、AsyncLocalStorage コンテキスト伝播 |
| `utils/telemetry/perfettoTracing.ts` | 1120 | Perfetto Chrome Trace フォーマット追跡（ant-only）|
| `utils/telemetry/betaSessionTracing.ts` | 491 | Beta 追跡拡張属性 |
| `utils/telemetry/bigqueryExporter.ts` | 252 | BigQuery 指標エクスポーター |
| `utils/telemetry/pluginTelemetry.ts` | 289 | プラグインテレメトリカプセル化 |
| `utils/telemetry/events.ts` | 75 | OTel イベント型定義 |
| `commands/doctor/index.ts` | 12 | /doctor コマンド登録 |
| `commands.ts` | — | コンパイル時 feature() 集中呼び出し箇所 |
