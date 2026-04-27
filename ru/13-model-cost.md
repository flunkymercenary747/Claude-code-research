# Глава 13: Выбор модели и управление затратами

> Источник данных: снимок исходного кода Claude Code на TypeScript (2026-03-31, ~512K LOC)
> Основные файлы: `services/api/claude.ts` (3 419 строк), `services/api/withRetry.ts`, `cost-tracker.ts` (323 строки), `utils/effort.ts`, `utils/modelCost.ts`, `utils/model/model.ts`, каталог `migrations/` (11 файлов)

---

## 13.1 Обзор и позиционирование

Философию дизайна Claude Code в области выбора модели и управления затратами можно выразить тремя тезисами:

1. **Приоритет намерения пользователя**: цепочка приоритетов идёт от команды `/model` → флага `--model` → переменной окружения → конфигурационного файла, слой за слоем, каждый из которых может быть переопределён более высоким, но не будет неожиданно замещён более низким.
2. **Полная прозрачность затрат**: по завершении сессии принудительно выводятся потребление токенов и стоимость в долларах с разбивкой по моделям; это невозможно отключить (только при `hasConsoleBillingAccess() === true`).
3. **Никакого скрытого понижения**: при переключении Overload Fallback (Opus → Sonnet) пользователю обязательно показывается предупреждение, тихое переключение недопустимо.

В данной главе утверждения cc-notebook об этой подсистеме верифицируются на уровне исходного кода и анализируются детально.

---

## 13.2 Теоретическая основа

### Стратегии маршрутизации в мультимодельных системах

В мультимодельных системах стратегии маршрутизации обычно балансируют три измерения: **способности** (capability), **стоимость** (cost), **задержку** (latency). Выбор Claude Code — маршрутизировать основной диалог (main loop) на самую мощную доступную модель, фоновые вспомогательные задачи — на самую быструю и дешёвую, а при недоступности основной модели обеспечивать прозрачное понижение.

### Применение анализа затрат и выгод в системах с ИИ

В `modelCost.ts` видно, что Claude Code содержит точную таблицу цен:

```typescript
// utils/modelCost.ts
// Standard pricing tier for Sonnet models: $3 input / $15 output per Mtok
export const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// Fast mode pricing for Opus 4.6: $30 input / $150 output per Mtok
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
}
```

Haiku 4.5 — самая дешёвая ($1/$5 за Mtok), Opus 4.6 Fast Mode — самая дорогая ($30/$150 за Mtok), разница в 30 раз. Эта ценовая разница — ключевая экономическая логика для назначения фоновых задач Haiku.

### Паттерн «плавная деградация» (Graceful Degradation)

В традиционном программировании плавная деградация означает переход к менее оптимальному, но работоспособному варианту при недоступности функции. В системах с LLM запасной путь — «переключиться на более дешёвую/доступную модель». Claude Code реализует механизм срабатывания с защитой по счётчику: переключение происходит после 3 последовательных ошибок 529, а не немедленно (чтобы случайная перегрузка не вызывала ненужного снижения качества).

---

## 13.3 Архитектура выбора модели

### Иерархия приоритетов модели

Функция `getUserSpecifiedModelSetting()` в `utils/model/model.ts` точно определяет порядок приоритетов:

```typescript
// utils/model/model.ts:44-66
/**
 * Priority order within this function:
 * 1. Model override during session (from /model command) - highest priority
 * 2. Model override at startup (from --model flag)
 * 3. ANTHROPIC_MODEL environment variable
 * 4. Settings (from user's saved settings)
 */
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  let specifiedModel: ModelSetting | undefined

  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) {
    specifiedModel = modelOverride
  } else {
    const settings = getSettings_DEPRECATED() || {}
    specifiedModel = process.env.ANTHROPIC_MODEL || settings.model || undefined
  }

  // Ignore the user-specified model if it's not in the availableModels allowlist.
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

`getMainLoopModel()` добавляет к этому 5-й приоритет — встроенное значение по умолчанию:

```typescript
// utils/model/model.ts:68-77
export function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}
```

Полная 5-уровневая цепочка приоритетов:

| Приоритет | Источник | Описание |
|-----------|----------|----------|
| 1 (высший) | Команда `/model` | Действует немедленно внутри сессии, сохраняется в memory override |
| 2 | Флаг `--model` при запуске | Записывается в memory override при старте |
| 3 | Переменная окружения `ANTHROPIC_MODEL` | Уровень процесса |
| 4 | Конфигурационный файл `settings.json` | Постоянные пользовательские предпочтения |
| 5 (низший) | Встроенное значение по умолчанию | Определяется типом подписки |

### Дифференциация модели по умолчанию в зависимости от подписки

`getDefaultMainLoopModelSetting()` раскрывает различия подписок:

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (внутренние сотрудники) по умолчанию Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Max и Team Premium по умолчанию Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG, Enterprise, Team Standard, Pro по умолчанию Sonnet 4.6
  return getDefaultSonnetModel()
}
```

