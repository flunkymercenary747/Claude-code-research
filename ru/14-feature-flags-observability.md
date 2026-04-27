# Глава 14: Feature Flags и наблюдаемость

## 14.1 Обзор и позиционирование

Система наблюдаемости Claude Code — многоуровневая, многоцелевая система, охватывающая весь путь от обрезки функций на этапе компиляции до отслеживания поведения во время выполнения. Вся система держится на трёх столпах:

1. **Система Feature Flag**: двухрежимный дизайн — компиляционные вызовы `feature()` (реализующие Dead Code Elimination через `bun:bundle`) и динамическая конфигурация GrowthBook во время выполнения. Первые контролируют границы функций, доступных разным группам пользователей; вторые позволяют переключать флаги без повторного развёртывания.

2. **Пайплайн наблюдаемости**: на базе стандарта OpenTelemetry, поддерживает три протокола экспорта (gRPC/HTTP/Protobuf), унифицированно собирает три типа сигналов (Metrics, Logs, Traces), а также предоставляет внутренний слой отладки в формате трассировки Perfetto.

3. **Сбор Analytics**: двухпутная маршрутизация — Datadog (внешний мониторинг) + собственное логирование событий 1P (внутренний BigQuery/proto). Все бизнес-события идентифицируются по префиксу `tengu_*` в имени события; динамическая конфигурация GrowthBook управляет объёмом данных.

Ключевой принцип дизайна этой системы — **послойная изоляция**: приоритет конфиденциальности пользователя (код и пути к файлам по умолчанию не записываются), различия между внутренними и внешними сборками (ant-only vs external), graceful degradation (каждый уровень имеет kill-switch).

---

## 14.2 Теоретическая основа

### Feature Flag Driven Development

Feature Flag (флаги функций) позволяют команде параллельно разрабатывать функции на разных стадиях в одной кодовой базе и включать их по необходимости. Claude Code использует двухуровневый механизм флагов:

- **Компиляционные флаги**: вызовы `feature()`, предоставляемые `bun:bundle`, выполняют Dead Code Elimination на этапе упаковки. Целые блоки кода, отсутствующие во внешних версиях, полностью удаляются — это не только уменьшает размер пакета, но и предотвращает реверс-инжиниринг внутренней логики.
- **Флаги времени выполнения**: через SDK GrowthBook динамически получаются с сервера, поддерживая A/B-тестирование, постепенный выкат, аварийные kill-switch и другие сценарии.

### Три столпа наблюдаемости

Сообщество OpenTelemetry определяет наблюдаемость через три типа сигналов:

- **Metrics (метрики)**: числовые данные временных рядов, например, задержка API, потребление токенов. Claude Code использует `@opentelemetry/sdk-metrics` с PeriodicExportingMetricReader, экспортируя данные каждые 60 секунд.
- **Logs (логи)**: структурированные записи событий. Все вызовы `logEvent()` в конечном счёте пакетно экспортируются через OTel `LoggerProvider` + `BatchLogRecordProcessor`.
- **Traces (трассировки)**: цепочки распределённых вызовов. Claude Code строит иерархическое дерево Span Interaction → LLM Request → Tool Call через `sessionTracing.ts`, поддерживая отслеживание отношений родитель-потомок в многоагентных сценариях.

### A/B-тестирование в CLI-инструментах

В отличие от веб-продуктов, A/B-тестирование CLI-инструментов сталкивается с уникальными вызовами: нет отпечатка браузера, несколько платформ и каналов распространения, сценарии оффлайн-работы. Стратегия Claude Code:

- Таргетинг по пользователям: `GrowthBookUserAttributes` несёт атрибуты `platform`, `subscriptionType`, `rateLimitTier` и другие, поддерживая многоуровневые эксперименты.
- Локальный кэш на диске: после каждого успешного получения значений с сервера они записываются в `cachedGrowthBookFeatures` в `~/.claude/config.json`, гарантируя использование последних известных значений в оффлайне.
- Дедупликация показов: событие показа эксперимента для каждого feature в рамках одной сессии записывается только один раз (Set `loggedExposures`).

