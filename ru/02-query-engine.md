# Глава 2: Query Engine — ядро взаимодействия с LLM

## 2.1 Обзор и позиционирование

**В одной строке:** Query Engine — «сердцебиение» Claude Code: он владеет полным жизненным циклом диалога с LLM, преобразуя пользовательский ввод в многораундовое agentic-взаимодействие с контролем прав, выполнением инструментов, сжатием контекста и отслеживанием стоимости.

**Ключевая проблема, которую он решает:** как в одном async generator pipeline надёжно оркестрировать цикл «потоковый ответ LLM → вызов инструмента → инъекция результата → повторный запрос», одновременно обрабатывая переполнение окна контекста, бюджет токенов, сбои API, прерывание пользователем, подтверждение прав и минимум 12 других ветвей исключений.

**Задействованные файлы и объём кода:**

| Файл | Строк | Ответственность |
|------|-------|-----------------|
| `QueryEngine.ts` | 1 295 | Управление состоянием на уровне сессии, точка входа SDK/headless |
| `query.ts` | 1 729 | Основной цикл запроса, оркестрация выполнения инструментов, многоуровневое восстановление |
| `query/config.ts` | 46 | Неизменяемый снимок конфигурации |
| `query/tokenBudget.ts` | 93 | Принятие решений по auto-continue для бюджета токенов |
| `query/stopHooks.ts` | 473 | Хуки при остановке (шлюз безопасности, постобработка) |
| `query/deps.ts` | 40 | Интерфейс внедрения зависимостей (удобен для тестирования) |
| `services/api/claude.ts` | 3 419 | Вызовы API, потоковый парсинг, повторы, стратегии кэширования |
| `cost-tracker.ts` | 323 | Отслеживание стоимости и накопление данных использования |
| **Итого** | **~7 418** | — |

Эти 7 400+ строк составляют ~1,4% кода Claude Code, но это самый критически важный путь всего продукта — каждое взаимодействие пользователя с Claude проходит здесь.

---

## 2.2 Теоретические основы

### 2.2.1 Async Generator Pipeline (конвейер сопрограмм)

Ключевая архитектура Query Engine основана на **ES2018 Async Generator** как примитиве потоковой обработки. Сигнатура функции `query()` раскрывает это решение:

```typescript
// query.ts:162
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

Это не случайный синтаксический выбор, а точное применение теории **Coroutine**. Async generator одновременно обладает:
- **Ленивым вычислением**: потребитель управляет темпом, не вычисляя все API-ответы заранее
- **Двунаправленной коммуникацией**: yield возвращает stream event, return возвращает причину завершения
- **Безопасностью ресурсов**: блок finally гарантирует освобождение потока (`releaseStreamResources()` в `claude.ts`)

Традиционные callback или цепочки Promise не могут одновременно удовлетворить требованиям «потоковый вывод для UI» и «ожидание результата выполнения инструмента». Async generator нативно поддерживает семантику «производитель ждёт, пока потребитель не обработает».

### 2.2.2 State Machine (неявный конечный автомат)

Основной цикл `query.ts` — не рекурсивный вызов (хотя ранние комментарии ещё называют его `query_recursive_call`), а явный **конечный автомат на основе `while(true)` + `continue`**:

```typescript
// query.ts:218 (определение типа State)
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

При каждом `continue` State перестраивается заново (неизменяемое обновление), поле `transition` фиксирует причину перехода. Это классическая **машина Мили (Mealy Machine)** — вывод зависит от комбинации текущего состояния и входного события.

### 2.2.3 Backpressure и управление ресурсами

Backpressure критически важен при LLM streaming. Решение Claude Code:

1. **for-await-of как естественное ограничение скорости**: в `claude.ts` поток потребляется через `for await (const part of stream)`, скорость обработки потребителя напрямую определяет скорость отправки производителем
2. **Stream idle watchdog** (`claude.ts:2397`): если данные не поступают 90 секунд, поток прерывается принудительно и происходит fallback на non-streaming
3. **Гарантия жизненного цикла генератора**: блок `finally` гарантирует выполнение `releaseStreamResources()` при всех путях выхода (включая `.return()` и исключения)

