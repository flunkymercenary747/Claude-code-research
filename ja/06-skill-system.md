# 第 6 章：Skill システム

## 6.1 概要とポジショニング

Skill システムは Claude Code の中で最も革新的なアーキテクチャの一つである。再利用可能なワークフロー（workflow）を Markdown ファイルとして記述し、スラッシュコマンド（`/skill-name`）または AI の能動的な呼び出しによってトリガーする。本質的に Skill は「AI のための SOP」——人間の専門家が複雑なタスクを実行する際の手順・判断条件・成功基準を構造化された Markdown 形式で記録し、AI に再現可能な専門的実行能力を与えるものである。

通常の Prompt と異なり、Skill システムは以下のコアとなる特徴を持つ：

1. **宣言型 + 実行型の融合**：Frontmatter でメタデータ（権限、モデル、トリガー条件）を宣言し、本文は実行指令となる
2. **マルチソース読み込み**：組み込み（bundled）、ユーザーレベル、プロジェクトレベル、Plugin レベル、MCP ソースを優先度順にマージ
3. **2 種類の実行モード**：inline（現在のセッションコンテキストに注入）と fork（独立したサブエージェント内で分離実行）
4. **条件付き有効化**：`paths` frontmatter によってファイルパスに応じて自動的に有効化される Skill を実装
5. **動的発見**：セッション中、ユーザーのファイル操作に応じてより深い階層のディレクトリの Skill を自動的に発見して読み込む

Skill システムは単純なコマンドエイリアスではなく、完全なワークフローオーケストレーションフレームワークである。

---

## 6.2 理論的背景

### 再利用可能ワークフロー（Reusable Workflows）のデザインパターン

Skill システムは AI ツール活用における核心的な問題を解決する：**専門知識をどのように蓄積し、再現可能にするか？** 従来のコード再利用は関数とクラスで行うが、AI が実行する「知識」は自然言語で記述されたワークフローであり、コードの関数として直接カプセル化することはできない。

Skill の設計は SOP（Standard Operating Procedure）の考え方を参考にしている——専門家の実行フロー、意思決定ポイント、成功基準を構造化して記録することで、AI が毎回同じ高品質なパスをたどれるようにする。

### 宣言型 vs 命令型ワークフロー定義

Skill システムは両方のスタイルに対応する：

- **宣言型**：frontmatter で `allowed-tools`、`model`、`context` などの属性を宣言し、システムが権限制御と実行コンテキスト設定を自動処理
- **命令型**：Skill 本文中にシェルコマンド（`!``command``）を直接埋め込んで実行し、「説明の中に操作を混在」させることができる

### Markdown-as-Code の理念

Skill フォーマットとして JSON/YAML ではなく Markdown を選んだのは熟慮の末のデザイン決定である：

- **人間による可読性**：開発者が Skill を直接読んで編集でき、その意図を理解できる
- **AI との親和性**：AI のトレーニングデータには Markdown が大量に含まれており、AI は JSON よりも Markdown を自然に理解する
- **段階的な構造化**：純散文から始めて、見出し・手順・ルールを徐々に追加でき、完全な構造を強制しない
- **バージョン管理への親和性**：Markdown の diff は人間にとって読みやすく、コードレビュー時にワークフローの変更が一目でわかる

---

## 6.3 Skill フォーマットとデータ構造

### Skill Markdown ファイルのフォーマット仕様

Skill ファイルは固定のディレクトリ構造に従う：

```
.claude/skills/<skill-name>/SKILL.md
```

ファイルフォーマットは frontmatter + Markdown 本文：

```markdown
---
name: my-skill
description: この Skill が何をするかを一文で説明
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  ユーザーが... をしたいとき。例："cherry-pick to release"、"hotfix"。
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## 手順

### 1. 最初のステップ
具体的な操作...

