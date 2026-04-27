# Глава 9: Система памяти

## 9.1 Обзор и позиционирование

Система памяти Claude Code — одна из наиболее тщательно спроектированных и глубоко проработанных подсистем всего инструментария. Она решает фундаментальное ограничение LLM: контекстное окно обнуляется по завершении сессии. При каждом новом сеансе Claude начинает с чистого листа — не знает, кто пользователь, каковы его предпочтения, какие ошибки были допущены ранее и какие нормы приняты в команде.

Цель системы памяти: **обеспечить непрерывность Claude между сессиями, чтобы он действовал как настоящий долгосрочный соавтор.**

По объёму исходного кода это весьма масштабная система:
- Каталог `memdir/`: 7 файлов, 1736 строк
- `services/SessionMemory/`: 3 файла, 1026 строк
- `services/extractMemories/`: 2 файла, 769 строк
- `services/teamMemorySync/`: 5 файлов, 2167 строк

Итого около 5700 строк — примерно 1,1% всей кодовой базы, однако сложность и плотность проектных решений значительно превышают эту долю.

---

## 9.2 Теоретическая основа

### Соответствие модели человеческой памяти

Архитектура системы явно соответствует трём видам памяти из когнитивной науки:

| Человеческая память | Аналог в Claude Code | Техническая реализация |
|---------------------|---------------------|----------------------|
| Рабочая память (Working Memory) | Текущее контекстное окно | Список сообщений сессии, очищается по завершении |
| Эпизодическая память (Episodic Memory) | Session Memory | `~/.claude/projects/<slug>/session-memory.md`, постоянно обновляется в рамках сессии |
| Семантическая память (Semantic Memory) | Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`, долгосрочное хранение между сессиями |

Session Memory соответствует «воспоминаниям текущего момента» — что делается в этой сессии и на каком этапе; Persistent Memory — «накопленным знаниям»: предпочтениям пользователя, урокам из ошибок, контексту проекта.

### Выбор между графом знаний и документарной памятью

Система выбрала **Markdown-документы в файловой системе** вместо базы данных или векторного индекса. Это решение явно изложено в комментарии к `memoryTypes.ts`:

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

Это раскрывает первопринцип: **информация, которую можно получить в реальном времени, не должна запоминаться.** Память хранит только «невыводимый» контекст — предпочтения пользователя, исторические уроки команды, мотивацию за решениями проекта. Это принципиально отличается от подхода графа знаний, который склонен структурировать и сохранять всё поддающееся структурированию.

### Применение eventual consistency в памяти

Дизайн синхронизации Team Memory явно использует семантику итоговой согласованности (eventual consistency):

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

Решение не распространять удаления принято намеренно — командная память является «только пополняемым» активом, случайные удаления не должны приводить к необратимым потерям. Это консервативная реализация принципа eventual consistency в распределённых системах.

---

## 9.3 Трёхуровневая архитектура памяти

Система состоит из трёх уровней, упорядоченных по длительности жизненного цикла от наименьшей к наибольшей:

### Первый уровень: Session Memory (уровень сессии)

**Путь к файлу**: `~/.claude/projects/<sanitized-cwd>/session-memory.md` (через `getSessionMemoryPath()`)

Session Memory — это Markdown-файл с **фиксированной структурой**, который непрерывно поддерживается в рамках **текущей сессии**:

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

(`services/SessionMemory/prompts.ts:14-36`, `DEFAULT_SESSION_MEMORY_TEMPLATE`)

Файл не очищается по завершении сессии, а считывается механизмом Auto Compact при сжатии контекста и вводится в новое контекстное окно в качестве «ранее в этом сезоне».

**Ограничения структуры данных**:
- Максимум 2000 токенов на секцию (`MAX_SECTION_LENGTH = 2000`)
- Максимум 12000 токенов для всего документа (`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`)
- При превышении лимитов система добавляет предупреждение в prompt и требует от Agent сжать содержимое

**Жизненный цикл**: привязан к текущей проектной сессии, считывается при срабатывании Auto Compact

### Второй уровень: Persistent Memory (долгосрочная межсессионная память)

**Путь к файлу**: `~/.claude/projects/<sanitized-git-root>/memory/`

Это ключевой уровень долгосрочной памяти. Каждый фрагмент памяти хранится как отдельный `.md`-файл с YAML frontmatter:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

(`memdir/memoryTypes.ts:230-240`, `MEMORY_FRONTMATTER_EXAMPLE`)

Логика разрешения пути реализована в `getAutoMemPath()` (`memdir/paths.ts:173-190`), приоритет разрешения:

1. Переменная окружения `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` (используется в сценарии Cowork с несколькими пользователями)
2. `autoMemoryDirectory` в `settings.json` (доверяются только источники policy/local/user, **не доверяется** projectSettings — для защиты от угона пути записи злоумышленным репозиторием)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` (по умолчанию)

