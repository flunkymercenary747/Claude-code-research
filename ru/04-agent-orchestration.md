# Глава 4: Оркестрация агентов и многоагентная архитектура

## 4.1 Обзор и позиционирование

Многоагентная система Claude Code является наиболее сложной подсистемой всей архитектуры продукта: она охватывает около 8 700 строк ядерного кода и пересекает 12 ключевых модулей. Система решает фундаментальную инженерную задачу: **как безопасно и эффективно оркестрировать параллельное выполнение нескольких LLM-агентов в однопоточном REPL-приложении**.

Система предоставляет три уровня кооперации:

| Режим | Способ запуска | Параллелизм | Механизм связи | Уровень изоляции |
|------|---------|--------|---------|---------|
| **Subagent (по умолчанию)** | Вызов AgentTool | Sync/async | Возвращаемое значение функции | AsyncGenerator внутри процесса |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | Полностью async | XML `<task-notification>` | Независимый AbortController |
| **Team Mode** | `spawnTeammate()` + TeamFile | Персистентный параллелизм | Файловый почтовый ящик + поллинг | Tmux Pane / InProcess / Remote |

Эти три режима не являются независимыми реализациями — все они совместно используют единый движок `runAgent()` (`runAgent.ts`) и реализуют различные поведенческие характеристики через комбинации параметров. Это одно из наиболее элегантных архитектурных решений всей системы.

**Статистика исходного кода:**

| Файл | Строк | Ответственность |
|------|------|------|
| `AgentTool.tsx` | 1 397 | Единая точка входа, маршрутизация, управление жизненным циклом |
| `runAgent.ts` | 973 | Движок выполнения агента, цикл query() |
| `loadAgentsDir.ts` | 755 | Разбор определений агентов (Markdown/JSON/Plugin) |
| `agentToolUtils.ts` | 686 | Фильтрация инструментов, права доступа, сериализация результатов |
| `UI.tsx` | 871 | Рендеринг прогресса и результатов агента |
| `coordinatorMode.ts` | 369 | Системный промпт и контекст Coordinator |
| `SendMessageTool.ts` | 917 | 5-маршрутная маршрутизация сообщений |
| `spawnMultiAgent.ts` | 1 093 | Создание Teammate (Tmux/InProcess) |
| `inProcessRunner.ts` | 1 552 | Полная реализация backend InProcess |
| `teammateMailbox.ts` | 1 183 | Протокол файлового почтового ящика |
| `worktree.ts` | 1 519 | Изоляция Git Worktree |

## 4.2 Теоретические основы

### 4.2.1 Модель акторов и оркестрация агентов

Многоагентная архитектура Claude Code является прагматичным вариантом модели акторов в области оркестрации LLM. Три ключевых примитива классической модели акторов (Hewitt, 1973) — **получение сообщений, создание новых акторов, отправка сообщений** — имеют чёткое соответствие в коде:

| Примитив актора | Реализация в Claude Code | Расположение в исходном коде |
|-----------|-----------------|---------|
| Получение сообщения | Цикл поллинга `waitForNextPromptOrShutdown()` | `inProcessRunner.ts:689-868` |
| Создание актора | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| Отправка сообщения | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

Однако имеются два ключевых отклонения от чистой модели акторов:

1. **Асимметричная иерархия**: Leader имеет глобальный вид (AppState), Worker — только свой собственный ToolUseContext. Это не равноправные акторы, а supervision tree с чёткой иерархией Leader-Worker.
2. **Канал с разделяемым состоянием**: Teammate с backend InProcess совместно использует корневое хранилище AppState через `setAppStateForTasks` (`runAgent.ts:336-337`), а не обменивается чистыми сообщениями. Это прагматичный компромисс — внутри одного процесса разделяемое состояние эффективнее, чем сериализация сообщений.

### 4.2.2 Модель параллелизма: передача сообщений vs. разделяемая память

Система одновременно использует оба подхода в зависимости от уровня изоляции:

**Модель передачи сообщений** (Team Mode — backend Tmux Pane):
```
Leader → writeToMailbox("worker-1", {...}) → файловая система → readMailbox() → Worker
```
Связь осуществляется через JSON-файлы + файловые блокировки. `LOCK_OPTIONS` в `teammateMailbox.ts` настраивает экспоненциальный backoff-retry (10 попыток, 5–100 мс) для сериализации параллельных записей:

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

**Модель разделяемой памяти** (backend InProcess):
```
Leader → setAppState(prev => {...}) → то же хранилище AppState ← getAppState() ← Worker
```
Teammate InProcess читает и пишет в корневое хранилище напрямую через `toolUseContext.setAppStateForTasks`. Гонки предотвращаются с помощью функционального обновления в стиле React `setAppState(prev => {...})` (хотя React не используется, применяется тот же CAS-паттерн).

