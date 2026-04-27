# 第 9 章：メモリシステム

## 9.1 概要と位置づけ

Claude Code のメモリシステムは、ツールチェーン全体の中で最も精密に設計され、エンジニアリングが深く投入されたサブシステムの一つである。LLM の最も根本的な制限を解決している：コンテキストウィンドウはセッション終了と同時にゼロになる。ユーザーが新しいセッションを開始するたびに、Claude は白紙の状態に直面する——ユーザーが誰なのか、何を好むのか、前回どのようなミスをしたのか、チームにどのような規約があるのかを知らない。

メモリシステムの設計目標は：**Claude がセッションをまたいで継続性を保ち、真の長期的なコラボレーターのように行動できるようにすること**。

ソースコードの規模から見ると、これはかなりの大規模システムである：
- `memdir/` ディレクトリ：7 ファイル、1736 行
- `services/SessionMemory/`：3 ファイル、1026 行
- `services/extractMemories/`：2 ファイル、769 行
- `services/teamMemorySync/`：5 ファイル、2167 行

合計約 5700 行で、コードベース全体の約 1.1% を占めるが、その複雑さと設計思考の密度はこの比率をはるかに超えている。

---

## 9.2 理論的基礎

### 人間の記憶モデルとの対応

システムアーキテクチャは認知科学の三種の記憶に明確に対応している：

| 人間の記憶 | Claude Code の対応 | 技術実装 |
|---------|-----------------|---------|
| 作業記憶（Working Memory）| 現在のコンテキストウィンドウ | セッションメッセージリスト、セッション終了とともにクリア |
| エピソード記憶（Episodic Memory）| Session Memory | `~/.claude/projects/<slug>/session-memory.md`、セッション内で継続更新 |
| 意味記憶（Semantic Memory）| Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`、セッションをまたいで長期保存 |

Session Memory は「今この瞬間の記憶」に対応している——今回のセッションで何をしているか、どこまで進んでいるかを記録する。Persistent Memory は「蓄積された知識」に対応している——ユーザーの好み、フィードバックの教訓、プロジェクトの背景など。

### 知識グラフ vs ドキュメントメモリの選択

システムは**ファイルシステム上の Markdown ドキュメント**を選択しており、データベースやベクトルインデックスではない。この選択は `memoryTypes.ts` のコメントに明記されている：

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

これは第一原理を明らかにしている：**リアルタイムでクエリできる情報はメモリに保存すべきでない。** メモリには「派生不可能な」コンテキストのみを保存する——ユーザーの好み、チームの歴史的教訓、プロジェクトの背後にある動機など。これは知識グラフの設計とは根本的に異なり、後者は構造化できるすべての情報を組み込む傾向がある。

### 記憶における最終的一貫性の適用

Team Memory の同期設計は明示的に最終的一貫性のセマンティクスを採用している：

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

削除が伝播しないという設計は意図的なものである——チームメモリは「追記型」の資産であり、誤って削除しても永続的な損失にならないようにしている。これは分散システムの最終的一貫性原則の保守的な実装である。

---

## 9.3 三層メモリアーキテクチャ

システムは三層で構成されており、ライフサイクルが最も短いものから長いものへと順に並んでいる：

### 第一層：Session Memory（セッションレベル）

**ファイルパス**：`~/.claude/projects/<sanitized-cwd>/session-memory.md`（`getSessionMemoryPath()` で取得）

Session Memory は**現在のセッション内で継続的に維持される** Markdown ファイルであり、内容の構造は固定されている：

```markdown
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

（`services/SessionMemory/prompts.ts:14-36`、`DEFAULT_SESSION_MEMORY_TEMPLATE`）

セッション終了時にクリアされることはなく、Auto Compact 機構がコンテキストを圧縮する際に読み込まれ、「前回のあらすじ」として新しいコンテキストウィンドウに注入される。

**データ構造の制約**：
- 各セクションの上限は 2000 tokens（`MAX_SECTION_LENGTH = 2000`）
- 全文の上限は 12000 tokens（`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`）
- 制限を超えた場合、システムはプロンプトに警告を追加し、Agent に圧縮を要求する

**ライフサイクル**：現在のプロジェクトセッションに紐づき、Auto Compact トリガー時に読み込まれる

### 第二層：Persistent Memory（セッションをまたぐ永続メモリ）