**成功基準**: このステップの完了を証明するチェックポイント
```

### frontmatter フィールド詳細

以下は `parseSkillFrontmatterFields` 関数（`loadSkillsDir.ts:184`）が解析する全フィールドである：

| フィールド | 型 | 説明 |
|------|------|------|
| `name` | string | 表示名（ディレクトリ名と異なっても可） |
| `description` | string | 一文での説明、`/help` に表示される |
| `allowed-tools` | string[] | 許可ツールのホワイトリスト、`Bash(git:*)` プレフィックスパターン対応 |
| `argument-hint` | string | ユーザーがトリガーする際の引数ヒント（例：`"<branch-name>"`） |
| `arguments` | string[] | 引数名リスト、`$arg_name` 変数置換に使用 |
| `when_to_use` | string | AI がこの Skill を能動的に呼び出すタイミングを伝える（トリガーフレーズを含む） |
| `version` | string | Skill バージョン番号 |
| `model` | string | モデルのオーバーライド（`opus`、`sonnet`、`inherit` は継承を意味する） |
| `disable-model-invocation` | boolean | AI による能動的な呼び出しを禁止し、ユーザーの手動トリガーのみ許可 |
| `user-invocable` | boolean | `/help` に表示されるか（デフォルト `true`） |
| `context` | `"fork"` | 設定時に独立したサブエージェント内で実行 |
| `agent` | string | agent タイプを指定 |
| `effort` | EffortValue | モデルの思考深度に影響 |
| `paths` | string[] | gitignore 構文のパスパターン、条件付き有効化に使用 |
| `hooks` | HooksSettings | Skill 実行中の hook 設定 |
| `shell` | FrontmatterShell | インライン shell コマンド実行設定 |

### SkillDefinition 型

`bundledSkills.ts` には `BundledSkillDefinition`（12-41 行）が定義されており、ファイルシステム Skill に対応するのは `Command` 型（`src/types/command.js`）である。両者は `createSkillCommand`（`loadSkillsDir.ts:269`）で統一された `Command` オブジェクトに集約される：

```typescript
// loadSkillsDir.ts:316-400
return {
  type: 'prompt',
  name: skillName,
  description,
  allowedTools,
  argumentHint,
  argNames: argumentNames.length > 0 ? argumentNames : undefined,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context: executionContext,
  agent,
  effort,
  paths,
  contentLength: markdownContent.length,
  isHidden: !userInvocable,
  progressMessage: 'running',
  loadedFrom,
  hooks,
  skillRoot: baseDir,
  async getPromptForCommand(args, toolUseContext) { ... }
} satisfies Command
```

---

## 6.4 Skill 読み込みメカニズム

### loadSkillsDir の完全な読み込みフロー

`getSkillDirCommands`（`loadSkillsDir.ts:638`）は読み込みフロー全体のエントリポイントであり、`lodash-es/memoize` で結果をキャッシュして重複 I/O を防ぐ：

```
起動時
  ├── policySettings: ~/.claude-managed/.claude/skills/（企業管理）
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/（cwd から home まで上位探索）
  ├── additionalDirs: --add-dir で指定した追加ディレクトリ
  └── legacyCommands: .claude/commands/（後方互換）

セッション中（動的発見）
  └── ユーザーがファイルを読み書き → discoverSkillDirsForPaths() → addSkillDirectories()
```

読み込み結果は `realpath` で重複排除される（`loadSkillsDir.ts:728-763`）。symlink による重複読み込みを防ぐ。

### マルチソース読み込み優先度

コードコメントに読み込み優先度が明示されている（`loadSkillsDir.ts:677-714`）：

```
managed（企業ポリシー） < user（ユーザーレベル） < project（プロジェクトレベル） < additional（--add-dir）
```

「より具体的なものほど優先度が高い」という原則：プロジェクトにはプロジェクト固有の要件があるため、プロジェクトレベルはユーザーレベルを上書きする。

**特殊ケース：**
- `--bare` モード：自動発見をスキップし、`--add-dir` で明示されたディレクトリのみ読み込む
- `skillsLocked`（plugin-only ポリシー）：ユーザー/プロジェクトレベルの Skill の読み込みを禁止し、Plugin ソースのみ許可
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS` 環境変数：managed レベルの Skill をスキップ

### Skill 発見とマッチングロジック