---

## 14.3 Система Feature Flag

### Интеграция GrowthBook

GrowthBook — платформа Feature Flag и A/B-тестирования с открытым исходным кодом. Claude Code интегрирует её через официальный SDK `@growthbook/growthbook`, файл находится в `src/services/analytics/growthbook.ts` (1155 строк).

**Процесс инициализации**:

```typescript
// growthbook.ts:529-600 (упрощено)
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

Ключевой дизайн: `memoize` гарантирует инициализацию клиента GrowthBook только один раз за жизненный цикл процесса; при изменении аутентификации (вход/выход) клиент уничтожается и пересоздаётся через `refreshGrowthBookAfterAuthChange()`, а не обновляется `apiHostRequestHeaders` (SDK не поддерживает обновление после инициализации).

**Модель атрибутов пользователя** (`growthbook.ts:31-46`):

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

**Стратегия обновления**:
- Внешние пользователи: обновление каждые 6 часов (`6 * 60 * 60 * 1000`)
- Внутренние сотрудники (ant): обновление каждые 20 минут

**Архитектура кэша** (три уровня приоритета):
1. Map `remoteEvalFeatureValues` в памяти (актуальное значение внутри процесса)
2. Дисковый кэш `cachedGrowthBookFeatures` в `~/.claude/config.json` (межпроцессная персистентность)
3. Устаревший `cachedStatsigGates` (слой совместимости миграции, постепенно выводится из эксплуатации)

**API Compatibility Workaround** (`growthbook.ts:320-390`): ответ remoteEval сервера использует поле `value`, тогда как SDK ожидает `defaultValue`; в коде присутствует явная логика преобразования формата с TODO-комментарием в ожидании исправления на сервере.

**Переопределение переменными окружения** (только для внутренних ant-пользователей):
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### Полный список компиляционных Feature Flags (80+)

Через вызовы `feature()` в `bun:bundle` реализуется удаление мёртвого кода. Ниже приведены все компиляционные флаги, извлечённые из исходного кода:

| Имя флага | Расположение | Контролируемая функция |
|-----------|-------------|----------------------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Трассировка Perfetto (только ant) |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | Бета расширенной телеметрии |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | Система автоматического режима/задач по расписанию |
| `KAIROS_BRIEF` | `commands.ts` | Краткий режим KAIROS |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | Поддержка каналов KAIROS |
| `KAIROS_DREAM` | `commands.ts` | Режим мечты KAIROS |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | Подписка на GitHub webhooks |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | Push-уведомления KAIROS |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Триггеры Agent (задачи по расписанию) |
| `AGENT_TRIGGERS_REMOTE` | — | Удалённые триггеры Agent |
| `AGENT_MEMORY_SNAPSHOT` | — | Снимок памяти Agent |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | Классификатор диалога |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | Агент верификации |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | Встроенные агенты исследования/планирования |
| `COORDINATOR_MODE` | `builtInAgents.ts` | Режим координатора |
| `FORK_SUBAGENT` | `commands.ts` | Fork подагента |
| `BUDDY` | `commands.ts` | Функция Buddy |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Входящие через Unix Domain Socket |
| `BRIDGE_MODE` | `commands.ts` | Режим моста (CCR) |
| `DAEMON` | `commands.ts` | Режим демона |
| `VOICE_MODE` | `commands.ts` | Голосовой режим |
| `ULTRAPLAN` | `commands.ts` | Команда UltraPlan |
| `ULTRATHINK` | — | Функция UltraThink |
| `TORCH` | `commands.ts` | Команда TORCH (динамическая загрузка) |
| `MCP_SKILLS` | `commands.ts` | Поддержка MCP Skill |
| `CHICAGO_MCP` | `metadata.ts` | Встроенный сервер Chicago MCP (computer-use) |
| `WORKFLOW_SCRIPTS` | `commands.ts` | Скрипты рабочих процессов |
| `CCR_REMOTE_SETUP` | `commands.ts` | Команда удалённой настройки CCR |
| `CCR_AUTO_CONNECT` | — | Автоподключение CCR |
| `CCR_MIRROR` | — | Зеркальный режим CCR |
| `PROACTIVE` | `commands.ts` | Проактивный режим |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | Экспериментальный поиск по навыкам |
| `HISTORY_SNIP` | `commands.ts` | Фрагменты истории |
| `HISTORY_PICKER` | — | Выбор из истории |
| `WEB_BROWSER_TOOL` | — | Инструмент веб-браузера |
| `QUICK_SEARCH` | — | Быстрый поиск |
| `MONITOR_TOOL` | — | Инструмент мониторинга |
| `OVERFLOW_TEST_TOOL` | — | Инструмент тестирования переполнения |
| `BREAK_CACHE_COMMAND` | — | Команда принудительного разрыва кэша |
| `TREE_SITTER_BASH` | — | Парсинг Bash через Tree-sitter |
| `TREE_SITTER_BASH_SHADOW` | — | Теневое сравнение Tree-sitter |
| `BASH_CLASSIFIER` | — | Классификатор безопасности Bash |
| `TERMINAL_PANEL` | — | Терминальная панель |
| `NATIVE_CLIPBOARD_IMAGE` | — | Нативная поддержка изображений из буфера обмена |
| `NATIVE_CLIENT_ATTESTATION` | — | Нативная аттестация клиента |
| `AUTO_THEME` | — | Автоматическая тема |
| `POWERSHELL_AUTO_MODE` | — | Автоматический режим PowerShell |
| `TOKEN_BUDGET` | — | Отображение бюджета токенов |
| `STREAMLINED_OUTPUT` | — | Упрощённый режим вывода |
| `CONNECTOR_TEXT` | — | Текст коннектора |
| `CONTEXT_COLLAPSE` | — | Сворачивание контекста |
| `COMPACTION_REMINDERS` | — | Напоминания о сжатии |
| `CACHED_MICROCOMPACT` | — | Кэшированная микрокомпрессия |
| `REACTIVE_COMPACT` | — | Реактивное сжатие |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Обнаружение разрыва Prompt Cache |
| `EXTRACT_MEMORIES` | — | Автоматическое извлечение памяти |
| `LODESTONE` | — | Функция Lodestone |
| `TEAMMEM` | — | Командная память |
| `TEMPLATES` | — | Шаблоны |
| `FILE_PERSISTENCE` | — | Персистентность файлов |
| `BG_SESSIONS` | — | Фоновые сессии |
| `DOWNLOAD_USER_SETTINGS` | — | Загрузка пользовательских настроек |
| `UPLOAD_USER_SETTINGS` | — | Выгрузка пользовательских настроек |
| `NEW_INIT` | — | Новый процесс инициализации |
| `HARD_FAIL` | — | Режим жёсткого отказа |
| `SLOW_OPERATION_LOGGING` | — | Логирование медленных операций |
| `SHOT_STATS` | — | Статистика запросов |
| `MEMORY_SHAPE_TELEMETRY` | — | Телеметрия формы памяти |
| `COWORKER_TYPE_TELEMETRY` | — | Телеметрия типа соавтора |
| `ANTI_DISTILLATION_CC` | — | Защита от дистилляции |
| `RUN_SKILL_GENERATOR` | — | Генератор навыков |
| `SKILL_IMPROVEMENT` | — | Улучшение навыков |
| `REVIEW_ARTIFACT` | — | Артефакт ревью кода |
| `MESSAGE_ACTIONS` | — | Действия с сообщениями |
| `AWAY_SUMMARY` | — | Резюме в режиме Away |
| `COMMIT_ATTRIBUTION` | — | Атрибуция коммита |
| `UNATTENDED_RETRY` | — | Повтор без пользователя |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | Определение типа libc (вставляется при сборке) |

### Флаги времени выполнения vs компиляционные флаги

| Измерение | Компиляционные (`feature()`) | Флаги времени выполнения (GrowthBook) |
|-----------|------------------------------|---------------------------------------|
| Момент выполнения | Этап упаковки | Асинхронная загрузка после старта процесса |
| Сохранение кода | Удалённые ветки не существуют в артефакте | Код присутствует, но логика управляется значением флага |
| Способ обновления | Выпуск новой версии | Серверный push, вступает в силу за 20 минут |
| Типичное применение | Функции только для ant, экспериментальные инструменты, платформозависимый код | A/B-тесты, постепенный выкат, kill-switch, динамическая конфигурация |
| Способ переопределения | Переменная сборки | Переменная окружения `CLAUDE_INTERNAL_FC_OVERRIDES` (только ant) |

### Механизм Dead Code Elimination

`feature()` в `bun:bundle` — специальная встроенная функция упаковщика Bun. На этапе сборки она заменяет `feature('X')` на `true` или `false` в соответствии с определениями build-time, после чего постоянная свёртка и удаление мёртвого кода срезают ветки, всегда дающие `false`.

Пример (`perfettoTracing.ts:216-220`):
```typescript
// Во внешних сборках этот блок if полностью удаляется
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... весь код инициализации Perfetto
}
```

Этот механизм не только уменьшает размер пакета, но и предотвращает утечку кода внутренних инструментов во внешние артефакты.

### Известные важные флаги времени выполнения

Ниже перечислены некоторые известные флаги GrowthBook с префиксом `tengu_*` и их функции:

| Имя флага | Тип | Описание функции |
|-----------|-----|-----------------|
| `tengu_auto_mode_config` | Object | Конфигурация автоматического режима (enabled/opt-in) |
| `tengu_1p_event_batch_config` | Object | Конфигурация пакетного экспорта событий 1P |
| `tengu_event_sampling_config` | Object | Словарь конфигурации частоты выборки событий |
| `tengu_log_datadog_events` | Boolean | Переключатель передачи событий в Datadog |
| `tengu_frond_boric` | Object | Kill-switch аналитического приёмника (отключение по имени приёмника) |
| `tengu_quartz_lantern` | Boolean | Управление поведением атомарной записи FileWriteTool |
| `tengu_hive_evidence` | Boolean | Управление поведением записи обновлений задач/Todo |
| `tengu_plum_vx3` | Boolean | Переключатель использования модели Haiku для WebSearchTool |
| `tengu_kairos_cron` | Object | Конфигурация расписания KAIROS |
| `tengu_kairos_cron_durable` | Boolean | Поддержка долговечных задач по расписанию |
| `tengu_agent_list_attach` | Boolean | Поведение прикрепления списка AgentTool |
| `tengu_amber_stoat` | Boolean | Управление доступностью встроенных агентов |
| `tengu_slim_subagent_claudemd` | Boolean | Упрощённая загрузка CLAUDE.md подагентом |
| `tengu_glacier_2xr` | Boolean | Управление решением о режиме ToolSearch |
| `tengu_max_version_config` | Object | Ограничение максимальной версии (kill-switch принудительного обновления) |
| `tengu_prompt_cache_1h_config` | Object | Конфигурация Prompt Cache на 1 час |
| `tengu_sm_compact_config` | Object | Конфигурация сжатия Session Memory |
| `tengu_ant_model_override` | String | Переопределение модели только для ant |
| `enhanced_telemetry_beta` | Boolean | Переключатель бета расширенной телеметрии |

---

## 14.4 Система наблюдаемости

### Интеграция OpenTelemetry

Claude Code полностью реализует поддержку трёх сигналов OpenTelemetry; основная точка входа — `src/utils/telemetry/instrumentation.ts` (825 строк).

**Инициализационная загрузка** (`instrumentation.ts:bootstrapTelemetry()`):
В ant-сборках конфигурация читается из переменных с префиксом `ANT_OTEL_*` и сопоставляется со стандартными переменными `OTEL_*`. Для внешних пользователей соблюдается стандартная спецификация переменных окружения OTel; по умолчанию temporality установлена в `delta` (инкрементная, а не накопительная).

**Конфигурация трёх экспортёров сигналов** (ленивая загрузка):

```typescript
// instrumentation.ts:169-190 (упрощено)
// Экспортёры OTLP/Prometheus используют динамический import с ленивой загрузкой
// для избежания загрузки @grpc/grpc-js (~700KB) при отсутствии необходимости
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