**ファイルパス**：`~/.claude/projects/<sanitized-git-root>/memory/`

これが核心的な長期メモリ層である。各メモリは YAML frontmatter を持つ独立した `.md` ファイルとして保存される：

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

（`memdir/memoryTypes.ts:230-240`、`MEMORY_FRONTMATTER_EXAMPLE`）

パス解析ロジックは `getAutoMemPath()` が担当する（`memdir/paths.ts:173-190`）。解析の優先順位は：

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 環境変数（Cowork 複数ユーザーシナリオで使用）
2. `settings.json` の `autoMemoryDirectory`（policy/local/user 由来のみ信頼、**projectSettings は信頼しない**——悪意ある repo が書き込みパスをハイジャックするのを防ぐため）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`（デフォルト）

Git の作業ツリーは canonical git root（`findCanonicalGitRoot`）に統一され、同一リポジトリの異なる worktree が同じメモリを共有できる。

**ライフサイクル**：永続的、ユーザーが明示的に削除するか Agent が積極的に更新/削除するまで

### 第三層：Team Memory（チーム共有メモリ）

**ファイルパス**：`~/.claude/projects/<sanitized-git-root>/memory/team/`（`getTeamMemPath()` の返り値）

Team Memory は Persistent Memory のサブディレクトリであり、REST API を通じて同一 GitHub リポジトリの認証済みメンバー間で同期される。これは Auto Memory の上位拡張であり、`isTeamMemoryEnabled()` はまず `isAutoMemoryEnabled()` をチェックして親システムが有効であることを確認する。

**ライフサイクル**：Anthropic のサーバーサイドで維持され、ユーザー間・マシン間で永続化される

---

## 9.4 MEMORY.md インデックス機構

MEMORY.md は Persistent Memory 層の**インデックスファイル**であり、コンテンツファイルではない。システムの複数の箇所でこの二つを明確に区別している：

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### フォーマット規範

MEMORY.md の各行は具体的なメモリファイルへのリンクである：

```
- [ユーザー設定：簡潔な返答](feedback_terse_responses.md) — ユーザーは返答の末尾にまとめを書くことを好まない
- [プロジェクトの背景：Auth ミドルウェアの書き直し](project_auth_rewrite.md) — 法務コンプライアンスの要件、技術的負債ではない
```

MEMORY.md は各セッションの開始時にシステムプロンプトに読み込まれるため、そのサイズは各リクエストの token 消費に直接影響する。

### 200 行 / 25KB の二重制限

システムは `memdir/memdir.ts` で厳格な二重上限を定義している：

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

（`memdir/memdir.ts:30-33`）

切り捨てロジックは `truncateEntrypointContent()` に実装されている（`memdir/memdir.ts:55-102`）：まず行数で切り捨て、次にバイト数で切り捨て（最後の改行位置で切り捨て、行の途中で切れないようにする）。切り捨て後、インデックスが長すぎることをユーザーに通知する警告メッセージが追加される。

**設計意図**：1 行約 125 文字 × 200 行 ≈ 25KB は実測（p97 分位）に基づく合理的な上限である。バイト制限は「200 行未満でも各行が極めて長い」エッジケースに対応する（実測 p100：197KB で行数制限を超えない）。

### メモリファイルとの関係

メモリへの書き込みは**二段階操作**である：
1. コンテンツファイルを書き込む（`user_role.md`、`feedback_testing.md` など）
2. MEMORY.md にその entry を追加する

読み込み時は findRelevantMemories で選択されたファイルのみが読み込まれる（詳細は 9.7 参照）。MEMORY.md 自体はシステムプロンプトに常駐する。

---

## 9.5 四種のメモリタイプ

システムはすべてのメモリを四種のタイプに制約している。これは設計上最も重要な決定の一つである。タイプの定義は `memdir/memoryTypes.ts`（`MEMORY_TYPES` 定数）にある：

### user タイプ

**適用シナリオ**：ユーザーの役割、目標、責任、知識背景

**トリガータイミング**：ユーザーの役割、好み、職務、知識レベルを把握したとき

**用途**：特定のユーザーの認知レベルとニーズに合わせた返答方法を調整する

**スコープ**：常に private（個人所有）、Team Memory モードでも同様

**保存すべきでない内容（反例）**：
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### feedback タイプ

**適用シナリオ**：作業方法に対するユーザーの修正と確認——「こうしないで」という指摘と「このまま続けて」という確認の両方

**構造要件**：
- ルール本体
- `**Why:**` 行（理由を示し、エッジケースで適用すべきか判断できるようにする）
- `**How to apply:**` 行（いつどこで有効か）

**独自の設計**：**失敗の教訓と成功の確認の両方**を明示的に記録することを求める：

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**トリガータイミング**：ユーザーが「こうしないで」と言う（明示的な修正）か、「まさにそれ」「完璧」と言う（暗黙の確認、より難しく認識しにくい）

**スコープ**：デフォルトは private。ガイドラインが明確にプロジェクトレベルの規範（テスト戦略、ビルド制約など）である場合のみ team として保存する

### project タイプ

**適用シナリオ**：進行中の作業、目標、計画、バグ、出来事に関する情報で、**コードや git 履歴から派生できないもの**

**構造要件**：
- 事実/決定そのもの
- `**Why:**` 行（動機——通常は制約、締め切り、ステークホルダーの要件）
- `**How to apply:**` 行（提案にどう影響するか）

**重要ルール**：保存時に相対的な日付を絶対的な日付に変換する（「来週木曜日」→「2026-04-08」）。時間が経過してもメモリを解釈できるようにするためだ。

**スコープ**：デフォルトは team（プロジェクトのコンテキストは本質的に共有される）

**減衰特性**：project タイプのメモリは最も速く減衰する。Why フィールドはメモリがまだ有効かどうかの判断に役立つ。

### reference タイプ

**適用シナリオ**：外部システムの情報の場所へのポインター（Linear プロジェクト、Slack チャンネル、Grafana ダッシュボードなど）

**トリガータイミング**：外部リソースの場所とその用途を知ったとき

**スコープ**：通常は team

**典型的な例**：

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### 保存すべきでない内容（明示的な除外）

`WHAT_NOT_TO_SAVE_SECTION` は保存すべきでない六種のコンテンツを明記している（`memdir/memoryTypes.ts:196-207`）：

1. コードパターン、規約、アーキテクチャ、ファイルパス——プロジェクトの現在の状態から派生可能
2. Git 履歴、最近の変更——`git log`/`git blame` が権威ある情報源
3. デバッグの解決策や修正方法——修正はコードの中にあり、コンテキストはコミットメッセージの中にある
4. CLAUDE.md に既に記録されている内容
5. 一時的なタスクの詳細：進行中の作業、一時的な状態、現在のセッションのコンテキスト
6. **ユーザーが明示的に保存を求めた上記の内容でも**——PR リストを保存するよう求められた場合は「意外なこと、明らかでないことはありますか？それこそが保存する価値のあることです」と問うべきだ

---

## 9.6 自動メモリ抽出

### Fork Agent による自動抽出機構

メモリ抽出は「Fork Agent」パターンを使用している——メインセッションと完全に同じ Agent コンテキストを作成し、バックグラウンドで非同期に実行し、メインの会話フローをブロックしない。

この機構の核心は `runForkedAgent()` であり、抽出 Agent が親セッションの prompt cache を共有することでキャッシュヒット率を最大化する：

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // メインセッションの記録に書き込まない、競合状態を防ぐ
  maxTurns: 5,            // ハード上限、検証の無限ループを防ぐ
})
```