**静的発見**（起動時）：`getSkillDirCommands` が各レベルの `~/.claude/skills/` ディレクトリをスキャン。ディレクトリ形式（`skill-name/SKILL.md`）のみ対応し、単一の `.md` ファイルは非対応。

**動的発見**（セッション中）：ユーザーがファイルを読み書きする際、`discoverSkillDirsForPaths`（`loadSkillsDir.ts:861`）がファイルパスを上位方向にたどり、各ディレクトリに `.claude/skills/` が存在するか確認し、発見した場合は `addSkillDirectories` で読み込む。`.gitignore` でマークされたディレクトリはスキップされる（`node_modules` 内の Skill による汚染を防ぐ）。

**条件付き有効化**（paths frontmatter）：`paths` フィールドを持つ Skill は初期状態でモデルに見えない状態で `conditionalSkills` Map に保存される。ユーザーが一致するパスのファイルを操作すると、`activateConditionalSkillsForPaths`（`loadSkillsDir.ts:997`）が `ignore` ライブラリ（gitignore 構文）でマッチングを行い、ヒットした場合は `dynamicSkills` に移動して有効化する。

---

## 6.5 SkillTool 実行フロー

### /skill-name から実行までの完全なパス

`SkillTool`（`tools/SkillTool/SkillTool.ts:330`）は標準的な `Tool` 実装であり、AI はこのツールを呼び出すことで Skill を実行する。完全な実行パスは以下の通り：

```
ユーザーが /skill-name を入力するか AI が SkillTool を呼び出すことを決定
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── 先頭のスラッシュを除去（互換処理）
  │     ├── _canonical_ プレフィックスを確認（リモート Skill、実験的）
  │     ├── findCommand() で登録済み Command を検索
  │     ├── disableModelInvocation フラグを確認
  │     └── type === 'prompt' であることを確認
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── deny ルールを確認
  │     ├── リモート canonical Skill を確認（自動許可）
  │     ├── allow ルールを確認
  │     ├── skillHasOnlySafeProperties() → 安全な Skill を自動許可
  │     └── デフォルト：ダイアログでユーザーに確認（behavior: 'ask'）
  │
  └── call (SkillTool.ts:580)
        ├── context === 'fork' を確認 → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent()（独立サブエージェント）
        └── それ以外（inline）→ processPromptSlashCommand()
              └── newMessages + contextModifier を現在のセッションに注入
```

### Skill コンテキスト注入

Inline 実行時、`call` は `newMessages` と `contextModifier` を返す（`SkillTool.ts:767-840`）：

- **newMessages**：Skill を展開したメッセージリスト、現在の会話コンテキストに注入
- **contextModifier**：`ToolUseContext` を変更する関数、以下に使用：
  - `allowedTools` を積み重ねる（Skill が宣言したツール権限）
  - `mainLoopModel` を上書き（Skill がモデルを指定した場合）
  - `effortValue` を上書き（Skill が effort を指定した場合）

注目すべき点として、`contextModifier` はチェーン呼び出しパターンを採用しており（`SkillTool.ts:777`）、複数の contextModifier が積み重なる場合を単純な上書きではなく正しく処理する。

### Skill 変数置換

`createSkillCommand` 内の `getPromptForCommand`（`loadSkillsDir.ts:343-398`）は Skill の内容を返す前に以下の置換を実行する：

1. **引数置換**：`$arg_name` → `substituteArguments()` でユーザーが渡した引数を注入
2. **ディレクトリ変数**：`${CLAUDE_SKILL_DIR}` → Skill ファイルが存在するディレクトリの絶対パス
3. **Session ID**：`${CLAUDE_SESSION_ID}` → 現在のセッション ID
4. **シェルコマンド実行**：`!``command`` ` → 実行結果をインライン展開（非 MCP Skill のみ）

MCP Skill ではシェルコマンド実行が無効化されている（`loadSkillsDir.ts:372`）。リモートの信頼できない Skill が任意の shell コマンドを注入することを防ぐためである。

### Skill とツールの連携

Fork 実行モード（`executeForkedSkill`、`SkillTool.ts:121`）では、Skill は完全に分離されたサブエージェント内で実行される：