Поддерживаются все три протокола: `grpc`, `http/json`, `http/protobuf`, выбираемых через переменную `OTEL_EXPORTER_OTLP_PROTOCOL`.

**Атрибуты ресурса**: имя сервиса `claude-code`, несёт архитектуру платформы, версию WSL, тип подписки, версию сервиса и другие атрибуты, автоматически заполняемые через `envDetector`, `hostDetector`, `osDetector`.

### Передача данных через gRPC

gRPC — рекомендуемый протокол передачи для корпоративных сценариев, обеспечивающий двунаправленную потоковую передачу и строго типизированное protobuf-кодирование. В Claude Code:

- Экспортёр gRPC (`@opentelemetry/exporter-metrics-otlp-grpc`) — зависимость с ленивой загрузкой, не влияет на время запуска
- Конфигурация mTLS поддерживается через `getMTLSConfig()`, для корпоративных внутренних сетей можно использовать самоподписанные сертификаты
- Поддержка прокси обрабатывается прозрачно через `getProxyUrl()` + `HttpsProxyAgent`

Дочерние процессы не наследуют переменные окружения, связанные с OTEL (`subprocessEnv.ts`):
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Трассировка Perfetto

Perfetto — высокопроизводительный фреймворк системной трассировки от Google. Claude Code реализует слой совместимости с форматом Chrome Trace Event (`src/utils/telemetry/perfettoTracing.ts`, 1120 строк, только ant).