（`services/extractMemories/extractMemories.ts:258-267`）

`maxTurns: 5` の設計コメントは意図を説明している：

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

抽出 Agent の効率的な戦略は「2 ターンで完了」と明示的に設計されている：
- **第 1 ターン**：更新が必要なすべてのファイルへの FileRead リクエストを並行して発行
- **第 2 ターン**：すべての FileWrite/FileEdit リクエストを並行して発行

### トリガータイミング（Stop Hooks）

抽出は**完全なクエリループが終了するたび**にトリガーされる——つまりモデルが tool_use を含まない最終応答を生成したとき、`handleStopHooks` を通じて `executeExtractMemories()` を呼び出す。

状態はクロージャで管理されており、主要な変数は：

```typescript
let lastMemoryMessageUuid: string | undefined    // カーソル：前回どこまで抽出したか
let inProgress = false                           // 並行実行を防ぐ
let pendingContext: {...} | undefined            // 実行中に到着した呼び出しをここに保存
let turnsSinceLastExtraction = 0                // スロットリング制御に使用
```

（`services/extractMemories/extractMemories.ts:225-240`）

**並行制御戦略**：抽出の進行中に新しい呼び出しが来た場合、新しい呼び出しは「スタッシュ」（`pendingContext` に保存）され、破棄されない。現在の抽出が完了した後、最新のコンテキストで「トレイリング抽出」を即座に実行し、最後のメッセージ群が見逃されないようにする。