### 2.2.4 Почему эта теория особенно важна в контексте LLM

Традиционный HTTP API-вызов — модель «запрос-ответ» с простой обработкой ошибок. Agentic loop LLM сталкивается с уникальными вызовами:

- **Один вызов может длиться 10 минут** (лимит non-streaming)
- **Ответ может инициировать новый I/O в середине** (вызов инструмента)
- **Окно контекста — конечный ресурс с состоянием**, требующий баланса между «потерей при сжатии» и «крашем при переполнении»
- **Стоимость накапливается в реальном времени** с возможностью прерывания в любой момент

Эти ограничения делают Event Loop + конечный автомат + Backpressure неизбежными теоретическими опорами.

---

## 2.3 Архитектура и структуры данных

### 2.3.1 Ключевой интерфейс класса QueryEngine

```typescript
// QueryEngine.ts:155-166
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
```

**Ключевая особенность проектирования:** QueryEngine создаётся на один диалог. Каждый вызов `submitMessage()` — новый ход в том же диалоге; состояние сохраняется между ходами.

### 2.3.2 Ключевые определения типов

**QueryEngineConfig** (`QueryEngine.ts:95-153`) — неизменяемая конфигурация, передаваемая при конструировании:

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**QueryConfig** (`query/config.ts:18-31`) — неизменяемый снимок среды выполнения на один запрос:

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

Обратите внимание на намерение проектирования, выраженное в комментарии к исходному коду (`config.ts:9-12`): «Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination.» Это чёткая граница между оптимизацией компиляции bun:bundle и конфигурацией во время выполнения.

### 2.3.3 Граф зависимостей между модулями

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- управление состоянием сессии
                    |  (1 295 строк)  |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- основной цикл & конечный автомат
                    |  (1 729 строк)  |  <-- State, цикл while(true)
                    +--------+--------+
                             |
              +--------------+------------------+
              |              |                  |
              v              v                  v
    +---------+---+  +-------+--------+  +------+-------+
    | query/      |  | query/         |  | query/       |
    | config.ts   |  | tokenBudget.ts |  | stopHooks.ts |
    | (46 строк)  |  | (93 строки)    |  | (473 строки) |
    +-------------+  +----------------+  +--------------+
              |              |
              v              v
    +---------+--------------+--------+
    |         query/deps.ts           |
    |   QueryDeps (DI interface)      |
    +---------+-----------------------+
              |
              |  callModel / autocompact / microcompact
              v
    +---------+-----------+     +----------+--------+
    | services/api/       |     | cost-tracker.ts   |
    | claude.ts           |     | (323 строки)      |
    | (3 419 строк)       |     | addToTotalSession |
    | queryModelWith      |     | Cost, formatCost  |
    | Streaming           |     +-------------------+
    +---------+-----------+
              |
              v
    +---------+-----------+
    | withRetry / stream  |
    | SSE parsing         |
    | Non-streaming       |
    | fallback            |
    +---------------------+
```

**Дизайн внедрения зависимостей в deps.ts** (`deps.ts:18-37`) заслуживает особого внимания:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

«Using `typeof fn` keeps signatures in sync with the real implementations automatically» — лучшая практика внедрения зависимостей в TypeScript: не нужно писать интерфейс вручную, сигнатуры автоматически отслеживают реализацию.

---

## 2.4 Ключевые алгоритмы и потоки выполнения

### 2.4.1 Полный поток выполнения основного цикла query()

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    query() вход
                        |
                +=======+========+  <--- начало основного цикла while(true)
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        проверка blocking-limit  |
                |                |
                v                |
        callModel() [потоково]   |
                |                |
        +-------+--------+      |
        | Потребление     |      |
        | stream-событий  |      |
        | сбор tool_use   |      |
        | streaming exec  |      |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           YES           |
       |              \          |
       v               v        |
  +---------+   runTools() или  |
  | Восст.  |   getRemainingR() |
  +---------+         |         |
       |              v         |
       |     getAttachments()   |
       |     prefetch памяти    |
       |     обнаружение skills |
       |              |         |
       v              v         |
  handleStopHooks()   |         |
       |         maxTurns?      |
       |              |         |
       v         state = next   |
  checkTokenBudget()  |         |
       |              +---------+
       v
  return Terminal { reason }
```