- `runAgent()` で独立した agent を起動し、独自のトークンバジェットを持つ
- 実行中の tool use メッセージは `onProgress` コールバックで上位に報告され、UI で進捗を表示できる
- 実行結果は `extractResultText` で最終テキストを抽出し、親 agent に返す
- `clearInvokedSkillsForAgent` でメモリを解放（`SkillTool.ts:286`）

---

## 6.6 Bundled Skills 完全リストと分析

組み込み Skill は `registerBundledSkill()`（`bundledSkills.ts:55`）で登録され、CLI 起動時に初期化される。以下は全 17 個の組み込み Skill の分析である：

### 1. `update-config`（`updateConfig.ts`、475 行）

**機能**：Permissions、Hooks、Model、MCP などすべての設定項目を含む Claude Code の `settings.json` を設定する。

**特徴**：Skill の本文は動的に生成される——`toJSONSchema(SettingsSchema())` を使って Zod schema から JSON Schema ドキュメントを自動生成し、ドキュメントが実際の型と常に同期されることを保証する。すべての Hooks ドキュメント（全 Hook イベント、Hook タイプ、JSON 出力フォーマット）を含む。

**トリガーシナリオ**：ユーザーが動作の自動化、権限ルール、環境変数、モデル設定を構成したいとき。

### 2. `schedule`（`scheduleRemoteAgents.ts`、447 行）

**機能**：リモートのスケジュール済み Agent（cron トリガー）を管理し、スケジュールタスクを作成・更新・一覧表示・実行する。

**特徴**：呼び出し前に複数の前提条件（OAuth トークン、リポジトリ情報、MCP コネクター、クラウド環境）を確認し、これらの動的情報を Skill プロンプトに注入する。`AskUserQuestion` ツールでユーザーと対話する。

**トリガーシナリオ**：ユーザーがスケジュール実行の Claude Code agent（例：毎日のコードレビュー、自動レポート）を作成したいとき。

### 3. `keybindings-help`（`keybindings.ts`、339 行）

**機能**：ユーザーがキーボードショートカットをカスタマイズし、`~/.claude/keybindings.json` を変更できるよう支援する。

**特徴**：`generateContextsTable()`、`generateActionsTable()` でコードの定数からドキュメントを動的に生成し、`generateReservedShortcuts()` でリバインドできないショートカットを一覧表示してユーザーの誤操作を防ぐ。

**トリガーシナリオ**：ユーザーがショートカットのリバインド、コンビネーションキーの追加、サブミットキーの変更をしたいとき。

### 4. `lorem-ipsum`（`loremIpsum.ts`、282 行）

**機能**：固定数のシングルトークン単語の占位テキストを生成し、トークンカウントと性能テストに使用する。

**特徴**：API で検証済みのシングルトークン単語リストを使用し、`lorem` パラメーターで正確なトークン数を制御できることを保証する。ベンチマークテストとトークン課金分析によく使われる。

**トリガーシナリオ**：正確なトークン数を持つテストテキストが必要なとき。

### 5. `skillify`（`skillify.ts`、197 行）

**機能**：現在のセッションの操作プロセスを再利用可能な SKILL.md ファイルに自動変換する。

**特徴**：これは Skill システムの「自己複製」メカニズムである。session memory とユーザーメッセージ履歴を読み込み、4 ラウンドの `AskUserQuestion` ダイアログでワークフロー名・手順・引数・トリガー条件を確認し、最終的に標準フォーマットの SKILL.md を生成してディスクに書き込む。

**制限**：`USER_TYPE === 'ant'`（Anthropic 社内従業員）のみ利用可能。

**トリガーシナリオ**：セッション終了時、ユーザーが直前に完了した操作フローを再利用可能な Skill として固化したいとき。

### 6. `claude-api`（`claudeApi.ts`、196 行 + `claudeApiContent.ts`、220 行）

**機能**：開発者が Claude API または Anthropic SDK を使ってアプリケーションを構築できるよう支援する。