Git worktree унифицируется к каноническому git root (`findCanonicalGitRoot`), чтобы разные worktree одного репозитория использовали общую память.

**Жизненный цикл**: постоянный, до явного удаления пользователем или обновления/удаления Agent

### Третий уровень: Team Memory (общая командная память)

**Путь к файлу**: `~/.claude/projects/<sanitized-git-root>/memory/team/` (возвращаемый `getTeamMemPath()`)

Team Memory — поддиректория Persistent Memory, синхронизируемая через REST API между всеми аутентифицированными участниками одного репозитория GitHub. Это расширение над Auto Memory; `isTeamMemoryEnabled()` сначала проверяет `isAutoMemoryEnabled()`, чтобы убедиться в активности родительской системы.

**Жизненный цикл**: поддерживается сервером Anthropic, персистентен между пользователями и машинами

---

## 9.4 Механизм индекса MEMORY.md

MEMORY.md — **индексный файл** уровня Persistent Memory, а не файл с содержимым. Система явно разграничивает эти два понятия в нескольких местах:

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### Требования к формату

Каждая строка MEMORY.md — ссылка на конкретный файл памяти:

```
- [Предпочтения пользователя: краткие ответы](feedback_terse_responses.md) — пользователь не любит итогов в конце ответов
- [Контекст проекта: переписка Auth middleware](project_auth_rewrite.md) — требование юридического соответствия, не технический долг
```

MEMORY.md загружается в системный prompt в начале каждой сессии, поэтому его размер напрямую влияет на потребление токенов при каждом запросе.

### Двойное ограничение: 200 строк / 25 КБ

Система определяет строгие двойные ограничения в `memdir/memdir.ts`:

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

(`memdir/memdir.ts:30-33`)

Логика усечения реализована в `truncateEntrypointContent()` (`memdir/memdir.ts:55-102`): сначала по количеству строк, затем по байтам (срез по ближайшему символу перевода строки, чтобы не разрывать строки). После усечения добавляется предупреждение о том, что индекс слишком длинный.

**Замысел**: примерно 125 символов × 200 строк ≈ 25 КБ — разумный предел, подтверждённый измерениями (p97 перцентиль). Ограничение в байтах учитывает граничный случай «менее 200 строк, но каждая очень длинная» (реально измеренный p100: 197 КБ без превышения лимита строк).

### Отношение к файлам памяти

Запись памяти — это **двухшаговая операция**:
1. Записать файл с содержимым (`user_role.md`, `feedback_testing.md` и т.д.)
2. Добавить указывающую запись в MEMORY.md

При чтении только файлы, выбранные `findRelevantMemories`, загружаются для просмотра (подробнее в 9.7), тогда как сам MEMORY.md постоянно присутствует в системном prompt.

---

## 9.5 Четыре типа памяти

Система ограничивает все записи четырьмя типами — одно из важнейших проектных решений. Типы определены в `memdir/memoryTypes.ts` (константа `MEMORY_TYPES`):

### Тип user

**Применимые сценарии**: роль пользователя, цели, обязанности, уровень знаний

**Когда записывать**: при любом узнавании роли, предпочтений, обязанностей или уровня компетентности пользователя

**Назначение**: адаптация способа ответа к когнитивному уровню и потребностям конкретного пользователя

**Область**: всегда private (личное), даже в режиме Team Memory