### 2.4.2 Цикл вызова инструментов (Tool-call Loop)

Ключевая инновация вызова инструментов — **streaming tool execution**: выполнение инструментов начинается *пока LLM ещё передаёт данные потоком*:

```typescript
// query.ts:443 (инициализация StreamingToolExecutor)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

В цикле потребления потока (`query.ts:536`):

```typescript
if (
  streamingToolExecutor &&
  !toolUseContext.abortController.signal.aborted
) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

Инструменты передаются на выполнение при `content_block_stop`, не дожидаясь завершения всего assistant-ответа. Это означает: если LLM выдал 3 блока tool_use, первые два могут быть уже выполнены, пока третий ещё передаётся потоком.

### 2.4.3 Конкретная реализация потоковой обработки

`claude.ts` вручную реализует SSE stream parsing, **намеренно обходя `BetaMessageStream` из Anthropic SDK**:

```typescript
// claude.ts комментарий (ок. строка 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

Модель накопления состояния потока:

```
message_start → partialMessage = part.message, usage = initial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → yield AssistantMessage (для каждого блока!)
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (завершение)
```

Ключевое решение: **каждый content block независимо порождает одно AssistantMessage**. Это означает: когда LLM-ответ содержит text + tool_use, UI может отобразить text сразу после его завершения, не дожидаясь завершения JSON для tool_use.

### 2.4.4 Пятиуровневый механизм повторов и восстановления после ошибок

Архитектура восстановления после ошибок Claude Code — это глубокая защита (defense in depth), состоящая из 5 уровней:

**Уровень 1: withRetry (уровень API)** — `withRetry()` в `claude.ts` обрабатывает 429 (rate limit), 529 (overload), 5xx и другие повторяемые ошибки, включая экспоненциальную задержку и model fallback.

**Уровень 2: Streaming → Non-streaming Fallback** — при разрыве потокового соединения (`claude.ts:2592`):

```typescript
// Fallback to non-streaming mode with retries
const result = yield* executeNonStreamingRequest(...)
```

Также включает stream idle watchdog (таймаут 90 с без данных) и fallback при 404 во время создания потока.

**Уровень 3: Восстановление при max_output_tokens** — трёхэтапное постепенное восстановление в `query.ts`:

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// Шаг 1: эскалация до 64k токенов (ESCALATED_MAX_TOKENS)
// Шаги 2-4: инъекция мета-сообщения с требованием "Resume directly — no apology, no recap"
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**Уровень 4: Восстановление при Prompt-too-long** — трёхступенчатая цепная реакция:
1. Context collapse drain (слив накопленных схлопываний)
2. Reactive compact (аварийное полное сжатие)
3. Выброс ошибки и выход (во избежание бесконечного цикла)

В исходном коде есть явная защита от бесконечных циклов (комментарий `query.ts`):

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**Уровень 5: Model Fallback** — в `query.ts:673-719` при перехвате `FallbackTriggeredError` происходит переключение на fallback model и повтор всего запроса.

### 2.4.5 Подсчёт токенов и управление бюджетом

Система бюджета токенов состоит из двух независимых механизмов:

**Механизм A: task_budget через API** — бюджет токенов, отслеживаемый на стороне сервера и пересекающий границы компрессии:

```typescript
// query.ts:270-280 (отслеживание taskBudgetRemaining через сжатие)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**Механизм B: Client-side auto-continue для бюджета токенов** (`tokenBudget.ts`) — автоматическое продолжение, когда вывод хода не достигает 90% бюджета:

```typescript
// tokenBudget.ts:46-62
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // Субагенты не используют auto-continue
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }
  // ...
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
  // ...
}
```