**Способ включения**:
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # запись в ~/.claude/traces/trace-<session-id>.json
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # запись по указанному пути
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # периодическая запись каждые 60 секунд
```

**Типы трассируемых Span**:

| Имя Span | Категория | Переносимая информация |
|----------|-----------|----------------------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | номер попытки |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**Управление памятью** (лимит 100 000 событий):
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// При достижении лимита удаляется старейшая половина, амортизированная стоимость O(1)
// Вставка маркера trace_truncated делает разрыв видимым в ui.perfetto.dev
```

**Иерархическая трассировка нескольких Agent**: каждый Agent (включая подагенты) сопоставляется с отдельным process ID; иерархические отношения записываются через metadata-события `parent_agent`; в интерфейсе Perfetto отображаются как отдельные дорожки (lanes).

**Стратегия записи** (тройная защита):
1. Асинхронный коллбэк `cleanup registry` (нормальный выход)
2. Обработчик `process.on('beforeExit')` (резерв)
3. Синхронная запись `process.on('exit')` (последний рубеж, async здесь недоступен)

### Трассировка сессий OpenTelemetry

`src/utils/telemetry/sessionTracing.ts` (927 строк) — расширенная точка входа телеметрии для внешних пользователей, основанная на стандартных Span OTel, а не на формате Perfetto.