**Что не следует сохранять**:
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### Тип feedback

**Применимые сценарии**: исправления и подтверждения пользователем способа работы — как «не делай так», так и «продолжай делать вот так»

**Требования к структуре**:
- Само правило
- Строка `**Why:**` (объяснение, чтобы в пограничных случаях понимать применимость)
- Строка `**How to apply:**` (когда и где действует)

**Особый дизайн**: явно требует записывать как **уроки из ошибок, так и подтверждения успехов**:

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**Когда записывать**: пользователь говорит «не делай так» (явное исправление) или «именно так» / «отлично» (неявное подтверждение, сложнее идентифицировать)

**Область**: по умолчанию private; только если рекомендации явно являются нормами уровня проекта (например, стратегии тестирования, ограничения сборки), сохраняется как team

### Тип project

**Применимые сценарии**: информация о текущей работе, целях, планах, багах или событиях, которую **невозможно вывести из кода или истории git**

**Требования к структуре**:
- Факт/решение
- Строка `**Why:**` (мотивация — обычно ограничения, дедлайны или требования стейкхолдеров)
- Строка `**How to apply:**` (как это влияет на рекомендации)

**Важное правило**: при сохранении относительные даты преобразуются в абсолютные («в следующий четверг» → «2026-04-08»), чтобы запись оставалась интерпретируемой спустя время.

**Область**: по умолчанию team (контекст проекта по природе своей является общим)

**Характеристика устаревания**: тип project устаревает быстрее всего, поле Why помогает судить о том, остаётся ли запись действительной.

### Тип reference

**Применимые сценарии**: указатели на расположение информации во внешних системах (проекты Linear, каналы Slack, дашборды Grafana и т.д.)

**Когда записывать**: при узнавании местоположения внешнего ресурса и его назначения

**Область**: обычно team

**Типичный пример**:

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### Что не следует сохранять (явные исключения)

`WHAT_NOT_TO_SAVE_SECTION` явно перечисляет шесть категорий, которые не нужно сохранять (`memdir/memoryTypes.ts:196-207`):

1. Паттерны кода, соглашения, архитектура, пути к файлам — выводимы из текущего состояния проекта
2. История git, последние изменения — `git log`/`git blame` являются авторитетным источником
3. Решения при отладке или способы исправления — исправление в коде, контекст в сообщении коммита
4. Уже задокументированное в CLAUDE.md
5. Временные детали задачи: текущая работа, временное состояние, контекст текущей сессии
6. **Всё вышеперечисленное, даже если пользователь явно просит сохранить** — если пользователь просит сохранить список PR, следует спросить «есть ли что-то неожиданное или неочевидное? Только это и стоит сохранять»

---

## 9.6 Автоматическое извлечение памяти

### Механизм автоматического извлечения Fork Agent

Извлечение памяти использует паттерн «Fork Agent» — создаётся экземпляр Agent с тем же контекстом, что у основной сессии, и запускается асинхронно в фоне, не блокируя основной диалог.

Ядро механизма — `runForkedAgent()`; извлекающий Agent разделяет prompt cache родительской сессии, достигая максимального процента попаданий в кэш:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // не записываем в лог основной сессии, избегаем гонок
  maxTurns: 5,            // жёсткий лимит для предотвращения петли верификации
})
```

(`services/extractMemories/extractMemories.ts:258-267`)

Комментарий к `maxTurns: 5` поясняет намерение:

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

Эффективная стратегия извлечения явно спроектирована как «завершить за 2 хода»:
- **Ход 1**: параллельно отправить запросы FileRead для всех файлов, которые нужно обновить
- **Ход 2**: параллельно отправить все запросы FileWrite/FileEdit

### Момент срабатывания (Stop Hooks)

Извлечение срабатывает **после каждого полного цикла запроса** — то есть когда модель выдаёт финальный ответ без tool_use — через `handleStopHooks`, вызывающий `executeExtractMemories()`.

Состояние управляется через замыкание, ключевые переменные:

```typescript
let lastMemoryMessageUuid: string | undefined    // курсор: до какого сообщения извлечено
let inProgress = false                           // защита от конкурентного запуска
let pendingContext: {...} | undefined            // вызовы, пришедшие во время работы
let turnsSinceLastExtraction = 0                // для управления throttle
```

(`services/extractMemories/extractMemories.ts:225-240`)

**Стратегия контроля конкурентности**: если при активном извлечении поступает новый вызов, он «прячется» (помещается в `pendingContext`), а не отбрасывается. После завершения текущего извлечения немедленно запускается «догоняющее извлечение» с актуальным контекстом, чтобы последняя порция сообщений не была упущена.

**Правило взаимного исключения**: если основной Agent сам записал файлы памяти (`hasMemoryWritesSince` это обнаруживает), Fork Agent пропускает текущее извлечение и только продвигает курсор:

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // Основной Agent записал — пропускаем fork agent, продвигаем курсор
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

(`services/extractMemories/extractMemories.ts:198-209`)

### Анализ промпта для извлечения

Ключевая философия дизайна промпта для извлечения — **информационная эффективность**:

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // предварительно вставленный список существующих записей памяти
  ].join('\n')
}
```