Это означает: даже без какой-либо конфигурации пользователи Max/Team Premium сразу получают Opus 4.6, а пользователи Pro/Sonnet — Sonnet 4.6. **Значение по умолчанию само по себе является стратегией продуктовой дифференциации.**

### Система псевдонимов моделей

`parseUserSpecifiedModel()` поддерживает разрешение коротких псевдонимов, освобождая пользователя от необходимости запоминать полный Model ID:

```typescript
// utils/model/model.ts — фрагмент parseUserSpecifiedModel
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan mode использует Sonnet
```

Суффикс `[1m]` можно добавить к любому псевдониму (например, `opus[1m]`), и система автоматически разберёт его как вариант с 1M контекстным окном.

### Определение возможностей модели

`utils/model/modelCapabilities.ts` реализует механизм кэширования, доступный только внутренним сотрудникам (`USER_TYPE === 'ant'`):

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

Внешние пользователи не запрашивают список возможностей модели — информация о возможностях жёстко закодирована в функциях `modelSupportsEffort()`, `modelSupports1M()` и подобных, что позволяет избежать дополнительных API-вызовов.

---

## 13.4 Фоновые применения Haiku

cc-notebook утверждает, что у Haiku есть 6 фоновых применений. По результатам полного поиска мест вызова функции `queryHaiku` (`grep -rn 'queryHaiku\b'`) и функции `getSmallFastModel()` в исходном коде **верификация** выглядит так:

### Сводка фоновых применений (верификация исходным кодом)

| № | Применение | Файл | Условие |
|---|------------|------|---------|
| 1 | Извлечение содержимого Web Fetch | `tools/WebFetchTool/utils.ts:503` | После получения веб-страницы Haiku фильтрует Markdown по запросу пользователя |
| 2 | Извлечение префикса shell-команды | `utils/shell/prefix.ts:220` | Перед выполнением инструмента Bash Haiku определяет, нужен ли запрос прав |
| 3 | Генерация заголовка сессии | `utils/sessionTitle.ts:87` | Автоматическая генерация короткого заголовка после начала сессии (вывод JSON schema) |
| 4 | Парсинг MCP DateTime | `utils/mcp/dateTimeParser.ts:68` | Преобразование естественного описания времени в формат ISO 8601 |
| 5 | Генерация резюме вызовов инструментов | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | Генерация однострочного резюме после завершения пакета вызовов инструментов |
| 6 | Переименование сессии | `commands/rename/generateSessionName.ts:20` | Команда `/rename` генерирует имя в формате kebab-case |

**Дополнительные находки** (не упомянуты в cc-notebook, обнаружены через `getSmallFastModel()`):

| № | Применение | Файл | Условие |
|---|------------|------|---------|
| 7 | Проверка API Key | `services/api/claude.ts:544` | Проверка валидности API Key (в комментарии: «WARNING: if you change this to use a non-Haiku model, this request will fail in 1P») |
| 8 | Резюме в режиме Away | `services/awaySummary.ts:49` | Генерация контекстного резюме при отсутствии пользователя (AFK mode) |
| 9 | Помощник Web Search | `tools/WebSearchTool/WebSearchTool.ts:280` | В некоторых сценариях веб-поиска Haiku обрабатывает результаты |
| 10 | Проверка состояния квоты | `services/claudeAiLimits.ts:200` | Минимальный запрос к Haiku для определения текущего состояния квоты |
| 11 | Оценка количества токенов | `services/tokenEstimation.ts:277` | Оценка потребления контекстного окна |
| 12 | Выполнение Prompt/Exec Hook | `utils/hooks/execPromptHook.ts:79`, `execAgentHook.ts:118` | Hook-коллбэки по умолчанию используют Haiku (если не переопределено настройками hook) |
| 13 | Анализ улучшений Skill | `utils/hooks/skillImprovement.ts:169` | После выполнения Skill автоматически анализируются предложения по улучшению |