**Условие включения** (`sessionTracing.ts:170-185`):
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

**Распространение контекста AsyncLocalStorage**: каждый Interaction и Tool Call используют отдельное хранилище ALS для хранения SpanContext, гарантируя отсутствие смешения Span при конкурентных многоагентных сценариях. WeakRef предотвращает утечки памяти долгоживущих Span; с интервалом 60 секунд очищаются «осиротевшие» Span старше 30 минут.

**Система событий logEvent**

Все бизнес-события унифицированно диспетчеризируются через функцию `logEvent()` из `src/services/analytics/index.ts`:

```typescript
// index.ts (упрощено)
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // разрешены только boolean | number | undefined
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

Ключевой дизайн: тип metadata намеренно исключает `string`, заставляя разработчика использовать преобразование типа `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`. На уровне типов это предотвращает случайную запись кода или путей к файлам.

---

## 14.5 Сбор Analytics

### Архитектура двухпутной маршрутизации

Все события маршрутизируются через `sink.ts` в два бэкенда:

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog (если включён гейт tengu_log_datadog_events)
    │     Передаёт только события из белого списка DATADOG_ALLOWED_EVENTS
    │     Удаляет ключи _PROTO_* (поля с маркером PII)
    └─→ 1P First-Party Logger (OpenTelemetry BatchLogRecordProcessor)
          Отправляет на /api/event_logging/batch
          Сохраняет ключи _PROTO_* (маршрутизируются в защищённые столбцы BigQuery)
```