**特徴**：
- 現在のプロジェクト言語を自動検出（ファイル拡張子をスキャン、Python/TypeScript/Java/Go/Ruby/C#/PHP/curl をサポート）
- 遅延読み込み（247KB の `.md` コンテンツは呼び出し時のみ読み込み）で起動時間への影響を回避
- 言語固有の API ドキュメント、Agent SDK パターン、streaming などを含む
- `files` メカニズムでドキュメントを一時ディレクトリに書き込み、モデルが Read/Grep ツールで必要に応じて読み込める

**トリガーシナリオ**：コードが `anthropic` をインポートしているか、ユーザーが Claude API の使い方を尋ねるとき。

### 7. `batch`（`batch.ts`、124 行）

**機能**：大規模なコード変更（マイグレーション、リファクタリング、一括リネームなど）を 5〜30 個の並列 worktree agent で実行するよう分解する。

**特徴**：3 フェーズ実行モデル——Plan（Plan Mode に入り深く研究して分解）→ Spawn Workers（`isolation: "worktree"` を持つバックグラウンド agent を並列起動）→ Track Progress（ステータステーブルをリアルタイム描画）。各 worker は独立した git worktree 内で動作し互いに影響しない。完了後に PR を作成する。

**トリガーシナリオ**：大規模なコードマイグレーション、全リポジトリのリファクタリング、一括変更。

### 8. `loop`（`loop.ts`、92 行）

**機能**：固定間隔でプロンプトまたはスラッシュコマンドを繰り返し実行する。

**特徴**：時間間隔をスマートに解析（`5m`、`2h` のプレフィックス形式と `every 20m` のサフィックス形式に対応）し、cron 式に変換して `ScheduleCronTool` でスケジュールタスクを登録する。設定後、最初のスケジュールトリガーを待たずに即座に一度実行する。

**トリガーシナリオ**：ユーザーが定期的にデプロイ状態を確認したい、または特定の Skill を周期的に実行したいとき。

### 9. `remember`（`remember.ts`、82 行）

**機能**：auto-memory エントリを審査し、`CLAUDE.md`、`CLAUDE.local.md`、またはチーム memory への昇格を提案する。

**特徴**：「先に提案、後に確認」の原則を採用し、ファイルを直接変更せず、分類レポート（昇格待ち/クリーン待ち/要検討/対応不要）を表示してユーザーの承認後に実行する。プロジェクトレベルの慣例（CLAUDE.md）、個人の好み（CLAUDE.local.md）、組織レベルの知識（チーム memory）を区別する。

**制限**：`USER_TYPE === 'ant'` かつ auto-memory 機能が有効なときのみ利用可能。

**トリガーシナリオ**：ユーザーが memory を整理して auto-memory の無限積み上げを防ぎたいとき。

### 10. `simplify`（`simplify.ts`、69 行）

**機能**：現在の git diff に対して 3 次元のコードレビュー（コード再利用、コード品質、効率性）を行い、発見された問題を直接修正する。

**特徴**：3 つの並列サブエージェントを同時に起動し、それぞれが担当する：
- **コード再利用 Agent**：車輪の再発明を発見し、既存のユーティリティ関数を指摘
- **コード品質 Agent**：冗長な状態、引数の肥大化、コピーペースト、漏れた抽象化などを発見
- **効率性 Agent**：不必要な計算、並行処理の欠如、N+1 パターン、メモリリークなどを発見

3 つの agent が完了後に発見内容をマージして直接修正し、報告だけで終わらない。

**トリガーシナリオ**：コードを書き終えた後の品質チェック、`batch` Skill の worker フローにも自動的に呼び出される。

### 11. `debug`（`debug.ts`）

**機能**：現在の Claude Code セッションのデバッグログを診断し、問題のトラブルシューティングを支援する。

**特徴**：tail 読み込み（最大 64KB）でデバッグログの最後の数行を読み込み、長いセッションでログファイルが大きくなることによるメモリピークを回避する。Anthropic 従業員以外のユーザーに対しては、デバッグログを有効化してから読み込む。`disableModelInvocation: true` としてマークされており、AI による自動呼び出しを防ぐ（ユーザーの手動トリガーのみ）。

### 12. `stuck`（`stuck.ts`）