Обратите внимание на **обнаружение убывающей отдачи**: если 3 последовательных продолжения дают прирост менее 500 токенов каждое — автоматическая остановка, предотвращающая трату бюджета на малоэффективный вывод.

### 2.4.6 Обработка Thinking Mode

В исходном коде «волшебный комментарий» излагает 3 железных правила Thinking Mode (`query.ts:105-118`):

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

Логика построения параметров thinking в `claude.ts` (ок. строки 2242):

```typescript
if (
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING) &&
  modelSupportsAdaptiveThinking(options.model)
) {
  thinking = {
    type: 'adaptive',
  } satisfies BetaMessageStreamParams['thinking']
} else {
  let thinkingBudget = getMaxThinkingTokensForModel(options.model)
  // ...
  thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
  thinking = {
    budget_tokens: thinkingBudget,
    type: 'enabled',
  }
}
```

Модели с поддержкой adaptive thinking используют adaptive (без необходимости задавать бюджет заранее), иначе — fallback на enabled + budget_tokens.

### 2.4.7 Управление границами Prompt Cache

Стратегия кэширования Claude Code впечатляет. Ключевое решение — в `addCacheBreakpoints()` (`claude.ts:3045`):

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**Размещается только один маркер cache_control** — на последнем сообщении (или на предпоследнем при `skipCacheWrite`). Это результат совместной оптимизации с менеджером KV-кэш-страниц на стороне inference (Mycro).

Кэш с TTL 1 часа имеет тонкий механизм «фиксации стабильности сессии» (`claude.ts:380-420`) — eligibility определяется один раз и фиксируется на всю сессию, предотвращая переключение TTL cache_control из-за изменений конфигурации GrowthBook в середине сессии.

---

## 2.5 Анализ проектных решений

### 2.5.1 Ключевые компромиссы

**Компромисс 1: Потоковое выполнение против гарантии целостности**

StreamingToolExecutor начинает выполнять инструменты, пока LLM ещё генерирует вывод — это даёт значительное снижение задержки, но вносит сложность: при fallback в середине потока уже выполненные инструменты нужно откатить:

```typescript
// query.ts:534-538 (очистка при fallback в потоке)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

Эта проблема уже приводила к баге (см. комментарий `claude.ts:2575`, ссылающийся на inc-4258: двойное выполнение инструментов).

**Компромисс 2: Стабильность кэша против гибкости функций**

Несколько beta-заголовков используют паттерн «sticky-on latch» (`claude.ts:2102-2126`) — будучи активированы однажды, они остаются активными на всю сессию, даже если функция отключена:

```typescript
// claude.ts:2104 комментарий
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

Это явный компромисс в пользу «частоты попаданий в кэш над гибкостью функций».

**Компромисс 3: Конечный автомат против рекурсии**

Основной цикл эволюционировал от рекурсивных вызовов `query()` к `while(true)` + перестройка State. В исходном коде сохранилось название checkpoint `query_recursive_call`, но на деле это уже итерация. Преимущества:
- Нет риска переполнения стека (длинный диалог может иметь сотни ходов)
- Перестройка State явная и удобна для отладки
- Поле `transition` обеспечивает полный аудиторский след переходов состояний

### 2.5.2 Известные проблемы, раскрытые комментариями в исходном коде

1. **Дублирование текстовых delta в SDK** (`claude.ts:2350`):

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Конфликт non-streaming fallback с streaming tool execution** (`claude.ts:2575`):

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Погрешность подсчёта токенов бюджета задач при сжатии** (`query.ts:268`):

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 Различия в проектировании с LangChain и аналогами

| Измерение | Claude Code Query Engine | LangChain AgentExecutor |
|-----------|--------------------------|-------------------------|
| Примитив потоковой передачи | ES Async Generator (нативный) | Callback + Stream wrapper |
| Управление состоянием | Явная структура State + неизменяемые обновления | Изменяемый dict AgentState |
| Выполнение инструментов | Параллельное потоковое (StreamingToolExecutor) | Последовательный await |
| Повторы | 5-уровневая глубокая защита + model fallback | Простой max_iterations |
| Внедрение зависимостей | QueryDeps + автосинхронизация сигнатур через typeof | Duck typing в рантайме |
| Кэширование | Глубокая интеграция с KV-кэшем inference | Нет (чёрный ящик API) |