**Интеграция Datadog** (`datadog.ts`):
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Пакетная отправка: 100 записей/пакет, интервал сброса 15 секунд
- Таймаут сети: 5 секунд
- Механизм белого списка: около 50 ключевых событий (Set `DATADOG_ALLOWED_EVENTS`)
- Условия отключения: сторонние облака Bedrock/Vertex/Foundry, тестовая среда, выбор пользователем no-telemetry

**Логирование событий 1P (FirstPartyEventLoggingExporter)**:
- Использует стандартный интерфейс `LogRecordExporter` OpenTelemetry
- Пакетный экспорт: по умолчанию 200 записей/пакет, задержка планирования 5 секунд
- Повторы при сбоях: экспоненциальная выдержка (базовая 500 мс, максимум 30 с, не более 8 попыток)
- Очередь постоянных сбоев: неудачные события записываются в `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl` и повторяются при следующем запуске
- Proto-сериализация: использует сгенерированный тип protobuf `ClaudeCodeInternalEvent`

### Отслеживание действий пользователя

Около 400+ имён событий с префиксом `tengu_*` охватывают полный жизненный цикл взаимодействия пользователя. Ключевые категории событий:

**Жизненный цикл сессии**: `tengu_started`, `tengu_init`, `tengu_exit`, `tengu_cancel`

**API-вызовы**: `tengu_api_query`, `tengu_api_success`, `tengu_api_error`, `tengu_api_retry`

**Использование инструментов**: `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_in_prompt_permanent`

**Запросы прав**: `tengu_internal_bash_tool_use_permission_request`, `tengu_tool_use_show_permission_request`, `tengu_tool_use_granted_by_classifier`

**OAuth-аутентификация**: `tengu_oauth_flow_start`, `tengu_oauth_success`, `tengu_oauth_token_refresh_*` (полная трассировка машины состояний блокировки)

**MCP-серверы**: `tengu_mcp_server_connection_succeeded`, `tengu_mcp_server_connection_failed`, `tengu_mcp_oauth_flow_*`

**Механизм обновлений**: `tengu_binary_download_attempt`, `tengu_native_update_complete`, `tengu_binary_download_failure`

### Сбор показателей производительности

API Call Span в `sessionTracing.ts` вычисляет следующие производные показатели:

```typescript
// perfettoTracing.ts (упрощение endLLMRequestPerfettoSpan)
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second (скорость обработки входных токенов)

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second (скорость выборки)

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // процент попаданий в Cache
```

### Контроль выборки событий

Динамическая конфигурация GrowthBook `tengu_event_sampling_config` управляет частотой выборки каждого события:

```typescript
// firstPartyEventLogger.ts (упрощение shouldSampleEvent)
// null = 100% выборка (нет конфигурации)
// 0 = полное отбрасывание
// rate (0-1) = случайная выборка, sample_rate записывается в metadata
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // пример: 10% выборка
}
```

### Отчётность об ошибках

Многоуровневая система событий ошибок:
- `tengu_uncaught_exception`, `tengu_unhandled_rejection`: необработанные ошибки уровня процесса
- `tengu_api_error`, `tengu_query_error`: ошибки API-вызовов
- `tengu_streaming_error`: ошибки потоковых ответов
- `tengu_atomic_write_error`: ошибки записи файлов
- `tengu_compact_failed`: сбой сжатия сессии

---

## 14.6 Диагностика и отладка

### Команда /doctor

`src/commands/doctor/index.ts` регистрирует команду `/doctor`:

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

Команда выполняется как тип `local-jsx` (напрямую рендерит React-компонент в REPL) и проверяет: целостность установки, статус подключения MCP-серверов, валидность конфигурации привязок клавиш, зависимости окружения (ripgrep и т.д.).