(`services/extractMemories/prompts.ts:20-47`)

Предварительная вставка списка существующих записей (`existingMemories`) — ключевая оптимизация: Agent не тратит ход на перечисление директории, а сразу получает структурированный список файлов (с именем, типом, временной меткой, описанием) прямо в prompt.

### Механизм срабатывания Session Memory

Session Memory использует другой механизм срабатывания — через `postSamplingHooks`, а не Stop Hooks; оценка необходимости обновления происходит после каждой выборки модели:

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

(`services/SessionMemory/sessionMemory.ts:130-150`)

Пороговые значения по умолчанию (`DEFAULT_SESSION_MEMORY_CONFIG`, `services/SessionMemory/sessionMemoryUtils.ts:29-33`):

| Параметр | Значение по умолчанию | Описание |
|----------|-----------------------|----------|
| `minimumMessageTokensToInit` | 10 000 | Минимальное количество токенов для инициализации session memory |
| `minimumTokensBetweenUpdate` | 5 000 | Минимальный прирост токенов между обновлениями |
| `toolCallsBetweenUpdates` | 3 | Минимальное количество вызовов инструментов между обновлениями |

Эти значения можно динамически переопределять через удалённую конфигурацию GrowthBook (`tengu_sm_config`).

---

## 9.7 Умный отзыв памяти

### Sonnet выбирает не более 5 релевантных записей

Отзыв памяти — это не полная загрузка, а **сначала сканирование frontmatter, затем выбор Sonnet не более 5 наиболее релевантных записей**.

Основной процесс в `findRelevantMemories()` (`memdir/findRelevantMemories.ts:32-66`):

1. `scanMemoryFiles()` сканирует директорию памяти, читает первые 30 строк каждого файла (frontmatter), возвращает `MemoryHeader[]`
2. Отфильтровываются уже показанные в предыдущих ходах записи (`alreadySurfaced`), освобождая 5 слотов для нового контента
3. Sonnet вызывает `selectRelevantMemories()` на основе запроса и описаний файлов, выбирая наиболее релевантные имена файлов
4. Возвращаются пути и mtime выбранных записей

### Логика определения релевантности

Системный промпт Sonnet тщательно разработан:

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

(`memdir/findRelevantMemories.ts:13-23`)

**Ключевой дизайн**: справочная документация по недавно использованным инструментам не должна выбираться (не нужна во время использования), тогда как записи с **подводными камнями / известными проблемами** для тех же инструментов всё равно должны выбираться (предупреждения о ловушках нужны именно во время использования).

Вызов API использует структурированный вывод (JSON Schema) для гарантии разбираемости формата:

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

(`memdir/findRelevantMemories.ts:97-108`)

### Способ инъекции памяти в контекст

Выбранные записи вводятся в сообщения пользователя в тегах `<system-reminder>` (`wrapMessagesInSystemReminder`). К записям старше 1 дня добавляется предупреждение о свежести:

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

(`memdir/memoryAge.ts:38-47`)