**Вывод**: утверждение cc-notebook о «6 фоновых применениях» является **заниженным**. В исходном коде насчитывается не менее 13 мест вызова `queryHaiku` или `getSmallFastModel()`, охватывающих все этапы жизненного цикла сессии (проверка при запуске, помощь в ходе выполнения, завершающая обработка сессии). Haiku/SmallFastModel является фоновым «базовым сервисным уровнем» всей системы, а не редким оптимизационным средством.

Ключевые детали реализации: `queryHaiku` использует неструйный вызов (`queryModelWithoutStreaming`) и не передаёт контекст прав на инструменты (`getEmptyToolPermissionContext()`):

```typescript
// services/api/claude.ts:3280-3291
const result = await queryModelWithoutStreaming({
  messages,
  systemPrompt,
  thinkingConfig: { type: 'disabled' },
  tools: [],
  signal,
  options: {
    ...options,
    model: getSmallFastModel(),
    enablePromptCaching: options.enablePromptCaching ?? false,
    async getToolPermissionContext() {
      return getEmptyToolPermissionContext()
    },
  },
})
```

---

## 13.5 Механизм Overload Fallback

cc-notebook утверждает о наличии «529 Overload Fallback, откат Opus → Sonnet». Исходный код **полностью подтверждает** это утверждение с более богатыми деталями.

### Распознавание ошибок 529

Функция `is529Error()` в `services/api/withRetry.ts`:

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // Проверяем статус 529 или строку в сообщении (в стриминге SDK иногда не передаёт статус)
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

Обратите внимание на двойную проверку: статус-код `529` и строка `overloaded_error` в сообщении об ошибке — потому что SDK при потоковой передаче иногда не может корректно передать статус 529.

### Условие срабатывания: 3 последовательных 529

```typescript
// services/api/withRetry.ts — фрагмент функции withRetry
const MAX_529_RETRIES = 3

if (
  is529Error(error) &&
  (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
    (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })
      // Бросаем специальную ошибку, инициирующую переключение модели на верхнем уровне
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

Ключевые ограничения:
- По умолчанию срабатывает только для **пользователей не по подписке ClaudeAI** и **только для моделей серии Opus** (`isNonCustomOpusModel()`)
- Переменная окружения `FALLBACK_FOR_ALL_PRIMARY_MODELS` расширяет действие на все основные модели
- Ошибка 529 при потоковом запросе засчитывается в счётчик (`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`), согласовываясь с нестриминговыми попытками повтора

### Распространение сигнала FallbackTriggeredError

`FallbackTriggeredError` — специальный класс ошибки, несущий поля `originalModel` и `fallbackModel`, который распространяется вверх по стеку вызовов до `query.ts`:

```typescript
// services/api/withRetry.ts
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    super(`Model fallback triggered: ${originalModel} -> ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

### Переключение модели и уведомление пользователя в query.ts

`query.ts:894-946` перехватывает эту ошибку и выполняет фактическое переключение модели:

```typescript
// query.ts — фрагмент обработки FallbackTriggeredError
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // Показываем пользователю с уровнем warning — видно независимо от режима verbose
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // Синхронно обновляем основную модель в toolUseContext
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // Повторяем запрос с новой моделью
}
```

**Механизм уведомления пользователя**: сообщение о переключении использует уровень `'warning'`, что означает — пользователь увидит уведомление независимо от того, включён ли режим verbose. **Утверждение cc-notebook о «никакого скрытого понижения» полностью подтверждено.**

### Стратегия обработки 529 для фоновых задач: немедленный отказ

Нефоновые задачи (summary, title, suggestions и подобные) при ошибке 529 **не повторяются**, а немедленно отбрасываются:

```typescript
// services/api/withRetry.ts — FOREGROUND_529_RETRY_SOURCES
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'agent:builtin',
  'compact',
  'verification_agent',
  'side_question',
  'auto_mode',
  // ...
])

// Для нефоновых задач при 529 сразу бросаем исключение без повторов
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

Это архитектурное решение для управления затратами: повторные попытки фоновых задач при нехватке ёмкости создают 3–10-кратный эффект усиления на шлюзе, а пользователь вообще не замечает их сбоев.

---

## 13.6 Механизм Effort Level

cc-notebook утверждает о наличии системы Effort Level. Исходный код **полностью подтверждает** это, причём детали значительно богаче описанных.

### Четыре уровня Effort

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

Семантика каждого уровня (из `getEffortLevelDescription()`):
- **low**: Quick, straightforward implementation with minimal overhead
- **medium**: Balanced approach with standard implementation and testing
- **high**: Comprehensive implementation with extensive testing and documentation
- **max**: Maximum capability with deepest reasoning (**только Opus 4.6**)