### Система диагностической трассировки

В сценариях интеграции с IDE Claude Code получает диагностическую информацию о коде через Language Server Protocol. После сохранения файла (событие `didSave`) TypeScript Server отправляет новые диагностические сообщения; система вводит их в XML-теге `<new-diagnostics>` для передачи модели:

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### Диагностика памяти кучи

`src/utils/heapDumpService.ts` предоставляет возможность диагностики памяти на уровне процесса: при инициировании дамп кучи синхронно собирает снимок использования памяти и выводит его в `~/Desktop/<session-id>-diagnostics.json`, включая `heapUsed`, `external`, `rss` и рекомендации по анализу. Соответствующее аналитическое событие: `tengu_heap_dump`.

### Логи восстановления после ошибок

`src/utils/telemetry/bigqueryExporter.ts` реализует экспортёр метрик BigQuery, интегрированный с пайплайном OTEL Metrics — для долгосрочного мониторинга производительности и планирования ёмкости внутри ant. Очередь постоянных сбоев `1p_failed_events` гарантирует сохранность критических событий даже при сетевых сбоях.

---

## 14.7 Анализ проектных решений

### Преимущества и недостатки компиляционных флагов

**Преимущества**:
1. **Нулевые накладные расходы во время выполнения**: удалённые ветки кода не существуют в артефакте, никаких условных переходов
2. **Безопасная изоляция**: код функций только для ant полностью невидим для внешних пользователей, реверс-инжиниринг невозможен
3. **Оптимизация размера пакета**: крупные модули (например, `@grpc/grpc-js` ~700 КБ) присутствуют только в нужных сборках
4. **Типобезопасность**: проверка типов TypeScript применяется до упаковки, не влияет на время выполнения

**Недостатки**:
1. **Зависимость от выпуска**: изменение состояния флага требует выпуска новой версии, горячее обновление невозможно
2. **Взрыв матрицы тестирования**: N компиляционных флагов теоретически требуют 2^N комбинаций сборок для тестирования
3. **Сложность отладки**: при поступлении отчёта об ошибке от внешнего пользователя некоторые пути кода просто не существуют в его сборке

### Баланс между конфиденциальностью и наблюдаемостью

Claude Code применяет многоуровневую защиту конфиденциальности:

1. **Защита системой типов**: `LogEventMetadata` разрешает только `boolean | number | undefined`, строки напрямую записывать нельзя. Для записи строки нужно явно объявить `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — это тип `never`, фактически не способный хранить значение; он лишь заставляет разработчика написать аннотацию типа, подтверждая ручную проверку отсутствия кода или путей.

2. **Маскирование имён инструментов MCP**: имена инструментов MCP формата `mcp__<server>__<tool>` могут раскрыть приватные конфигурации сервисов пользователя; по умолчанию маскируются как `mcp_tool`. Оригинальное имя сохраняется только для точки входа `cowork`, серверов из официального реестра MCP или серверов, явно объявленных встроенными.

3. **Поля с маркером PII**: metadata-ключи с префиксом `_PROTO_*` обозначают PII-чувствительные поля, маршрутизируемые только в защищённые столбцы BigQuery 1P; `sink.ts` удаляет эти поля перед пересылкой в Datadog.

4. **Отключение для сторонних облаков**: корпоративные клиенты, использующие Bedrock/Vertex/Foundry, по умолчанию имеют отключённую всю аналитику на стороне Anthropic (включая Datadog и 1P).

### Причины ленивой загрузки Telemetry

Пакеты, связанные с OTLP (gRPC ~700 КБ, proto ~300 КБ), используют динамический `import()` с ленивой загрузкой по следующим причинам:

1. **Критичность времени запуска**: ключевой показатель производительности CLI-инструментов — Time-to-First-Output, любую ненужную инициализацию следует откладывать
2. **Взаимная исключительность протоколов**: процесс использует только один протокол передачи; статический импорт всех вариантов (6 пакетов) — чистое расточительство
3. **Совместимость с оптимизациями Bun**: ленивая загрузка соответствует паттернам разрешения модулей Bun, обеспечивая упаковку по требованию после статического анализа

---

## 14.8 Переносимые паттерны

Следующие проектные паттерны имеют высокую ценность для других проектов:

### 1. Предотвращение утечки PII через систему типов

Через marker type типа `never` принудительно обеспечивается явное подтверждение отсутствия чувствительной информации на этапе компиляции. Нулевые накладные расходы на выполнение, 100% эффективность защиты (обход требует явного приведения типов). Применимо в любой системе с требованиями к отчётности данных.

### 2. Двухуровневая архитектура Feature Flag

Компиляционные (разделение кода) + время выполнения (управление поведением) — двойная схема, соответствующая разным потребностям скорости развёртывания:
- Структурные функции (наличие/отсутствие целых модулей) → компиляционное время
- Настройка поведения (параметры, доли, выбор алгоритма) → время выполнения

### 3. Паттерн Sink Kill-Switch

Конфигурация GrowthBook `tengu_frond_boric` позволяет независимо отключать любой аналитический бэкенд по имени (`datadog`, `firstParty`) без выпуска новой версии. Это универсальный аварийный паттерн прерывателя цепи для любой системы событий с несколькими downstream-приёмниками.

### 4. Персистентные повторы при сбоях событий

При сбое экспорта событий 1P они записываются в локальный JSONL-файл и повторяются при следующем запуске. Это гарантирует сохранность критических данных телеметрии при сетевых сбоях — особенно подходит для инструментов, работающих в оффлайне или при нестабильном подключении.

### 5. Дедупликация показов экспериментов

События показов экспериментов GrowthBook (используемые для анализа результатов A/B-тестов) дедублируются через Set уровня сессии, гарантируя однократную запись показа для каждого feature в аналитической системе и предотвращая завышение счётчика показов из-за многократных вызовов одного флага.

---

## 14.9 Индекс исходного кода

| Путь к файлу (относительно `src/`) | Строк | Основная ответственность |
|-------------------------------------|-------|--------------------------|
| `services/analytics/growthbook.ts` | 1155 | Интеграция SDK GrowthBook, чтение Feature Flag, запись показов A/B |
| `services/analytics/index.ts` | 173 | Публичный API logEvent, очередь событий, определение интерфейса Sink |
| `services/analytics/sink.ts` | 114 | Реализация двухпутной маршрутизации (Datadog + 1P), инициализация |
| `services/analytics/datadog.ts` | 307 | Пакетная отправка логов Datadog, фильтрация белого списка |
| `services/analytics/firstPartyEventLogger.ts` | 449 | Инициализация OTel LoggerProvider, управление выборкой |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | HTTP-экспорт событий 1P, персистентные повторы, proto-сериализация |
| `services/analytics/metadata.ts` | 973 | Обогащение метаданных событий, маскирование имён MCP-инструментов, обработка PII |
| `services/analytics/config.ts` | 38 | Общая логика isAnalyticsDisabled() |
| `services/analytics/sinkKillswitch.ts` | 25 | Kill-Switch уровня Sink (tengu_frond_boric) |
| `utils/telemetry/instrumentation.ts` | 825 | Инициализация OTel SDK, конфигурация трёх сигналов (Metrics/Logs/Traces) |
| `utils/telemetry/sessionTracing.ts` | 927 | Управление OTel Span, распространение контекста AsyncLocalStorage |
| `utils/telemetry/perfettoTracing.ts` | 1120 | Трассировка в формате Chrome Trace для Perfetto (только ant) |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Расширенные атрибуты бета-трассировки |
| `utils/telemetry/bigqueryExporter.ts` | 252 | Экспортёр метрик BigQuery |
| `utils/telemetry/pluginTelemetry.ts` | 289 | Обёртка телеметрии плагинов |
| `utils/telemetry/events.ts` | 75 | Определения типов событий OTel |
| `commands/doctor/index.ts` | 12 | Регистрация команды /doctor |
| `commands.ts` | — | Централизованные вызовы компиляционного feature() |