**機能**：マシン上の他のフリーズまたはハングした Claude Code プロセスを診断し、レポートを Slack チャンネルに送信する。

**特徴**：Anthropic 内部診断ツール。高 CPU 使用率（≥90% 継続）、D 状態（I/O 待ち）、T 状態（Ctrl+Z 停止）、Z 状態（ゾンビプロセス）、高メモリ使用量（≥4GB）などの異常を検出する。2 メッセージ構造で Slack レポートを送信（トップレベルサマリー + スレッド詳細）。

### 13. `verify`（`verify.ts`）

**機能**：アプリケーションを実行することでコード変更が期待通りかどうかを検証する。

**特徴**：`verifyContent.ts` から Skill 本文を読み込み（SKILL.md を解析）、`files` メカニズムで補助ファイルを一時ディレクトリに書き込む。`USER_TYPE === 'ant'` のみ利用可能。

### 14. `claudeInChrome`（`claudeInChrome.ts`）

**機能**：実際の Chrome ブラウザに接続した headless セッションを起動する。Side Panel 拡張機能付きで、Claude がリアルタイムでブラウザを制御できる。

### 15. `claudeCodeGuide`（`AgentTool` システム内に組み込み）

Claude Code 内部のガイダンスフローに使用する。

---

## 6.7 Skill と Command の関係

### 両者の境界

Claude Code の設計では、Skill と Command はかつて異なる概念であったが、現在は統一されている：

- **歴史的には**：`/commands/` ディレクトリはシンプルな prompt コマンド（`.md` ファイル）を格納し、`/skills/` ディレクトリはより複雑なディレクトリ構造を持つワークフロー（`skill-name/SKILL.md`）を格納していた
- **現在**：両者とも `loadSkillsDir.ts` で読み込まれ、統一して `Command` 型に変換される。`/commands/` は `loadedFrom: 'commands_DEPRECATED'` としてマークされている（`loadSkillsDir.ts:608`）

現在の実際の違いは読み込みパスのみである：
- `/skills/skill-name/SKILL.md`：新フォーマット、推奨使用、`baseDir` に対応（Skill が補助ファイルを持てる）
- `/commands/skill-name.md` または `/commands/skill-name/SKILL.md`：旧フォーマット、後方互換

### Skill を使うべき場面、Command を使うべき場面

| シナリオ | 推奨方法 |
|------|---------|
| マルチファイルワークフロー（Skill が補助リソースファイルを持つ） | `/skills/` ディレクトリ形式 |
| シンプルな prompt の再利用（単一 md ファイルで十分） | `/commands/` のまま使用可（互換） |
| `${CLAUDE_SKILL_DIR}` 変数が必要 | `/skills/` ディレクトリ形式必須 |
| `files:` でリソースを埋め込む必要（bundled skill） | `BundledSkillDefinition.files` |
| CLI バイナリに組み込む | `registerBundledSkill()` |

---

## 6.8 設計決定の分析

### なぜ JSON/YAML ではなく Markdown を選んだか

Skill の実行指令（本文）は AI が理解して従えるように自然言語で書く必要がある。JSON/YAML は構造化データのみをエンコードでき、「まず関連ファイルを検索し、次に依存関係を分析し、テストファイルは変更しないよう注意する」といった複雑な指令を直接書くことはできない。

Markdown は両方の長所を持つ：frontmatter（YAML）が構造化されたメタデータを担い、本文（Markdown）が人間が読める実行指令を担う。これは実用主義的なフォーマット選択である。

### Skill の権限制御