Этот дизайн решает реальную проблему: пользователи сообщали о случаях «уверенных утверждений на основе устаревшей памяти» — упомянутые пути к файлам или имена функций уже были изменены, но цитаты в памяти делали утверждения более убедительными, а не более подозрительными.

**Механизм защиты от дрейфа**: в системный промпт вставляется `MEMORY_DRIFT_CAVEAT`, требующий от Agent верифицировать актуальное состояние перед ответом, основанным на записях памяти:

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Синхронизация Team Memory

### Механизм синхронизации через REST API

Team Memory синхронизируется через сервер с помощью `services/teamMemorySync/`, дизайн API полностью описан в начале `index.ts`:

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → только метаданные + хэши
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert entries
404  = данных ещё нет
```

(`services/teamMemorySync/index.ts:10-13`)

Синхронизация требует **OAuth-аутентификации** (`CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`) и использует репозиторий GitHub (`owner/repo`) в качестве области действия.

**Механизм наблюдателя**: `watcher.ts` использует `fs.watch({recursive: true})` для мониторинга изменений в директории team, запуская push с debounce 2 секунды (`DEBOUNCE_MS = 2000`). Намеренно выбран нативный `fs.watch`, а не chokidar:

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS использует FSEvents (O(1) файловых дескрипторов), Linux — inotify (O(количество поддиректорий)), оба лучше kqueue-подхода chokidar.

### Оптимистичная блокировка (If-Match)

При загрузке используется оптимистичное управление параллелизмом через HTTP-заголовок `If-Match` с ETag (checksum):

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

(`services/teamMemorySync/index.ts:uploadTeamMemory`)

При ответе сервера 412 Precondition Failed возникает конфликт (другой пользователь изменил общую память за это время). Система использует эндпоинт `GET ?view=hashes` (лёгкий, возвращает только SHA-256 хэш каждого ключа без тела содержимого) для обновления локального `serverChecksums`, затем пересчитывает дельту и повторяет:

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### Стратегия разрешения конфликтов

Стратегия разрешения конфликтов — **сервер побеждает (server wins per-key)**: при Pull содержимое сервера перезаписывает локальное. Дельта-push загружает только ключи, локальные хэши которых отличаются от серверных; сервер использует upsert-семантику (ключи, не представленные в PUT, сохраняются).

Ограничение размера загрузки (`MAX_PUT_BODY_BYTES = 200_000`) предотвращает отклонение слишком большого тела запроса API Gateway (наблюдалось, что шлюз возвращает HTML-формат 413 при ~256-512 КБ, в отличие от структурированного 413 уровня приложения). При превышении лимита автоматически выполняется разбивка на несколько последовательных PUT; upsert-семантика гарантирует безопасность:

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // Жадная упаковка: разбиение по байтам, каждый пакет не превышает MAX_PUT_BODY_BYTES
  ...
}
```

(`services/teamMemorySync/index.ts:batchDeltaByBytes`)

**Подавление перманентных сбоев**: некоторые ошибки (no_oauth, no_repo, 4xx кроме 409/429) не могут быть устранены повторными попытками. При обнаружении таких ошибок система устанавливает `pushSuppressedReason`, предотвращая попадание push, запущенных watcher, в бесконечный цикл повторов (наблюдалось: устройство без OAuth за 2,5 дня отправило 167 тыс. событий push).

---

## 9.9 Анализ проектных решений

### Почему файловая система, а не база данных

Дизайн на основе файловой системы и Markdown имеет несколько ключевых преимуществ:

1. **Agent может напрямую работать**: FileRead/FileWrite/FileEdit — нативные инструменты Claude, никакого дополнительного API-слоя не нужно. Agent записывает память с теми же инструментами, что и код — минимальная когнитивная нагрузка.

2. **Пользователь может проверить**: `~/.claude/projects/.../memory/` — обычная папка, пользователь может напрямую выполнить `ls`, `cat`, `vim`, полная прозрачность.

3. **Совместимость с Git**: Markdown-файлы нативно поддерживают diff, grep, git history, удобно для дельта-расчётов Team Memory.

4. **Отказ от лишних абстракций**: база данных требует миграции схем, стратегии резервного копирования, слоя запросов — для «нескольких сотен КБ Markdown-файлов» это чрезмерная инженерия.