**相互排他ルール**：メイン Agent 自身がメモリファイルを書き込んだ場合（`hasMemoryWritesSince` で検出）、Fork Agent はその抽出をスキップし、カーソルのみ進める：

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // メイン Agent が書き込んだ、fork agent をスキップしてカーソルを進める
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

（`services/extractMemories/extractMemories.ts:198-209`）

### 抽出プロンプトの分析

抽出プロンプトの核心的な設計哲学は**情報効率**である：

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // 既存メモリリストを事前注入し、Agent が ls に 1 ターン使うのを避ける
  ].join('\n')
}
```

（`services/extractMemories/prompts.ts:20-47`）

既存メモリのリスト（`existingMemories`）を事前注入することが重要な最適化である——Agent がディレクトリ一覧を取得するために 1 ターンの呼び出しを無駄にすることなく、プロンプト内で構造化されたファイルリスト（ファイル名、タイプ、タイムスタンプ、description）を直接確認できる。

### Session Memory のトリガー機構

Session Memory は異なるトリガー機構を使用する——Stop Hooks ではなく `postSamplingHooks` を通じて、モデルがサンプリングするたびに更新が必要かどうかを評価する：

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

（`services/SessionMemory/sessionMemory.ts:130-150`）

デフォルトのトリガー閾値（`DEFAULT_SESSION_MEMORY_CONFIG`、`services/SessionMemory/sessionMemoryUtils.ts:29-33`）：

| パラメータ | デフォルト値 | 説明 |
|-----|-------|------|
| `minimumMessageTokensToInit` | 10,000 | Session Memory の初期化に必要な最低 token 数 |
| `minimumTokensBetweenUpdate` | 5,000 | 2 回の更新間で最低限増加する必要がある token 数 |
| `toolCallsBetweenUpdates` | 3 | 2 回の更新間で最低限必要な tool 呼び出し回数 |

これらの値は GrowthBook リモート設定（`tengu_sm_config`）で動的に調整可能である。

---

## 9.7 スマートなメモリ召喚

### Sonnet が最大 5 件の関連メモリを選択

メモリの召喚は全量読み取りではなく、**まず frontmatter をスキャンし、次に Sonnet を使って最も関連する最大 5 件を選択する**。

核心的なフローは `findRelevantMemories()`（`memdir/findRelevantMemories.ts:32-66`）にある：

1. `scanMemoryFiles()` がメモリディレクトリをスキャンし、各ファイルの最初の 30 行（frontmatter）を読み込んで `MemoryHeader[]` を返す
2. 前のターンで既に表示されたメモリをフィルタリングする（`alreadySurfaced`）、5 枠を新しいコンテンツのために確保する
3. Sonnet を使って `selectRelevantMemories()` を呼び出し、クエリとファイルの description に基づいて最も関連するファイル名を選択する
4. 選択されたメモリのパスと mtime を返す

### 関連性判断ロジック

Sonnet のシステムプロンプトは慎重に設計されている：

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

（`memdir/findRelevantMemories.ts:13-23`）

**重要な設計点**：最近使用されたツールのリファレンスドキュメントは選択すべきでない（使用中はリファレンスが不要）が、同じツールの**落とし穴/既知の問題**のメモリは依然として選択すべきである（使用中が最もその警告を必要とするタイミング）。

API 呼び出しは構造化出力（JSON Schema）を使用して、返り値のフォーマットを解析可能にする：

```typescript
output_format: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      selected_memories: { type: 'array', items: { type: 'string' } },
    },
    required: ['selected_memories'],
    additionalProperties: false,
  },
},
```

（`memdir/findRelevantMemories.ts:97-108`）

### コンテキストへのメモリ注入方法

選択されたメモリは `<system-reminder>` タグで包まれ、ユーザーメッセージの前に注入される（`wrapMessagesInSystemReminder`）。1 日を超えたメモリには鮮度警告が付加される：

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

（`memdir/memoryAge.ts:38-47`）

この設計は実際の問題を解決している：ユーザーから「古いメモリに基づいて自信を持って断言する」という報告があった——参照されたファイルパスや関数名は既に変更されているが、メモリ内の引用が断言をより信頼できるように、あるいはより疑わしく見えるようにしていた。

**ドリフト防止機構**：`MEMORY_DRIFT_CAVEAT` がシステムプロンプトに注入され、Agent がメモリに基づいて回答する前に現在の状態を確認することを求める：

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Team Memory の同期

### REST API 同期機構

Team Memory は `services/teamMemorySync/` を通じてサーバーサイドの同期を実現する。API の設計は `index.ts` の先頭に完全な説明がある：

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → メタデータ + ハッシュのみ
PUT  /api/claude_code/team_memory?repo={owner/repo}            → エントリをアップサート
404  = データがまだない
```