権限制御は「ホワイトリスト + 確認要求」メカニズムを採用する（`SkillTool.ts:871-900`）：

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // CommandBase properties...
  'name', 'description', 'hasUserSpecifiedDescription', ...
  // NOT included: 'allowedTools', 'hooks', 'paths', etc.
])
```

`skillHasOnlySafeProperties()` は Skill が「安全なプロパティ」のみを使用しているかを確認する——Skill が `allowedTools`、`hooks`、`paths` などの敏感なプロパティを宣言していない場合、ユーザー確認なしに自動的に実行を許可する。これは優れたセキュリティ設計である：新しく追加されたプロパティはデフォルトで安全でないとみなされ、明示的に審査されてからホワイトリストに追加される必要がある。

### 安全なファイル書き込みメカニズム

組み込み Skill は `files` フィールドで補助ファイルを埋め込み、ディスクへの書き込み時に厳格なセキュリティ対策を使用する（`bundledSkills.ts:171-194`）：

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

`O_NOFOLLOW | O_EXCL` で symlink 攻撃を防ぎ、ファイルパーミッションは `0o600`（所有者のみ読み書き可能）。書き込みディレクトリにはプロセス起動ごとのランダムな nonce が含まれており、パス予測攻撃を防ぐ。

### MCP Skill の統合戦略

MCP Skill は `mcpSkillBuilders.ts` を通じて優雅な依存性逆転を実現している（`mcpSkillBuilders.ts:1-43`）：

MCP 発見ロジック（`mcpSkills.ts`）は `createSkillCommand` と `parseSkillFrontmatterFields` を使用する必要があるが、直接 import すると循環依存が生じる。解決策は：

1. `loadSkillsDir.ts` がモジュール初期化時に `registerMCPSkillBuilders()` を呼び出してこの 2 つの関数を登録する
2. `mcpSkills.ts` が必要なときに `getMCPSkillBuilders()` で取得する

この設計は Bun バンドルの技術的制約も解決している：Bun バンドルでは変数を使った動的 import（非リテラル）は解決できないため、`await import(variable)` 方式は使えない。そのためこのレジストリパターンを使うしかない。

---

## 6.9 移植可能なパターン

### Doramagic Skill システムとの比較

| 次元 | Claude Code Skill | Doramagic Skill |
|------|------------------|-----------------|
| ファイルフォーマット | `SKILL.md`（Markdown + YAML frontmatter） | `SKILL.md`（同じフォーマット）|
| ディレクトリ構造 | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| 実行エンジン | SkillTool（AI ツール呼び出し） | OpenClaw ツール呼び出し |
| ソース優先度 | policy < user < project < plugin | OpenClaw プラットフォームルール |
| 組み込み Skill | 15+ 個、バイナリにコンパイル | 構築中 |
| 引数置換 | `$arg_name`、frontmatter `arguments` | 同じメカニズム |
| 実行コンテキスト | inline / fork（サブ agent） | inline（現段階）|
| 条件付き有効化 | `paths` frontmatter | 未実装 |
| 動的発見 | ファイル操作で自動発見をトリガー | 未実装 |

### 参考にすべきコアパターン

**1. `skillify` パターン：ワークフローの自己複製**

Claude Code の `skillify` Skill は極めて優雅な設計である——AI に自分が直前に実行した操作を分析させ、ダイアログでユーザーを誘導しながら再利用可能な Skill として固化する。Doramagic も同様に `/dora-skillify` を実装でき、一度成功した知識抽出プロセスをプロジェクト固有の Skill として固化できる。

**2. `when_to_use` による AI の能動的呼び出しメカニズム**

`when_to_use` frontmatter フィールドにより、AI はユーザーが明示的にスラッシュコマンドを入力しなくても、いつ Skill を能動的に呼び出すべきかを知ることができる。Doramagic の Skill もこのフィールドを重視し、知識抽出が適切なタイミングで自動的にトリガーされるようにすべきである。

**3. 動的 Skill 発見と条件付き有効化**

ファイルパスに基づいて Skill を有効化するメカニズムは、Doramagic のプロジェクト固有の知識シナリオに非常に適している：ユーザーがあるドメインのファイルを操作するとき、対応するドメインの抽出 Skill を自動的に有効化する（例：TypeScript ファイルを操作するときにフロントエンドアーキテクチャ分析 Skill を有効化）。

**4. `files` メカニズムによる補助リソース管理**

組み込み Skill は `files` フィールドで参照ドキュメントやサンプルコードを Skill パッケージに埋め込み、モデルがコンテキストに一括注入するのではなく必要に応じて読み込む。Doramagic の大型 Skill（Soul Extractor など）はこのパターンを採用して抽出テンプレートと参考資料を管理できる。

**5. セキュリティモデル：allowedTools ホワイトリスト + 安全な Skill の自動許可**

Skill は frontmatter で宣言したツールのみ使用できる。Claude Code はさらに「安全な Skill」（特別な権限なし）と「確認が必要な Skill」（allowedTools/hooks あり）を区別し、前者を自動許可することで摩擦を低減する。この権限モデルは OpenClaw プラットフォームが参考にする価値がある。

---

## 6.10 ソースコードインデックス

| ファイル | 行数 | 役割 |
|------|------|------|
| `skills/loadSkillsDir.ts` | 1,087 | Skill 読み込みコア：発見、解析、重複排除、条件付き有効化、動的発見 |
| `skills/bundledSkills.ts` | 220 | 組み込み Skill レジストリ、ファイル展開、安全な書き込み |
| `tools/SkillTool/SkillTool.ts` | 1,108 | Skill 実行ツール：検証、権限、inline/fork 実行 |
| `skills/mcpSkillBuilders.ts` | 44 | MCP Skill ビルダーレジストリ（循環依存を解消） |
| `skills/bundled/updateConfig.ts` | 475 | update-config：settings.json 設定ヘルパー |
| `skills/bundled/scheduleRemoteAgents.ts` | 447 | schedule：スケジュール済みリモート agent 管理 |
| `skills/bundled/keybindings.ts` | 339 | keybindings-help：キーボードショートカット設定 |
| `skills/bundled/loremIpsum.ts` | 282 | lorem-ipsum：正確なトークンカウントの占位テキスト |
| `skills/bundled/skillify.ts` | 197 | skillify：セッションワークフローの自動 Skill 化 |
| `skills/bundled/claudeApi.ts` | 196 | claude-api：Claude API 開発ヘルパー（多言語対応） |
| `skills/bundled/claudeApiContent.ts` | 220 | claude-api の 247KB ドキュメントコンテンツ（ビルド時インライン化） |
| `skills/bundled/batch.ts` | 124 | batch：大規模並列 worktree 変更 |
| `skills/bundled/loop.ts` | 92 | loop：間隔指定でプロンプトを繰り返し実行 |
| `skills/bundled/remember.ts` | 82 | remember：memory の審査と昇格 |
| `skills/bundled/simplify.ts` | 69 | simplify：3 次元コードレビューと修正 |
| `skills/bundled/debug.ts` | 約 60 | debug：セッションデバッグログの診断 |
| `skills/bundled/stuck.ts` | 約 60 | stuck：プロセスフリーズ診断 + Slack レポート |
| `skills/bundled/verify.ts` | 約 30 | verify：アプリ実行によるコード変更の検証 |
| `skills/bundled/claudeInChrome.ts` | 約 40 | claude-in-chrome：Chrome ブラウザ制御 |
| `skills/bundled/index.ts` | - | 全組み込み Skill の登録エントリポイント |

**主要関数インデックス：**

| 関数 | ファイル:行番号 | 説明 |
|------|----------|------|
| `getSkillDirCommands` | `loadSkillsDir.ts:638` | メインの読み込みエントリポイント（memoized） |
| `parseSkillFrontmatterFields` | `loadSkillsDir.ts:184` | frontmatter フィールドの解析 |
| `createSkillCommand` | `loadSkillsDir.ts:269` | Command オブジェクトの構築 |
| `loadSkillsFromSkillsDir` | `loadSkillsDir.ts:407` | `/skills/` ディレクトリからの読み込み |
| `discoverSkillDirsForPaths` | `loadSkillsDir.ts:861` | Skill ディレクトリの動的発見 |
| `activateConditionalSkillsForPaths` | `loadSkillsDir.ts:997` | 条件付き Skill の有効化 |
| `registerBundledSkill` | `bundledSkills.ts:55` | 組み込み Skill の登録 |
| `executeForkedSkill` | `SkillTool.ts:121` | Fork モードでの実行 |
| `skillHasOnlySafeProperties` | `SkillTool.ts:871+` | 安全な Skill の判定 |
| `registerMCPSkillBuilders` | `mcpSkillBuilders.ts:31` | MCP ビルダーの登録 |