### 4.2.3 Паттерн координатора в распределённых системах

Дизайн Coordinator Mode отображает классический паттерн координатора распределённых систем (также называемый Master-Worker), добавляя уникальное ограничение: **Coordinator сам является LLM-агентом, и его «логика координации» не захардкожена, а запрограммирована через системный промпт**.

Функция `getCoordinatorSystemPrompt()`, определённая в `coordinatorMode.ts:126-369`, возвращает структурированный промпт объёмом около 5 000 символов с полной стратегией планирования Worker:

```typescript
// coordinatorMode.ts:161-167 — ключевые правила планирования
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

Этот паттерн «программирования координирующей логики через промпт» означает, что поведение Coordinator регулируется изменением промпта — четырёхфазный рабочий процесс «исследование → синтез → реализация → верификация» не принудительно задан кодом, а реализован через следование инструкциям LLM. Это кардинально отличается от жёстко закодированной логики планирования в традиционных распределённых координаторах.

## 4.3 Архитектура и структуры данных

### 4.3.1 Общая архитектурная схема (Leader-Worker)

```
                    ┌─────────────────────────────────────────┐
                    │           Human User (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ user input
                    ┌──────────────▼──────────────────────────┐
                    │         Main REPL (query() loop)         │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — маршрут.  │     │
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
                    │      │ query() │     │  3 Backends:   │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    Уровень связи:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → parent collects      │
    │  Async Agent:   <task-notification> XML → user msg   │
    │  Teammate:      файловый ящик (.claude/teams/*/inboxes/)  │
    │  InProcess:     AppState shared + mailbox fallback   │
    │  Remote (ant):  teleportToRemote() → CCR session     │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 Система типов AgentDefinition

Определение агента использует трёхуровневый дизайн объединённого типа:

```typescript
// loadAgentsDir.ts — основная иерархия типов

// Базовый тип: поля, общие для всех агентов
type BaseAgentDefinition = {
  agentType: string              // ключ маршрутизации (например, "Explore", "worker")
  whenToUse: string              // основание для выбора агента LLM
  tools?: string[]               // белый список (undefined = все)
  disallowedTools?: string[]     // чёрный список
  model?: string                 // 'inherit' | конкретное имя модели
  effort?: EffortValue           // уровень усилий при рассуждении
  permissionMode?: PermissionMode // стратегия наследования прав
  maxTurns?: number              // максимальное число диалоговых ходов
  background?: boolean           // всегда выполнять в фоне
  isolation?: 'worktree' | 'remote' // режим изоляции
  memory?: AgentMemoryScope      // персистентная память
  omitClaudeMd?: boolean         // пропустить CLAUDE.md (экономия ~5-15 Gtok/неделю)
  // ...
}

// Built-in Agent: динамический промпт, без статического systemPrompt
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent: загружается из Markdown/JSON
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent: из системы плагинов
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// Итоговый объединённый тип
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

Изящество дизайна заключается в методе `getSystemPrompt`: Built-in Agent принимает параметр `toolUseContext` (промпт может динамически меняться в зависимости от текущего набора инструментов), тогда как Custom/Plugin Agent используют замыкание для захвата содержимого Markdown, разобранного во время загрузки. Это означает:

- **Промпт Built-in Agent динамический**: может отличаться при каждом вызове
- **Промпт Custom Agent статический**: определяется файлом Markdown, но при включённой `memory` содержимое памяти добавляется во время выполнения (`loadAgentsDir.ts:335-340`)

Приоритет загрузки определений агентов следует цепочке переопределений: `builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents` — реализован через `getActiveAgentsFromList()` с помощью Map, где более поздние записи перекрывают более ранние (`loadAgentsDir.ts:169-186`).

### 4.3.3 Унифицированная абстракция трёх backend'ов выполнения

Все три backend'а используют единый интерфейс AsyncGenerator `runAgent()`, но кардинально различаются по модели процессов и механизму связи:

| Аспект | Tmux Pane | InProcess | Remote (только ant) |
|------|-----------|-----------|-------------------|
| **Модель процесса** | Независимый процесс Claude CLI | Изоляция AsyncLocalStorage в том же процессе | Удалённая CCR-сессия |
| **Способ запуска** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **Связь** | Поллинг файлового ящика (500 мс) | Разделяемый AppState + файловый ящик (fallback) | HTTP API |
| **Права** | Независимый контекст прав | Проброс через UI-очередь Leader | Удалённый независимый |
| **Ресурсы** | Высокие (полный процесс) | Низкие (разделяемая V8-куча) | Очень высокие (удалённый экземпляр) |
| **Время жизни** | Независимо от Leader | Привязан к процессу Leader | Независимо |
| **Логика определения** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

Определение backend'а и откат при недоступности реализованы в `spawnMultiAgent.ts:339-375` в виде элегантной цепочки деградации:

```
iTerm2 (it2 backend) → Tmux → InProcess fallback
```

Если обнаружен iTerm2, но CLI it2 не установлен, система показывает интерактивный промпт настройки (`It2SetupPrompt`), позволяя пользователю выбрать установку it2 или переключиться на Tmux.

### 4.3.4 Структуры данных протокола связи

**Формат сообщения файлового ящика** (`teammateMailbox.ts:42-49`):

```typescript
type TeammateMessage = {
  from: string       // имя отправителя
  text: string       // содержимое сообщения (текст или структурированный JSON)
  timestamp: string  // временная метка ISO
  read: boolean      // признак прочтения
  color?: string     // цветовой идентификатор отправителя
  summary?: string   // краткое превью для UI (5–10 слов)
}
```

Путь почтового ящика имеет фиксированный формат: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**Типы структурированных сообщений** (передаются через JSON-кодирование поля `text`):

| Тип сообщения | Направление | Назначение |
|---------|------|------|
| `shutdown_request` | Leader → Worker | Запрос завершения |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | Ответ на завершение |
| `idle_notification` | Worker → Leader | Уведомление о простое |
| `permission_request` | Worker → Leader | Запрос прав |
| `permission_response` | Leader → Worker | Ответ на запрос прав |
| `plan_approval_request` | Worker → Leader | Запрос утверждения плана (Plan Mode) |
| `plan_approval_response` | Leader → Worker | Ответ на утверждение плана |
| `sandbox_permission_request` / `_response` | Двунаправленный | Права сетевой песочницы |
| `task_assignment` | Leader → Worker | Назначение задачи |
| `team_permission_update` | Leader → Workers | Широковещательное обновление прав |

## 4.4 Ключевые алгоритмы и потоки выполнения

### 4.4.1 Дерево решений маршрутизации AgentTool (полное)

`AgentTool.call()` является единой точкой входа системы; логика маршрутизации реализована в `AgentTool.tsx:238-764`. Полное дерево решений:

```
AgentTool.call(input) — вход
│
├─ [1] Оба параметра team name и name присутствуют?
│   ├─ ДА: попытка Teammate породить вложенного?
│   │   ├─ ДА: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ НЕТ: → spawnTeammate() (return teammate_spawned)
│   └─ НЕТ: продолжить
│
├─ [2] Разобрать effectiveType (subagent_type)
│   ├─ указан явно → использовать указанное значение
│   ├─ не указан + Fork Gate ВКЛ → undefined (путь Fork)
│   └─ не указан + Fork Gate ВЫКЛ → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (путь Fork)
│   ├─ ДА: рекурсивная проверка Fork
│   │   ├─ уже в дочернем Fork-процессе → throw
│   │   └─ прошёл → selectedAgent = FORK_AGENT
│   └─ НЕТ: искать в activeAgents
│       ├─ найден → selectedAgent = found
│       ├─ запрещён правилом → throw (с информацией о правиле)
│       └─ не существует → throw (перечислить доступных агентов)
│
├─ [4] Разобрать effectiveIsolation
│   ├─ 'remote' (только ant) → teleportToRemote() → return remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → использовать worktreePath на следующих шагах
│
├─ [5] Сформировать system prompt и prompt messages
│   ├─ путь Fork: наследовать промпт родителя + buildForkedMessages()
│   └─ обычный: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] Определение shouldRunAsync
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — НО НЕ isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → return { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → вход в цикл while(true)
│       ├─ Гонка: nextMessage vs backgroundSignal
│       │   ├─ побеждает background → переключиться на async (wasBackgrounded=true)
│       │   └─ побеждает message → yield message, отслеживать прогресс
│       └─ цикл завершён → finalizeAgentTool() → return AgentToolResult
```

### 4.4.2 Процесс выполнения AsyncGenerator runAgent()

`runAgent()` — основной движок всей многоагентной системы (`runAgent.ts:247-860`). Это `AsyncGenerator<Message, void>`: при каждом yield одного сообщения вызывающая сторона может обработать его (записать, отобразить или поместить в очередь фонового режима).

**Ключевые фазы выполнения:**

1. **Разрешение инструментов**: `resolveAgentTools()` разрешает белый список `tools` из определения агента в реальные объекты Tool, одновременно применяя чёрный список `disallowedTools` (`runAgent.ts:500-502`)

2. **Формирование System Prompt**: строится на основе `override?.systemPrompt` или `getAgentSystemPrompt()`; Explore/Plan Agent пропускают `claudeMd` и `gitStatus`, экономя ~5–15 Gtok/неделю по всему флоту (`runAgent.ts:389-409`)

3. **Стратегия AbortController** (`runAgent.ts:524-528`):
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // внешнее управление (async-путь)
     : isAsync
       ? new AbortController()      // async: независимый controller
       : toolUseContext.abortController  // sync: общий с родителем
   ```

4. **Переопределение прав** (`runAgent.ts:414-497`): `permissionMode` агента перекрывает режим родителя, но `bypassPermissions`, `acceptEdits` и `auto` родителя всегда имеют приоритет — это гарантирует, что политика безопасности, заданная администратором, не может быть понижена дочерним агентом.

5. **Основной цикл** — прямой вызов `query()` с yield (`runAgent.ts:748-806`):
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
     // ... обработка stream_event, attachment, recordable messages
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **Блок очистки finally** (`runAgent.ts:816-858`): очистка MCP, session hooks, отслеживание кэша промптов, освобождение кэша состояния файлов, завершение фоновых bash-задач, дерегистрация Perfetto, очистка todos в AppState — 9 операций очистки, гарантирующих отсутствие утечек ресурсов.

### 4.4.3 Жизненный цикл асинхронного агента (fire-and-forget)

Полный жизненный цикл асинхронного агента управляется `runAsyncAgentLifecycle()` (`agentToolUtils.ts:322-497`):

```
registerAsyncAgent() → регистрация задачи в AppState
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — сбор всех сообщений
   │   ├─ agentMessages.push(message)
   │   ├─ если task.retain → добавить в AppState.tasks[taskId].messages
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — событие прогресса SDK
   │
   ├─ finalizeAgentTool() — извлечение итогового результата
   │
   ├─ completeAsyncAgent() — отметить завершение (ПЕРВЫМ, до медленных операций)
   │   │                      ↑ ключевое решение: исправление gh-20236
   │   │                        classifyHandoff и worktree cleanup могут зависнуть
   │   │                        и не должны блокировать переход состояния
   │
   ├─ classifyHandoffIfNeeded() — проверка классификатора безопасности (необязательно)
   │
   ├─ getWorktreeResult() — очистка worktree
   │
   └─ enqueueAgentNotification() — уведомление родителя через XML <task-notification>
```

**Исправление gh-20236** — это архитектурное решение, заслуживающее документирования: `completeAsyncAgent()` вызывается до `classifyHandoffIfNeeded()` и `getWorktreeResult()`. Комментарий явно объясняет причину:

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 Фильтрация инструментов и наследование прав

Фильтрация инструментов представляет собой трёхуровневую цепочку фильтров (`agentToolUtils.ts:66-115`):

```
Уровень 1: ALL_AGENT_DISALLOWED_TOOLS — инструменты, запрещённые для всех агентов
Уровень 2: CUSTOM_AGENT_DISALLOWED_TOOLS — дополнительно запрещённые только для Custom Agent
Уровень 3: ASYNC_AGENT_ALLOWED_TOOLS — белый список для асинхронных агентов (обратная логика)
```

Особые исключения:
- Инструменты MCP (префикс `mcp__`) всегда разрешены
- `ExitPlanMode` всегда разрешён в Plan Mode
- InProcess Teammate в режиме Agent Swarms может использовать `AgentTool` (для создания синхронных дочерних агентов) и инструменты Task (для координации через общий список задач)

Разрешение инструментов также поддерживает подстановочный знак (`'*'` или `undefined` = все инструменты) и ограничения на уровне агента (синтаксис `AgentTool(worker, researcher)`, `agentToolUtils.ts:165-172`).

### 4.4.5 Четырёхфазный рабочий процесс Coordinator Mode

Основная логика Coordinator Mode определена в `coordinatorMode.ts:126-369` через `getCoordinatorSystemPrompt()`. Все задачи разбиваются на четыре фазы:

**Фаза 1: Research** (параллельное выполнение Workers)
- Несколько Workers одновременно исследуют кодовую базу
- Ключевая инструкция: *"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Фаза 2: Synthesis** (выполняет сам Coordinator)
- Это самая критичная фаза — Coordinator обязан лично прочитать результаты исследования и понять их
- Явно запрещённый антипаттерн: *"Never write 'based on your findings'"*
- Требуется synthesized spec с конкретными путями к файлам, номерами строк и содержанием изменений

**Фаза 3: Implementation** (выполняют Workers)
- Coordinator решает: continue (`SendMessageTool`) или spawn fresh (`AgentTool`)
- Критерий решения — степень перекрытия контекста (в промпте приведена полная таблица решений)

**Фаза 4: Verification** (независимый Worker)
- Явное требование независимой верификации: *"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- Стандарт верификации: *"proving the code works, not confirming it exists"*

### 4.4.6 Персистентное сотрудничество в Team Mode

Team Mode реализует персистентное состояние команды на основе TeamFile (`.claude/teams/{team_name}/team.json`). В отличие от Workers в Coordinator Mode с fire-and-forget, Teammate — это **долгоживущий процесс**:

1. **Создание**: `spawnTeammate()` создаёт Tmux pane или InProcess task
2. **Работа**: Teammate выполняет промпт → завершает → отправляет `idle_notification` → ожидает следующего промпта
3. **Связь**: все сообщения проходят через файловый почтовый ящик (любой backend может использовать файловую систему)
4. **Завершение**: Leader отправляет `shutdown_request` → LLM Teammate решает approve или reject

Основной цикл InProcess Runner (`inProcessRunner.ts:883-1464`) реализует полную семантику персистентности:

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. Выполнить текущий промпт (вызов runAgent())
  // 2. Отметить как idle
  // 3. Отправить idle_notification Leader
  // 4. waitForNextPromptOrShutdown() — поллинг почтового ящика
  //    ├─ shutdown_request → передать LLM для принятия решения
  //    ├─ new_message → задать как промпт следующего раунда
  //    └─ aborted → shouldExit = true
}
```

Стратегия приоритетности сообщений (`inProcessRunner.ts:760-804`) заслуживает внимания:
1. Наивысший приоритет: `shutdown_request` (команда завершения от Leader не может потеряться)
2. Далее: сообщения от `team-lead` (Leader представляет намерение пользователя)
3. Последними: peer-сообщения из FIFO-очереди

### 4.4.7 Протокол связи через файловый почтовый ящик

Файловый почтовый ящик является коммуникационной основой для всех backend'ов. При его проектировании была выбрана **простота** в ущерб производительности:

**Протокол записи** (`teammateMailbox.ts:133-191`):
```
1. ensureInboxDir() — убедиться, что директория существует
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — атомарное создание (если не существует)
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — получить файловую блокировку
4. readMailbox() — повторное чтение под блокировкой (избежать грязного чтения)
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — записать обратно
7. release() — освободить блокировку
```

**Протокол чтения** (`teammateMailbox.ts:83-107`):
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. вернуть TeammateMessage[]
```

Обратите внимание: чтение **без блокировки** — это намеренное решение. Читатель требует только eventual consistency, а атомарность записи гарантируется `lockfile`.

### 4.4.8 5-маршрутная маршрутизация SendMessage

`SendMessageTool.call()` реализует 5 независимых маршрутов (`SendMessageTool.ts`):

```
значение input.to
│
├─ [маршрут 1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — Remote Control между машинами
│   (требует проверки безопасности: межмашинные сообщения требуют явного согласия пользователя)
│
├─ [маршрут 2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — локальный Unix Domain Socket
│
├─ [маршрут 3] совпадение с agentNameRegistry или toAgentId
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ задача остановлена/вытолкнута → resumeAgentBackground()
│       (автоматическое восстановление остановленного агента из дискового транскрипта)
│
├─ [маршрут 4] to === '*'
│   → handleBroadcast() — итерировать по TeamFile.members, писать в каждый ящик
│
└─ [маршрут 5] остальное
    ├─ чистый текст → handleMessage() — запись в почтовый ящик
    └─ структурированное сообщение → диспетчеризация в соответствующий обработчик:
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

Механизм **автоматического восстановления** в маршруте 3 особенно изящен: при отправке сообщения остановленному агенту система автоматически восстанавливает его из дискового транскрипта и запускает в фоне. Это позволяет Coordinator бесшовно продолжать работу ранее завершённого Worker через `SendMessage`, не заботясь о том, запущен ли он.

### 4.4.9 Полный поток делегирования прав

Обработка прав для InProcess Teammate — одна из наиболее сложных частей всей системы (`inProcessRunner.ts:127-449`). Главная проблема: **как фоновый агент может запросить авторизацию у человека?**

Решение использует двухуровневый fallback:

**Основной путь: проброс через UI-очередь Leader**
```
Worker инициирует инструмент, требующий прав
  → вызывается createInProcessCanUseTool()
  → hasPermissionsToUseTool() возвращает { behavior: 'ask' }
  → проверка автоматического одобрения Bash classifier (если доступен)
  → getLeaderToolUseConfirmQueue() — получить очередь подтверждений UI Leader
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ идентификатор Worker
     └→ терминал Leader показывает диалог подтверждения с бейджем Worker
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Fallback-путь: запрос прав через почтовый ящик**
```
Worker инициирует инструмент, требующий прав
  → UI-очередь Leader недоступна
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → поллинг собственного почтового ящика (интервал 500 мс)
  → ожидание ответа permission_response от Leader
  → processMailboxPermissionResponse()
```

Важна и схема распространения обновлений прав: когда Leader одобряет право с опцией «Always allow», `persistPermissionUpdates()` записывает на диск, а `getLeaderSetToolPermissionContext()` записывает обновление обратно в разделяемый контекст Leader — но с `preserveMode: true`, чтобы режим `acceptEdits` Worker не просочился обратно к Coordinator (`inProcessRunner.ts:275-277`).

### 4.4.10 Полный жизненный цикл Worker

```
Рождение
  │
  ├─ путь Sync Agent:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → return AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ путь Async Agent:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ путь InProcess Teammate:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — персистентный цикл:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → новое сообщение/shutdown/abort → решение о продолжении
          }

Выполнение
  │
  ├─ цикл query() → вызов API → tool_use → проверка canUseTool
  │   ├─ allow → выполнить инструмент
  │   ├─ deny → инструмент отклонён
  │   └─ ask → диалог прав (sync) или права через почтовый ящик (async/teammate)
  │
  ├─ отслеживание прогресса:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (событие SDK)
  │
  └─ автоматический уход в фон (только Sync Agent):
      гонка с backgroundPromise → если пользователь нажал Ctrl+Z
      → wasBackgrounded = true → продолжить выполнение в фоне

Связь
  │
  ├─ Sync Agent: yield message → родитель напрямую собирает
  ├─ Async Agent: <task-notification> вводится в user messages родителя
  └─ Teammate: writeToMailbox() → Leader читает при поллинге

Завершение
  │
  ├─ нормальное завершение: finalizeAgentTool() → извлечь итоговый текст → отметить completed
  ├─ Kill пользователем: AbortError → killAsyncAgent() → извлечь partialResult → уведомить
  ├─ ошибка: catch → failAsyncAgent() → уведомить об ошибке
  └─ очистка: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Создание и очистка изоляции Worktree

Git Worktree обеспечивает изоляцию на уровне файловой системы для агентов (`worktree.ts`). Основной процесс:

**Создание** (`worktree.ts:234-374`):
```
1. validateWorktreeSlug(slug) — предотвратить атаку path traversal
2. Проверка быстрого восстановления: readWorktreeHeadSha() — если worktree уже существует, пропустить fetch
3. Если не существует:
   a. Попытка прочитать локальный ref origin/<default> (избежать задержки 6–8 сек от `git fetch`)
   b. Если локально нет → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. Необязательно: sparse-checkout (проверить только указанные пути)
4. performPostCreationSetup():
   - скопировать settings.local.json
   - настроить git hooks (решить проблему core.hooksPath для husky)
   - создать символические ссылки на node_modules и другие большие директории
   - скопировать файлы, указанные в .worktreeinclude (gitignored)
```

**Решение об очистке** (`AgentTool.tsx:644-685`):
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // hook-based всегда сохраняется
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // нет изменений — удалить worktree
    }
  }
  return { worktreePath, worktreeBranch }; // есть изменения — сохранить
};
```

Ключевые меры безопасности:
- `validateWorktreeSlug()` проверяет, что каждый сегмент, разделённый `/`, соответствует `[a-zA-Z0-9._-]+`, предотвращая обход пути `../../../`
- `flattenSlug()` выравнивает вложенные slug (`user/feature` → `user+feature`), избегая конфликтов D/F для git-ref и проблем с вложенностью директорий
- `GIT_NO_PROMPT_ENV` отключает все запросы git-учётных данных, предотвращая зависание CLI

## 4.5 Анализ архитектурных решений

### 4.5.1 Почему выбран файловый почтовый ящик, а не IPC

Файловый почтовый ящик выглядит «примитивным» выбором — почему не Unix Domain Socket, Named Pipe или gRPC?

**Главная причина: независимость от backend'а**. Файловая система — наибольший общий делитель для всех трёх backend'ов (Tmux, InProcess, Remote):
- Tmux Pane — независимый процесс, без разделяемой памяти
- InProcess — в том же процессе, но с изоляцией через AsyncLocalStorage
- Remote — через сеть, но может использовать общую сетевую файловую систему

Дополнительные преимущества файлового почтового ящика:
1. **Наблюдаемость**: отладка простым `cat ~/.claude/teams/*/inboxes/*.json`
2. **Персистентность**: сообщения не теряются при сбоях процесса
3. **Простота**: нет сложного управления соединениями, heartbeat, переподключений
4. **Потокобезопасность**: файловых блокировок `proper-lockfile` достаточно

Ценой является **задержка**: интервал поллинга 500 мс означает максимальную задержку доставки 500 мс. Но для LLM-агентов каждый вызов инструмента занимает несколько секунд, и 500 мс незначительны.

### 4.5.2 Компромиссы InProcess vs. Pane backend

| Аспект | InProcess | Tmux Pane |
|------|-----------|-----------|
| **Память** | Разделяемая V8-куча (мало) | Независимая куча процесса (много) |
| **Задержка запуска** | ~0 мс | ~2–3 с (запуск CLI) |
| **Изоляция** | AsyncLocalStorage (слабая) | Процесс ОС (сильная) |
| **Права** | Проброс через UI Leader (реальное время) | Поллинг почтового ящика (с задержкой) |
| **Отладка** | Общий лог (сложнее) | Отдельный терминал (нагляднее) |
| **Время жизни** | Привязан к Leader | Независимо |

Главное преимущество backend InProcess — **проброс прав**: через `getLeaderToolUseConfirmQueue()` диалог прав Worker отображается прямо в терминале Leader с бейджем Worker. Пользователю не нужно переключаться на терминал Worker для подтверждения прав.

Однако InProcess имеет фундаментальное ограничение: **Worker не может создавать фоновые агенты** (`AgentTool.tsx:277-278`), поскольку его жизненный цикл привязан к процессу Leader, а фоновые агенты требуют независимого AbortController.

### 4.5.3 Философия дизайна: права всегда под контролем человека

Вся система управления правами в многоагентной архитектуре следует непреложному принципу: **человек всегда является конечным гарантом прав**.

Реализация этого принципа в коде:
1. **Дочерний агент не может повысить права**: `runAgent.ts:419` — настройки родителя `bypassPermissions`, `acceptEdits`, `auto` всегда имеют приоритет над `permissionMode` дочернего агента
2. **Права Leader не просачиваются к Worker**: `runAgent.ts:467-477` — при задании `allowedTools` правила allow на уровне сессии очищаются, сохраняются только правила уровня CLI-аргументов
3. **Межмашинные сообщения требуют явного согласия**: `SendMessageTool.ts:checkPermissions` — отправка на адрес `bridge:` требует `safetyCheck`, при этом `classifierApprovable: false` (классификатор безопасности не может автоматически одобрить)
4. **Утверждение Plan Mode**: Teammate может быть настроен с `plan_mode_required`, в этом случае план должен быть сначала передан Leader на утверждение

### 4.5.4 Рекурсивный дизайн с повторным использованием цикла query()

Ядро `runAgent()` — это вызов функции `query()`, той самой, которую использует главный цикл REPL. Это означает, что **дочерний агент и главный агент используют абсолютно одинаковые конвейеры вызовов API и выполнения инструментов**.

```typescript
// runAgent.ts:748-757 — вызов query() агентом
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

Глубокие следствия этого дизайна:
- **Согласованность инструментов**: инструменты агента идентичны инструментам пользователя (только отфильтрованы)
- **Рекурсивная способность**: если `AgentTool` входит в набор инструментов агента, агент может создавать дочерних агентов (InProcess Teammate разрешено создавать синхронные дочерние агенты)
- **Повторное использование Prompt Cache**: путь Fork через `useExactTools` гарантирует побайтовое совпадение префиксов API-запросов дочернего и родительского агентов, максимизируя частоту попаданий в кэш

Рекурсия также несёт риск — бесконечного рекурсивного fork. Решение — двойная проверка (`AgentTool.tsx:331-333`):
1. `querySource === 'agent:builtin:fork'` — устойчив к компиляции (context.options не затрагивается autocompact)
2. `isInForkChild(messages)` — откат через сканирование сообщений

### 4.5.5 Сравнение с LangGraph / AutoGen / CrewAI

| Аспект | Claude Code | LangGraph | AutoGen | CrewAI |
|------|------------|-----------|---------|--------|
| **Модель оркестрации** | Leader-Worker (prompt-programmed) | DAG/StateGraph | Agent Chat | Sequential/Hierarchical |
| **Связь** | Файловый ящик + разделяемый AppState | State channels | Python function calls | Shared memory |
| **Изоляция** | 3 уровня (InProcess/Pane/Remote) | Нет | Нет | Нет |
| **Права** | Human-in-the-loop, всегда | Необязательно | Необязательно | Нет |
| **Персистентность** | Дисковый транскрипт + TeamFile | Необязательный checkpointing | Нет | Нет |
| **Совместное использование инструментов** | Единый пул инструментов + фильтрация | Независимая привязка к узлу | Независимо для каждого агента | Независимо для каждого агента |
| **Гетерогенность моделей** | Параметр `model` задаётся для каждого агента | Поддерживается | Поддерживается | Поддерживается |

Два главных отличия Claude Code:

1. **Логика Coordinator запрограммирована через промпт** — у других фреймворков логика оркестрации жёстко задана в виде DAG или машины состояний. Coordinator Claude Code программируется на естественном языке, что означает возможность изменять стратегию оркестрации правкой промпта без изменения кода.
2. **Файловая система как коммуникационная основа** — выглядит примитивно, но обеспечивает единую коммуникацию между процессами и машинами с полной наблюдаемостью. Другие фреймворки полагаются на Python function call внутри процесса и требуют дополнительного RPC-слоя для многомашинных сценариев.

## 4.6 Переносимые паттерны

### 4.6.1 Универсальные паттерны оркестрации агентов

Из реализации Claude Code можно извлечь 5 универсальных паттернов оркестрации агентов:

**Паттерн 1: AsyncGenerator как интерфейс агента**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGenerator обеспечивает семантику pull-based потока сообщений — вызывающая сторона решает, когда потреблять следующее сообщение, что естественно поддерживает переключение в фон (вставка гонки в точке yield) и отслеживание прогресса.

**Паттерн 2: Бесшовное переключение Foreground → Background**

Sync Agent в Claude Code может уйти в фон в процессе выполнения — через `Promise.race([nextMessage, backgroundSignal])`. Этот паттерн применим к любому сценарию, где «долгая задача может быть переведена в фон». Ключ — стабильный `taskId`, передаваемый между foreground и background.

**Паттерн 3: Файловая система как «наименьший общий знаменатель» коммуникации между агентами**

При необходимости унифицированной коммуникации между несколькими backend'ами (в-процессе/между-процессами/между-машинами) файловая система — самый простой выбор. JSON-файлы + файловые блокировки обеспечивают достаточные гарантии согласованности.

**Паттерн 4: Coordination через промпт**

Запись логики оркестрации в system prompt, а не в код, превращает стратегию координации в «конфигурацию», а не «реализацию». Это особенно ценно на этапе быстрой итерации над оркестрацией агентов — стоимость изменения промпта несравнимо ниже изменения кода.

**Паттерн 5: Безопасный переход состояний с приоритетом над декоративными уведомлениями**

Паттерн исправления gh-20236: в асинхронном потоке сначала выполнять ключевой переход состояния (`completeAsyncAgent`), затем — возможно зависающие декоративные операции (classifier check, worktree cleanup). Любая потенциально блокирующая операция не должна блокировать критические переходы состояния.

### 4.6.2 Полезное для Doramagic FlowController

Архитектура агентов Claude Code и FlowController Doramagic (система lease + изоляция staging/delivery + конечный автомат из 12 состояний) имеют несколько точек сопоставления:

1. **Конечный автомат vs. Prompt-Programmed**: Doramagic использует конечный автомат из 12 состояний для жёсткого контроля процесса, Claude Code — промпт для программирования Coordinator. Оба подхода имеют свои области применения: детерминированные процессы — конечный автомат, процессы, требующие гибкого суждения — программирование через промпт.

2. **Прямая переносимость файлового почтового ящика**: структура директорий staging/delivery в Doramagic аналогична структуре `.claude/teams/*/inboxes/` в Claude Code. FlowController Doramagic может напрямую применить паттерн файлового почтового ящика для реализации слабосвязанной коммуникации между skill'ами.

3. **Заимствование модели прав**: принцип «дочерний агент не может повысить права» из Claude Code применим к правам skill в Doramagic — вызываемый skill не должен получать более высокий доступ к системе, чем вызывающий.

4. **Идея изоляции через Worktree**: для параллельного выполнения skill'ов в Doramagic (например, параллельное извлечение нескольких проектов несколькими soul extractor'ами) можно позаимствовать паттерн изоляции файловой системы через Worktree, создавая независимые рабочие директории для каждого параллельного выполнения.

## 4.7 Индекс исходного кода

| Файл | Путь | Ключевые экспорты |
|------|------|---------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool` (определение buildTool), `inputSchema`, `outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | AsyncGenerator `runAgent()`, `filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | объединённый тип `AgentDefinition`, `getAgentDefinitionsWithOverrides()`, `parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`, `resolveAgentTools()`, `finalizeAgentTool()`, `runAsyncAgentLifecycle()`, `classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`, `getCoordinatorSystemPrompt()`, `getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool` (5-маршрутная маршрутизация), `handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`, `handleSpawnSplitPane()`, `resolveTeammateModel()`, `buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`, `createInProcessCanUseTool()`, `waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`, `writeToMailbox()`, `markMessageAsReadByIndex()`, все типы структурированных сообщений |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`, `createAgentWorktree()`, `removeAgentWorktree()`, `validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | объединённый тип `TaskState` (7 видов задач), `isBackgroundTask()` |

**Объединённый тип TaskState** (`tasks/types.ts`):
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

*Глава основана на анализе снапшота TypeScript-исходников Claude Code (2026-03-31, ~512K LOC). Все ссылки на код помечены конкретными именами файлов и диапазонами строк.*