（`services/teamMemorySync/index.ts:10-13`）

同期は **OAuth 認証**（`CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE` が必要）に依存し、GitHub リポジトリ（`owner/repo`）をスコープとして使用する。

**Watcher 機構**：`watcher.ts` は `fs.watch({recursive: true})` を使用して team ディレクトリの変更を監視し、2 秒のデバウンス後に push をトリガーする（`DEBOUNCE_MS = 2000`）。意図的に chokidar ではなくネイティブの `fs.watch` を選択している：

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS は FSEvents（O(1) ファイルディスクリプター）、Linux は inotify（O(サブディレクトリ数)）を使用し、どちらも chokidar の kqueue アプローチより優れている。

### 楽観的ロック（If-Match）

アップロードは楽観的な並行制御を使用し、`If-Match` HTTP ヘッダーで ETag（checksum）を送信する：

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

（`services/teamMemorySync/index.ts:uploadTeamMemory`）

サーバーが 412 Precondition Failed を返した場合、競合が発生していることを意味する（その間に別のユーザーが共有メモリを変更した）。システムは `GET ?view=hashes` エンドポイント（軽量、各 key の SHA-256 ハッシュのみを返す、コンテンツ本体なし）を使ってローカルの `serverChecksums` を更新し、デルタを再計算してリトライする：

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### 競合解決戦略

競合解決の戦略は**サーバーが勝つ（server wins per-key）**——Pull 時にサーバーコンテンツがローカルを上書きする。デルタ push はローカルとサーバーのハッシュが異なる key のみアップロードし、サーバーはアップサートセマンティクスを使用する（PUT に含まれない key は保持される）。