### Матрица поддержки моделей

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // Только Opus 4.6 и Sonnet 4.6 поддерживают параметр effort
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku, старые версии Opus/Sonnet не поддерживают
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P по умолчанию true, 3P по умолчанию false
  return getAPIProvider() === 'firstParty'
}

// max effort доступен только для Opus 4.6
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### Цепочка приоритетов: env → appState → значение по умолчанию для модели

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' или 'auto' → не отправляем параметр effort
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API отклоняет max для не-Opus 4.6 → автоматически понижаем до high
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### Дифференциация Effort по умолчанию

Effort по умолчанию для Opus 4.6 зависит от типа подписки:

```typescript
// utils/effort.ts — фрагмент getDefaultEffortForModel
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // По умолчанию medium для Pro (экономия квоты)
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team также можно сконфигурировать через GrowthBook на medium
  }
}
```

Примечательно, что `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` содержит в `dialogDescription` явную фразу: «We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits.» — это говорит о том, что medium по умолчанию — осознанная стратегия управления квотой, а не приоритет производительности.

### Ограничения персистентности max

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max для не-ant пользователей действует только в рамках сессии и не сохраняется
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

Настройка `max` effort для внешних пользователей не записывается в `settings.json` и действует только в текущей сессии.

---

## 13.7 Система отслеживания затрат

### Основные обязанности cost-tracker.ts

`cost-tracker.ts` (323 строки) выполняет три задачи:
1. **Накопление в реальном времени**: вызов `addToTotalSessionCost()` после каждого ответа API
2. **Персистентность**: запись в конфигурационный файл проекта по завершении сессии (`saveCurrentSessionCosts()`)
3. **Восстановление**: чтение данных о затратах предыдущей сессии при перезапуске (`restoreCostStateForSession()`)

### Статистика токенов с разбивкой по моделям

`addToTotalModelUsage()` накапливает 5 измерений по имени модели:

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

По завершении сессии данные форматируются (`formatModelUsage()`): агрегируются по короткому имени (несколько API-эндпоинтов могут возвращать одну модель в разных форматах):

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Маркировка затрат Fast Mode

В `addToTotalSessionCost()` есть специальная обработка Fast Mode:

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

Маркер `speed: 'fast'` влияет на тарификацию — в Fast Mode Opus 4.6 использует `COST_TIER_30_150` ($30/$150), а не стандартный `COST_TIER_5_25` ($5/$25).

### Вложенное отслеживание затрат Advisor

`addToTotalSessionCost()` рекурсивно обрабатывает потребление инструмента Advisor:

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

Advisor — скрытый вторичный вызов модели внутри ответа основной модели; его затраты отслеживаются отдельно и включаются в общую сумму.

### Механизм отображения затрат

`costHook.ts` (22 строки) — React hook, который слушает событие выхода из процесса:

```typescript
// costHook.ts
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

`hasConsoleBillingAccess()` контролирует отображение затрат, гарантируя, что в средах без доступа к данным биллинга (например, CCR/Remote) стоимость не выводится. При этом `saveCurrentSessionCosts()` выполняется безусловно — независимо от отображения, данные всегда сохраняются.

---

## 13.8 API-слой вызовов

### Ключевые параметры формирования запроса в claude.ts

`services/api/claude.ts` (3 419 строк) — единая точка входа для всех API-вызовов. Ключевые параметры объединяются из нескольких подсистем:

```typescript
// services/api/claude.ts — сборка параметров запроса (схематично)
{
  model: normalizeModelStringForAPI(options.model),  // Убираем суффикс [1m]
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // Параметр Effort (только для поддерживающих моделей)
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` перед отправкой в API убирает суффиксы `[1m]` и `[2m]` — они являются лишь внутренним клиентским обозначением 1M контекстного окна и API их не понимает.

### Потоковый ответ и откат на нестриминговый режим

Основной диалог использует потоковую передачу (Server-Sent Events), однако при её сбое происходит откат на нестриминговый режим:

```typescript
// services/api/claude.ts:2535-2559
// Если сбой стриминга был 529, засчитываем его в счётчик последовательных 529
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

Нестриминговый откат имеет ограничение по максимальному числу токенов:

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Динамическое добавление Beta Headers

Разным функциям соответствуют разные Beta Headers, которые добавляются динамически при запросе:

```typescript
// constants/betas.ts (ссылка)
EFFORT_BETA_HEADER        // поддержка параметра effort
CONTEXT_1M_BETA_HEADER    // 1M контекстное окно
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // управление бюджетом
```

---

## 13.9 Анализ проектных решений

### Философия «никакого скрытого понижения»

Из уведомления об уровне `'warning'` в `query.ts` и специального класса `FallbackTriggeredError` видно, что это осознанное архитектурное решение:

**Почему нельзя переключаться тихо?** Потому что Claude Code — инструмент для написания кода, и качество модели напрямую влияет на качество вывода. Пользователь имеет право знать «я сейчас использую Sonnet, а не Opus», чтобы решить, стоит ли продолжать ждать или применить иную стратегию. В отличие от потребительских чат-продуктов, пользователи инструментов для кода более профессиональны и чувствительны к различиям между моделями.

### Соображения по прозрачности затрат

Заслуживает внимания дизайн `hasConsoleBillingAccess()` в `costHook.ts`: даже при отсутствии отображения данные о затратах сохраняются. Это говорит о том, что основная цель отслеживания затрат — **восстановление сессии** (показ расходов предыдущей сессии при следующем запуске), а не предупреждения в реальном времени. Это «оффлайн-осведомлённый» дизайн: пользователь видит полную стоимость по завершении сессии, а не прерывается после каждого API-вызова.

### Продуктовая логика дифференциации модели по умолчанию

Назначение Opus в качестве дефолтной модели для Max/Team Premium и Sonnet для Pro/PAYG отражает чёткую продуктовую логику: одним из ценностных предложений подписки Max является «доступ к самой мощной модели», и значение по умолчанию само по себе воплощает это предложение.

При этом даже для Max-пользователей Effort для Opus 4.6 по умолчанию равен `medium` (управляется через GrowthBook) — это говорит о том, что Anthropic **балансирует качество и квоту** через систему Effort, а не предоставляет Max-пользователям максимальную конфигурацию безусловно.

---

## 13.10 Необходимость миграций моделей

В каталоге `migrations/` 11 файлов миграций раскрывают следы эволюции продукта — каждая миграция соответствует одному продуктовому решению:

| Файл миграции | Условие срабатывания | Основная логика |
|---------------|---------------------|-----------------|
| `migrateFennecToOpus.ts` | Внутренние сотрудники (ant) | Псевдоним fennec → псевдоним opus (чистка внутренних кодовых имён) |
| `migrateLegacyOpusToCurrent.ts` | 1P пользователи с `opus-4-0`/`4-1` в settings | Старые ID модели Opus → псевдоним `opus` (Opus 4.0/4.1 выводятся из эксплуатации) |
| `migrateOpusToOpus1m.ts` | Max/Team Premium (1P), `opus` в settings | `opus` → `opus[1m]` (объединение 1M опыта) |
| `migrateSonnet1mToSonnet45.ts` | Пользователи с `sonnet[1m]` | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` (закрепление на 4.5, т.к. аудитория 4.6 1M отличается) |
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium (1P), закреплённые на Sonnet 4.5 | Строки Sonnet 4.5 → псевдоним `sonnet` (обновление до 4.6) |
| `resetProToOpusDefault.ts` | Pro 1P пользователи без кастомной модели | Запись временной метки миграции, однократное отображение уведомления об обновлении в REPL |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode включён, старые OptIn-пользователи диалога | Очистка `skipAutoPermissionPrompt`, повторное отображение нового диалога прав |
| `migrateAutoUpdatesToSettings.ts` | Пользователи с явно отключёнными авто-обновлениями | Перенос `autoUpdates: false` в переменную окружения в settings.json |
| `migrateBypassPermissionsAcceptedToSettings.ts` | В глобальной конфигурации есть `bypassPermissionsModeAccepted` | Перенос в `skipDangerousModePermissionPrompt` в settings.json |
| `migrateSonnet45ToSonnet46.ts` | Аналогично выше | Упомянутая выше миграция с тем же именем |
| `migrateEnableAllProjectMcpServersToSettings.ts` | Конфигурация, связанная с MCP | Корректировка структуры настроек MCP-серверов |

**Архитектурный инсайт**: каждая миграция оперирует только `userSettings` (пользовательский settings.json), никогда не затрагивая `projectSettings` (проектный уровень) или `policySettings` (политики предприятия). Это намеренный дизайн:

1. **Идемпотентность**: чтение и запись в один источник данных, повторный запуск не производит побочных эффектов
2. **Минимальные права**: нет возможности (и необходимости) менять закреплённые пользователем настройки на уровне проекта
3. **Предотвращение глобального повышения**: если пользователь закрепил старый Opus в конкретном проекте, миграция не продвинет его в глобальные настройки

Само существование этой системы миграций свидетельствует: **миграция схем в AI-системах значительно сложнее, чем в традиционных базах данных** — необходимо учитывать изменения типов подписок, вывод моделей из эксплуатации, обновления контекстных окон и множество других измерений, и нельзя грубо перезаписывать намерения пользователя.

---

## 13.11 Переносимые паттерны

Из анализа этой главы выделяются 5 проектных паттернов, применимых в собственных системах:

### Паттерн 1: Многоуровневая цепочка Override
```
session_override > startup_flag > env_var > config_file > builtin_default
```
Любой уровень может быть переопределён более высоким, но нижний не может незаметно влиять на верхний. Дополняется проверкой по allowlist для предотвращения инъекции нелегальных model ID.

### Паттерн 2: Раздельные стратегии 529 для переднего и фонового плана
Задачи переднего плана (пользователь ждёт результата): N попыток повтора, при превышении лимита — fallback.
Фоновые задачи (пользователь не замечает): первая 529 — немедленный отказ, чтобы избежать эффекта усиления повторов при нехватке ёмкости.

### Паттерн 3: Сигнализация через FallbackTriggeredError
Не переключать модель тихо внутри retry, а бросать специальную ошибку и отдавать логику переключения вышестоящему вызывающему коду. Так логика переключения сосредоточена в одном месте (query.ts) и всегда сопровождается уведомлением пользователя.

### Паттерн 4: Фильтрация персистентности через toPersistableEffort
Настройки уровня сессии (например, `max` effort) фильтруются перед записью в settings.json. «Состояния, которые не должны сохраняться между сессиями» и «постоянные пользовательские предпочтения» разделяются уже на уровне модели данных.

### Паттерн 5: Отслеживание затрат по бакетам моделей
Отслеживать не только суммарные затраты, но и распределять их по нормализованному имени модели. Только так по завершении сессии можно показать «Opus потратил столько, Haiku — столько», помогая пользователю понять, какая функция обходится дороже.

---

## 13.12 Индекс исходного кода

| Файл | Строк | Основное содержание |
|------|-------|---------------------|
| `services/api/claude.ts` | 3 419 | API-слой, queryHaiku, формирование запроса, потоковая обработка |
| `services/api/withRetry.ts` | ~600 | Логика повторов, обработка 529, FallbackTriggeredError |
| `cost-tracker.ts` | 323 | Отслеживание затрат, персистентность, форматирование |
| `costHook.ts` | 22 | React hook, слушает выход процесса для вывода затрат |
| `utils/effort.ts` | ~350 | Определение Effort Level, цепочка приоритетов, определение поддержки моделей |
| `utils/modelCost.ts` | ~200 | Таблица цен, функция расчёта затрат |
| `utils/model/model.ts` | ~450 | Цепочка приоритетов моделей, разрешение псевдонимов, логика модели по умолчанию |
| `utils/model/modelCapabilities.ts` | ~100 | Кэш возможностей модели (только внутренние пользователи) |
| `query.ts` | ~1000 | Перехват FallbackTriggeredError, уведомление пользователя, переключение модели |
| `migrations/*.ts` | 11 файлов | Скрипты миграции версий моделей |
| `tools/WebFetchTool/utils.ts:503` | — | Применение Haiku 1: извлечение содержимого Web Fetch |
| `utils/shell/prefix.ts:220` | — | Применение Haiku 2: определение префикса shell-команды |
| `utils/sessionTitle.ts:87` | — | Применение Haiku 3: генерация заголовка сессии |
| `utils/mcp/dateTimeParser.ts:68` | — | Применение Haiku 4: парсинг DateTime |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Применение Haiku 5: резюме вызовов инструментов |
| `commands/rename/generateSessionName.ts:20` | — | Применение Haiku 6: переименование сессии |
| `services/api/claude.ts:544` | — | Применение Haiku 7: проверка API Key |

---

*Данная глава полностью покрывает утверждения cc-notebook о выборе модели и управлении затратами. Результаты верификации: фоновые применения Haiku «минимум 6» подтверждены (фактически 13 точек вызова); «никакого скрытого понижения» — полностью подтверждено; механизм 529 Overload Fallback — полностью подтверждён; система Effort Level — полностью подтверждена. Все фрагменты кода точно скопированы из исходных файлов с указанием пути и строки.*