### Почему ограничен размер MEMORY.md

Ограничение в 200 строк / 25 КБ подкреплено реальными измеренными данными (значения на p97/p100). Ключевые причины:

- MEMORY.md загружается в системный prompt **при каждом запросе**, размер напрямую влияет на потребление токенов
- Слишком большой индекс вытесняет действительно полезный контекст
- Принудительное ограничение побуждает пользователей и Agent поддерживать индекс в лаконичном состоянии — одна строка, только «наживка», но не содержимое

Это классический дизайн «ограничение как инструмент качества» — не потому что технически нельзя вместить больше, а чтобы через ограничение направлять правильное использование.

### Соображения безопасности памяти

Система имеет несколько уровней защиты:

**Защита от обхода пути**: `teamMemPaths.ts` реализует три уровня проверок — сначала строковая проверка `..`, URL-кодированного обхода, атак через нормализацию Unicode, затем разрешение символических ссылок через `realpath` с проверкой фактического пути в файловой системе:

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

(`memdir/teamMemPaths.ts:130-133`)

**Сканирование секретов**: при записи в Team Memory `scanForSecrets()` проверяет 30 высокодостоверных шаблонов учётных данных (из библиотеки правил gitleaks), включая форматы токенов AWS, GCP, GitHub, Anthropic, OpenAI и других основных платформ. Сканирование выполняется двойным образом — **перед загрузкой** и **перед записью**:

- `checkTeamMemSecrets()` в `teamMemSecretGuard.ts` перехватывает записи на этапе `validateInput` инструментов FileWriteTool/FileEditTool
- `readLocalTeamMemory()` повторно сканирует перед push, пропуская файлы с чувствительной информацией

**Контроль инструментов по принципу минимальных прав**: функция `canUseTool` извлекающего Agent разрешает только:
- FileRead/Grep/Glob (только чтение)
- Bash-команды только для чтения (ls/find/cat/stat/wc/head/tail)
- FileEdit/FileWrite только внутри директории memory

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

(`services/extractMemories/extractMemories.ts:171-176`)

**Исключение безопасности ProjectSettings**: настройка `autoMemoryDirectory` доверяет только источникам policy/local/user, явно исключая projectSettings (`.claude/settings.json`):

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 Переносимые паттерны

Следующие паттерны из системы памяти напрямую применимы в других проектах:

### Паттерн 1: Принцип невыводимости

**Что следует запоминать**: любая информация, которую можно получить, запросив текущее состояние (код, файлы, git), не заслуживает запоминания. Память должна хранить только «исторический контекст» — почему было принято то или иное решение, какие ловушки встречались, неявные предпочтения пользователя.

**Применение в Doramagic**: слои «UNSAID» и «WHY», извлекаемые Soul Extractor, по природе соответствуют этому принципу. Документы с правилами OpenClaw можно запросить — их не нужно запоминать; но «это правило OpenClaw когда-то привело к сбою публикации» — вот что стоит запомнить.

### Паттерн 2: Двухшаговая запись + лёгкий индекс

Двухшаговый паттерн «файл + индекс» гарантирует постоянную лаконичность индекса (принудительное ограничение: не более 150 символов на строку), тогда как файлы содержимого могут быть детализированы. Потребление токенов индексом фиксировано, загрузка содержимого — по требованию.

**Применение в Doramagic**: `MEMORY.md` системы памяти аналогичен «каталогу кирпичиков» Doramagic — лёгкий загружаемый индекс, указывающий на детализированные файлы, которые разворачиваются по требованию.

### Паттерн 3: Фоновое извлечение через Fork Agent

Не блокировать основной диалог, разделять prompt cache, максимизировать попадания в кэш — стандартный паттерн для фоновых пост-обработчиков. Ключевые детали реализации:
- `skipTranscript: true` — не записывать в лог основной сессии
- `maxTurns: N` — предотвращать попадание Agent в цикл верификации
- Механизм курсора (`lastMemoryMessageUuid`) — обрабатывать только инкрементальные данные
- Stash + trailing run — не терять последние сообщения при занятости Agent