バッチアップロード制限（`MAX_PUT_BODY_BYTES = 200_000`）はリクエストボディが大きすぎて API Gateway に拒否されるのを防ぐ（約 256〜512KB 時に HTML 形式の 413 を返すことが観測されており、アプリケーション層の構造化 413 とは異なる）。制限を超えた場合、複数の順序付き PUT に自動分割し、アップサートセマンティクスで安全を保証する：

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // 貪欲なビンパッキング：バイト数でバッチ分割、各バッチは MAX_PUT_BODY_BYTES を超えない
  ...
}
```

（`services/teamMemorySync/index.ts:batchDeltaByBytes`）

**永続的な失敗の抑制**：一部のエラー（no_oauth、no_repo、4xx（409/429 以外））はリトライで自然回復できない。システムがこれらを検出すると `pushSuppressedReason` を設定し、watcher がトリガーする push が無限リトライループに陥るのを防ぐ（OAuth なしのデバイスが 2.5 日間で 167K の push イベントを発行したことが観測されていた）。

---

## 9.9 設計上の意思決定分析

### なぜデータベースではなくファイルシステムを使うのか

ファイルシステム + Markdown の設計にはいくつかの重要な利点がある：

1. **Agent が直接操作できる**：FileRead/FileWrite/FileEdit ツールは Claude のネイティブツールであり、追加の API 層が不要。Agent がメモリを書くこととコードを書くことは同じツールを使用するため、認知的負荷が最も低い。

2. **ユーザーが確認できる**：`~/.claude/projects/.../memory/` は通常のフォルダであり、ユーザーは直接 `ls`、`cat`、`vim` を使用でき、完全に透明。

3. **Git フレンドリー**：Markdown ファイルは diff、grep、git history をネイティブにサポートし、Team Memory のデルタ計算が容易。

4. **不必要な抽象化を避ける**：データベースはスキーママイグレーション、バックアップ戦略、クエリ層が必要——「数百 KB の Markdown ファイル」に対しては過剰エンジニアリングである。

### なぜ MEMORY.md のサイズを制限するのか

200 行 / 25KB の制限には実測データの裏付けがある（p97/p100 の観測値）。核心的な理由：

- MEMORY.md は**リクエストごとに**システムプロンプトに読み込まれ、そのサイズは token 消費に直接影響する
- インデックスが大きすぎると、本当に有用なコンテキストのスペースを圧迫する
- 強制的な制限により、ユーザーと Agent はインデックスを精炼に保つ動機づけがされる。各行には「フック」のみを書き、内容は書かない

これは「制約で品質を促進する」典型的な設計だ——技術的に容量を持てないからではなく、制約を通じて正しい使用方法に誘導するためである。

### メモリセキュリティの設計考慮

システムには複数の階層的なセキュリティ設計がある：

**パストラバーサル保護**：`teamMemPaths.ts` は三層のチェックを実装している——まず文字列レベルで `..`、URL エンコードのトラバーサル、Unicode 正規化攻撃を確認し、次に `realpath` でシンボリックリンクを解決して実際のファイルシステムパスを検証する：

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

（`memdir/teamMemPaths.ts:130-133`）

**シークレットスキャン**：Team Memory への書き込み時に、`scanForSecrets()` が 30 種の高信頼度の認証情報パターン（gitleaks ルールライブラリから）をスキャンする。AWS、GCP、GitHub、Anthropic、OpenAI などの主要プラットフォームの token フォーマットが含まれる。スキャンは**アップロード前**と**書き込み前**の二重実行：

- `teamMemSecretGuard.ts` の `checkTeamMemSecrets()` が FileWriteTool/FileEditTool の `validateInput` 段階で書き込みをインターセプト
- `readLocalTeamMemory()` が push 前に再度スキャンし、機密情報を含むファイルをスキップ

**最小権限ツール制御**：抽出 Agent の `canUseTool` 関数は以下のみを許可：
- FileRead/Grep/Glob（読み取り専用）
- 読み取り専用 Bash コマンド（ls/find/cat/stat/wc/head/tail）
- memory ディレクトリ内のパスに限定した FileEdit/FileWrite

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

（`services/extractMemories/extractMemories.ts:171-176`）

**ProjectSettings のセキュリティ免除**：`autoMemoryDirectory` の設定は policy/local/user 由来のみを信頼し、projectSettings（`.claude/settings.json`）を明示的に除外している：

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 移植可能なパターン

以下は Doramagic のメモリシステム設計に直接参考になるパターンである：

### パターン 1：派生不可能原則

**何をメモリに保存すべきか**：現在の状態（コード、ファイル、git）をクエリして得られる情報はすべてメモリに保存する価値がない。メモリには「歴史的なコンテキスト」のみを保存する——なぜその決定をしたのか、どんな落とし穴にはまったのか、ユーザーの暗黙の好みなど。

**Doramagic への応用**：Soul Extractor が抽出する「UNSAID」と「WHY」層は、この原則に自然に合致している。OpenClaw のルールドキュメントはクエリ可能であり、メモリに保存する必要はない。しかし「この OpenClaw ルールがかつてリリース失敗を引き起こした」という教訓こそ、メモリに保存する価値がある。

### パターン 2：二段階書き込み + 軽量インデックス

ファイル + インデックスの二段階書き込みパターンにより、インデックスは常に精炼に保たれる（強制的に各行 150 文字以内）。コンテンツファイルは詳細を展開できる。インデックスの token 消費は固定されており、コンテンツの読み込みはオンデマンドである。

**Doramagic への応用**：メモリシステムの `MEMORY.md` は Doramagic の「積木目録」に似ている——軽量で読み込み可能なインデックス、詳細なファイルをオンデマンドで展開できる。

### パターン 3：Fork Agent によるバックグラウンド抽出

メインの会話をブロックせず、prompt cache を共有し、キャッシュヒット率を最大化する——これはバックグラウンド後処理タスクの標準パターンである。重要な実装の詳細：
- `skipTranscript: true` でメインセッションの記録への書き込みを回避
- `maxTurns: N` で Agent が検証ループに陥るのを防ぐ
- カーソル機構（`lastMemoryMessageUuid`）で毎回増分のみを処理
- Stash + trailing run で Agent が忙しいときも最新メッセージを見逃さない

### パターン 4：鮮度認識

メモリは永続的な事実ではなく、有効期限のある観察である。システムは以下の方法で対処する：
1. 召喚時に「N 日前」の経過時間ヒントを付加
2. システムプロンプトにドリフト防止指令を埋め込む（引用前に検証）
3. Agent に古いメモリを発見したときに積極的に更新させ、保持させない

これは Doramagic の「知識抽出」シナリオに特に関連している——抽出された WHY/UNSAID はプロジェクトの進化とともに古くなる可能性があり、鮮度を維持するための同様の機構が必要だ。

### パターン 5：シークレットスキャンの前置き

「境界をまたいだ」書き込み（共有スペースへの書き込み、ネットワークアップロード）の前には、必ずシークレットをスキャンすべきである。gitleaks ルールライブラリは高信頼度のパターンセットを提供しており、直接再利用できる。重要な設計：スキャンは書き込みツールの `validateInput` 段階で実行する（事後ではなく）。これによりシークレットが永続化パスに触れることを防げる。

---

## 9.11 ソースコードインデックス

| ファイル | 行数 | 核心的な責任 |
|-----|------|---------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Session Memory メインロジック：トリガー条件の判断、Fork Agent の呼び出し、手動トリガー API |
| `services/SessionMemory/prompts.ts` | 324 | Session Memory テンプレート、更新プロンプトの構築、セクションサイズの分析 |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Session Memory の状態管理：設定、閾値の判断、待機/同期ユーティリティ関数 |
| `services/extractMemories/extractMemories.ts` | 615 | Persistent Memory の抽出：Fork Agent の呼び出し、クロージャ状態、並行制御 |
| `services/extractMemories/prompts.ts` | 154 | 抽出プロンプトの構築：auto-only と combined（Team Memory を含む）の二変体 |
| `memdir/memdir.ts` | 507 | MEMORY.md の切り捨てロジック、メモリプロンプトの構築、ディレクトリの作成保証 |
| `memdir/paths.ts` | 278 | Auto Memory のパス解析、有効/無効の判断、パスのセキュリティチェック |
| `memdir/memoryTypes.ts` | 271 | 四種のメモリタイプの定義、frontmatter のフォーマット、召喚/ドリフト防止/派生不可能原則 |
| `memdir/findRelevantMemories.ts` | 141 | Sonnet による召喚選択：frontmatter のスキャン → 5 件の関連メモリ |
| `memdir/memoryScan.ts` | 94 | ディレクトリスキャンのプリミティブ：frontmatter の読み込み、リストのフォーマット |
| `memdir/memoryAge.ts` | 53 | 鮮度の計算：日数、human-readable テキスト、staleness 警告 |
| `memdir/teamMemPaths.ts` | 292 | Team Memory のパス、パストラバーサル保護（三層検証）、シンボリックリンクの解決 |
| `memdir/teamMemPrompts.ts` | 100 | Team Memory + Auto Memory のマージプロンプト構築 |
| `services/teamMemorySync/index.ts` | 1256 | 同期コア：fetch/push ロジック、楽観的ロック、バッチ分割、競合リトライ |
| `services/teamMemorySync/watcher.ts` | 387 | ファイル監視：デバウンス push、永続的な失敗の抑制、起動/停止ライフサイクル |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 種のシークレットスキャンルール（gitleaks のサブセット）、redact ユーティリティ関数 |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema：TeamMemoryData、同期結果タイプ、SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | 書き込み前シークレットのインターセプト：FileWriteTool/FileEditTool の validateInput 統合 |

**主要定数クイックリファレンス**：

| 定数 | 値 | 場所 |
|-----|---|------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH`（Session Memory 各セクション）| 2,000 tokens | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 tokens | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10,000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5,000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| 召喚上限 | 5 件 | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| メモリファイル数の上限 | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Frontmatter 読み込み行数 | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Team Memory タイムアウト | 30,000ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Push デバウンス遅延 | 2,000ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| 単一ファイルサイズ上限 | 250,000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| PUT リクエストボディ上限 | 200,000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