Фундаментальное отличие: Claude Code **inference-aware** — он понимает физический механизм Prompt Cache (менеджер страниц Mycro) и оптимизирует под него, тогда как фреймворки с открытым исходным кодом могут только работать с API как с чёрным ящиком.

---

## 2.6 Переносимые паттерны

### 2.6.1 Универсальные инженерные паттерны, извлечённые из Query Engine

**Паттерн 1: Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

Запись **причины** каждого перехода состояния в самом состоянии делает отладку и телеметрию первоклассными. Применимо к любой системе с многошаговым принятием решений.

**Паттерн 2: Типизированное внедрение зависимостей через `typeof`**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

Не нужно писать интерфейс вручную, сигнатуры автоматически синхронизируются с реализацией. Подходит для любой системы, требующей имитации тяжёлого I/O в тестах.

**Паттерн 3: Withholding Pattern (отложенное раскрытие ошибок)**

Для восстанавливаемых ошибок (prompt-too-long, max_output_tokens) — не передавать потребителю сразу, решать о раскрытии после выполнения логики восстановления:

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

Это предотвращает получение SDK-потребителем ложных сигналов об ошибке в ситуациях, когда ошибка уже устранена.

**Паттерн 4: Session-stable Latching**

Для конфигурационных элементов, влияющих на ключ кэша, — фиксировать однажды активированное значение на всю сессию:

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // запись в bootstrap state
}
```

Применимо к любому сценарию, где изменение конфигурации приводит к инвалидации дорогостоящих ресурсов.

### 2.6.2 Что можно взять в Doramagic

Конвейер `flow_controller` в Doramagic сталкивается со схожими задачами оркестрации, можно позаимствовать:

1. **Паттерн State + Transition**: 12-состоятельная машина Doramagic может использовать похожую структуру `{ state, transition: { reason } }`, упрощающую отладку и обеспечивающую аудиторский след
2. **Внедрение зависимостей через typeof**: слой вызовов LLM в Doramagic может применить паттерн QueryDeps, инжектируя fake model при тестировании вместо мокирования всего модуля
3. **Обнаружение убывающей отдачи** (`tokenBudget.ts`): итеративное уточнение в Soul Extractor Doramagic может использовать ту же стратегию «остановиться, если N последовательных приростов меньше порога» для экономии токенов на малокачественном выводе

---

## 2.7 Индекс исходного кода

| Файл | Строк | Ответственность в одной строке |
|------|-------|-------------------------------|
| `QueryEngine.ts` | 1 295 | Владелец на уровне сессии: хранит mutableMessages, totalUsage, транслирует submitMessage() в вызов query(), обрабатывает маршрутизацию SDK-сообщений |
| `query.ts` | 1 729 | Конечный автомат основного цикла: while(true) оркестрирует compaction → API call → tool execution → stop hooks → budget check |
| `query/config.ts` | 46 | Неизменяемый снимок QueryConfig: sessionId + 4 runtime gate (feature() gates намеренно исключены для сохранения tree-shaking) |
| `query/tokenBudget.ts` | 93 | Client-side auto-continue бюджета токенов: порог завершения 90% + ранняя остановка при убывающей отдаче |
| `query/stopHooks.ts` | 473 | Оркестрация хуков конца хода: Stop hooks → TaskCompleted hooks → TeammateIdle hooks, поддержка обратной инъекции блокирующих ошибок |
| `query/deps.ts` | 40 | Интерфейс инъекции 4 I/O-зависимостей: callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3 419 | Полный жизненный цикл API: построение параметров → создание потока → парсинг SSE → накопление content block → расчёт стоимости → non-streaming fallback → управление cache breakpoint |
| `cost-tracker.ts` | 323 | Накопление стоимости сессии: отслеживание использования по моделям, персистентность/восстановление сессии, форматированный вывод |