### Паттерн 4: Осведомлённость о свежести

Запись памяти — не вечный факт, а наблюдение с временны́м сроком действия. Система обеспечивает это через:
1. Добавление подсказки о возрасте «N дней назад» при отзыве
2. Вставку инструкции защиты от дрейфа в системный промпт (сначала верифицировать, потом ссылаться)
3. Требование к Agent активно обновлять устаревшие записи, а не сохранять их

Это особенно актуально для сценариев «извлечения знаний» Doramagic — извлечённые WHY/UNSAID устаревают по мере развития проекта, нужен аналогичный механизм поддержания свежести.

### Паттерн 5: Сканирование секретов на входе

Перед любой «трансграничной» записью (в общее пространство, сетевой загрузкой) следует сканировать секреты. Библиотека правил gitleaks предоставляет высокодостоверный набор шаблонов, готовый к повторному использованию. Ключевой дизайн: сканирование выполняется на этапе `validateInput` инструмента записи (а не постфактум), гарантируя, что секреты не попадут ни в один путь персистентности.

---

## 9.11 Индекс исходного кода

| Файл | Строк | Основная ответственность |
|------|-------|--------------------------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Основная логика Session Memory: условие срабатывания, вызов Fork Agent, ручной API |
| `services/SessionMemory/prompts.ts` | 324 | Шаблон Session Memory, построение промпта обновления, анализ размеров секций |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Управление состоянием Session Memory: конфигурация, пороги, вспомогательные функции |
| `services/extractMemories/extractMemories.ts` | 615 | Извлечение Persistent Memory: вызов Fork Agent, состояние замыкания, контроль параллелизма |
| `services/extractMemories/prompts.ts` | 154 | Построение промпта извлечения: два варианта (auto-only и combined с Team Memory) |
| `memdir/memdir.ts` | 507 | Логика усечения MEMORY.md, построение промпта памяти, гарантированное создание директории |
| `memdir/paths.ts` | 278 | Разрешение пути Auto Memory, проверка включения/отключения, валидация безопасности пути |
| `memdir/memoryTypes.ts` | 271 | Определение четырёх типов памяти, формат frontmatter, принципы отзыва/защиты от дрейфа/невыводимости |
| `memdir/findRelevantMemories.ts` | 141 | Отзыв через Sonnet: сканирование frontmatter → 5 релевантных записей |
| `memdir/memoryScan.ts` | 94 | Примитивы сканирования директории: чтение frontmatter, форматирование списка |
| `memdir/memoryAge.ts` | 53 | Расчёт свежести: дни, человекочитаемый текст, предупреждение об устаревании |
| `memdir/teamMemPaths.ts` | 292 | Пути Team Memory, защита от обхода пути (три уровня проверок), разрешение символических ссылок |
| `memdir/teamMemPrompts.ts` | 100 | Построение объединённого промпта Team Memory + Auto Memory |
| `services/teamMemorySync/index.ts` | 1256 | Ядро синхронизации: логика fetch/push, оптимистичная блокировка, пакетное разбиение, повторы при конфликтах |
| `services/teamMemorySync/watcher.ts` | 387 | Мониторинг файлов: debounce push, подавление перманентных сбоев, жизненный цикл запуска/остановки |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 правил сканирования секретов (подмножество gitleaks), вспомогательные функции redact |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema: TeamMemoryData, типы результатов синхронизации, SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | Перехват секретов перед записью: интеграция с validateInput FileWriteTool/FileEditTool |

**Быстрая справка по ключевым константам**:

| Константа | Значение | Расположение |
|-----------|----------|--------------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25 000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH` (Session Memory на секцию) | 2 000 токенов | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12 000 токенов | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10 000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5 000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| Лимит отзыва | 5 записей | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| Максимум файлов памяти | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Строки чтения Frontmatter | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Таймаут Team Memory | 30 000 мс | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Задержка debounce push | 2 000 мс | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| Максимум размера одного файла | 250 000 байт | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| Максимум тела PUT-запроса | 200 000 байт | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
| Количество правил сканирования секретов | 30 | `secretScanner.ts:SECRET_RULES` |
