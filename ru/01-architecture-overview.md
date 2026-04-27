# Глава 1: Архитектурный обзор и процесс запуска

> Источник данных: снимок TypeScript-исходников Claude Code (31 марта 2026 г.)
> Путь к исходникам (mini): `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 Обзор и позиционирование

**Что такое Claude Code:** Claude Code — это AI-ассистент по программированию, работающий в терминале. Он рендерит интерактивный TUI (Terminal User Interface) с помощью React/Ink и управляет Claude API через цикл REPL для выполнения задач разработки: редактирования кода, выполнения команд, файловых операций и других.

### Обзор технологического стека

| Уровень | Технология | Назначение |
|---------|-----------|-----------|
| Runtime | Bun (основной) / Node.js 18+ (совместимость) | JavaScript-среда выполнения |
| Язык | TypeScript | Строгая типизация всего проекта |
| UI-фреймворк | React + Ink | Рендеринг TUI в терминале |
| CLI-фреймворк | Commander.js (`@commander-js/extra-typings`) | Парсинг аргументов командной строки |
| API-клиент | `@anthropic-ai/sdk` | Вызовы Claude API |
| Интеграция MCP | `@modelcontextprotocol/sdk` | Протокол MCP-сервера |
| Feature flags | GrowthBook + `bun:bundle` feature flags | A/B-тестирование и DCE |
| Телеметрия | OpenTelemetry (ленивая загрузка ~400 КБ) | Метрики/журналы/трейсы |
| Валидация | Zod v4 | Рантайм-валидация схем |

### Статистика объёма кода

- **Всего строк**: 512 664 (файлы `.ts` + `.tsx`)
- **Файлов**: 1 884 TypeScript-файла
- **Каталогов верхнего уровня**: 35

Распределение LOC по основным каталогам:

```
utils/       180 472 строки  (35,2%)  — утилиты, права, аутентификация, настройки и др.
components/   81 546 строки  (15,9%)  — React UI-компоненты
services/     53 680 строки  (10,5%)  — API, MCP, аналитика, память и другие сервисы
tools/        50 828 строки   (9,9%)  — 30 реализаций инструментов (Bash/File/Agent и др.)
commands/     26 428 строки   (5,2%)  — реализации slash-команд
screens/       5 977 строк    (1,2%)  — экраны верхнего уровня (REPL и др.)
bootstrap/    ~5 000 строк    (1,0%)  — глобальное состояние (state.ts — 1 758 строк)
entrypoints/  ~3 000 строки   (0,6%)  — точки входа CLI/SDK/MCP
main.tsx       4 683 строки   (0,9%)  — главный координатор точки входа
setup.ts         477 строк    (0,1%)  — инициализация при старте
```

---

## 1.2 Теоретические основы

### Архитектурные паттерны CLI-приложений

Claude Code объединяет два классических архитектурных паттерна CLI:

**REPL Loop (Read-Eval-Print Loop)**
Традиционный REPL читает ввод, вычисляет и выводит результат в синхронном цикле. Claude Code модернизирует его до асинхронного событийно-ориентированного REPL: ввод захватывается React-компонентами, «вычисление» — это round-trip к Claude API (включая несколько раундов вызовов инструментов), а вывод рендерится в терминал через React/Ink reconciler.

**Event-Driven Architecture**
При запуске не происходит блокирующего ожидания завершения всей инициализации: чтение MDM, предварительная выборка Keychain, подключение MCP, загрузка plugin hook — всё запускается параллельно в режиме fire-and-forget (подробнее см. раздел 1.4). Это минимизирует TTFR (Time To First Render), согласуясь с оптимизацией Critical Rendering Path в веб-приложениях.

### Философия проектирования TUI-фреймворка: React in Terminal

Ink переносит компонентную модель React, декларативное состояние и механизм reconciliation в терминал. Ключевые идеи:

- **Virtual DOM → виртуальный терминальный буфер**: каждое изменение состояния запускает diff, перерисовываются только изменившиеся строки символов — без мерцания
- **Flexbox → разметка терминала**: CSS Yoga вычисляет ширину колонок и перенос строк, позволяя описывать TUI-интерфейс декларативно в JSX
- **Повторное использование компонентов**: спиннер загрузки, диалоги подтверждения, отображение диффов и другая UI-логика оформлены в тестируемые React-компоненты

Это позволяет UI-коду Claude Code разделять когнитивный фреймворк с кодом веб-фронтенда: 81 546 строк в каталоге `components/` можно понимать через привычные React-паттерны.

### Теоретические основы плагинной архитектуры

Плагинная система Claude Code основана на модели «регистрации возможностей» (Capability Registration Pattern):

- Инструменты (Tools), команды (Commands) и хуки (Hooks) регистрируются в глобальном реестре при запуске
- Плагины расширяют списки инструментов/команд через соглашение о файловой системе (`~/.claude/plugins/`)
- Функция `feature()` из `bun:bundle` выполняет Dead Code Elimination (DCE) на этапе компиляции — экспериментальные функции не попадают во внешние артефакты сборки

---

## 1.3 Общая архитектурная схема

### Многоуровневая архитектура (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                 Уровень входных точек (Entry Layer)       │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (CLI)       (маршрутизация Commander.js) (режим MCP-сервера) │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│              Уровень начальной загрузки (Bootstrap Layer) │
│    setup.ts      │    entrypoints/init.ts                 │
│  (инициализация  │    bootstrap/state.ts                 │
│   сессии)        │    (глобальный синглтон состояния)    │
│  (worktree/tmux) │                                       │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  UI-уровень (Ink/React TUI)               │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (основной интерфейс) │  components/ (81К LOC)           │
│  replLauncher.tsx  │  (ввод/вывод/диалоги/анимации)      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Уровень движка (Engine Layer)            │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (управление     │  (API-     │  (дерево React-состояния) │
│   жизн. циклом)  │   вызовы)  │                          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Уровень инструментов (Tool Layer)        │
│  tools/ (30 инструментов, 50К LOC)                       │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Уровень сервисов (Service Layer)         │
│  services/ (53К LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (MCP-клиент)    (GrowthBook/OTel)         │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (языковой    │ (память       │ (корпоративная             │
│   сервер)     │  сессии)      │  конфигурация)            │
└─────────────────────────────────────────────────────────┘
```

### Обзор зависимостей модулей

```
main.tsx
  ├── entrypoints/init.ts       (memoized, инициализируется единожды)
  ├── entrypoints/cli.tsx       (маршрутизация подкоманд Commander)
  ├── bootstrap/state.ts        (глобальное состояние, запрет циклических зависимостей)
  ├── setup.ts                  (вызывается при каждой сессии)
  ├── QueryEngine.ts            (путь headless/SDK)
  ├── replLauncher.tsx          (интерактивный путь)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (загрузка MCP-инструментов/ресурсов)
```

**Особый статус `bootstrap/state.ts`**: в коде есть явный комментарий `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`, а правило ESLint `custom-rules/bootstrap-isolation` запрещает импортировать этот файл в не-листовых модулях, предотвращая циклические зависимости.

### Сравнение трёх точек входа

| Точка входа | Файл | Способ запуска | Особенности |
|-------------|------|----------------|-------------|
| Интерактивный CLI | `entrypoints/cli.tsx` | Команда `claude` | Полный REPL + React TUI |
| SDK (headless) | `QueryEngine.ts` | Флаг `-p` / SDK API | Без UI, разовый или потоковый вывод |
| MCP-сервер | `entrypoints/mcp.ts` | `claude --mcp` | Набор инструментов как MCP-сервер |

---

## 1.4 Детали процесса запуска

### Полная последовательность запуска main.tsx

4 683 строки `main.tsx` не выполняются последовательно — побочные эффекты импортов в начале файла представляют собой тщательно спроектированную параллельную последовательность предварительного прогрева.

**Фаза 0: загрузка модулей (побочные эффекты импортов, ~135 мс)**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. Начальная точка отсчёта производительности

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. Параллельно: дочерний процесс MDM (plutil/reg query)

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. Параллельно: предварительное чтение macOS Keychain (OAuth + API key)

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // Все импорты завершены
```

Комментарии точно указывают выгоду трёх параллельных операций: чтение MDM экономит ~135 мс времени вычисления модулей, предварительное чтение Keychain — ~65 мс последовательного синхронного spawn. Это ключевой приём оптимизации запуска Claude Code: **использование возможности статического анализа ES-модулей для запуска I/O-интенсивных операций в период вычисления графа модулей**.

**Фаза 1: маршрутизация Commander (синхронная)**

В `entrypoints/cli.tsx` Commander.js парсит argv и диспетчеризует к разным путям выполнения по подкоманде (`chat`, `api`, `mcp`, `resume` и др.) или флагу:

```typescript
// entrypoints/cli.tsx (упрощённая структура)
async function main(): Promise<void> {
  // Быстрый путь: --version без дополнительных импортов
  // Стандартный путь: await init() → setup() → ветвление
}
```

**Фаза 2: инициализация init() (memoized, выполняется единожды)**

Функция `init` в `entrypoints/init.ts` обёрнута в `memoize`, гарантируя единственную инициализацию при многократных вызовах:

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // Активация системы конфигурации
  applySafeConfigEnvironmentVariables()  // Безопасные env vars до установления доверия
  applyExtraCACertsFromConfig()     // CA-сертификаты до любых TLS-соединений
  setupGracefulShutdown()           // Регистрация хуков очистки при выходе
  // Ленивая загрузка: OpenTelemetry (~400 КБ) + gRPC (~700 КБ)
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // Асинхронный кэш
  detectCurrentRepository()          // Определение GitHub-репозитория
  preconnectAnthropicApi()           // TCP+TLS предподключение (~100-200 мс перекрытие)
  configureGlobalMTLS()
  configureGlobalAgents()            // Настройка прокси
})
```

**Фаза 3: инициализация сессии setup() (вызывается при каждой сессии)**

```typescript
// setup.ts — ключевая последовательность шагов
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. UDS messaging server (режим swarm/ant)
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Проверка резервного копирования терминала (iTerm2/Terminal.app)
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — должен вызываться раньше всего зависящего от cwd
  setCwd(cwd)
  // 4. Снимок конфигурации хуков (после setCwd)
  captureHooksConfigSnapshot()
  // 5. Создание Worktree (если --worktree)
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. Регистрация фоновых задач (SessionMemory, схлопывание контекста)
  if (!isBareMode()) initSessionMemory()
  // 7. Предварительная загрузка плагинов (параллельно, без блокировки)
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. Активация аналитики + первое событие телеметрии
  initSinks()
  logEvent('tengu_started', {})
  // 9. Проверка примечаний к выпуску (интерактивный режим)
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**Фаза 4: рендеринг REPL**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // Ленивая загрузка UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

В итоге Ink захватывает терминал, дерево React-компонентов начинает рендеринг, REPL готов к работе.

### Стратегия параллельного предварительного получения данных

Оптимизация запуска Claude Code следует принципу «**чем раньше запустить, тем позже ждать**»:

| Операция | Момент запуска | Момент ожидания |
|----------|----------------|-----------------|
| Дочерний процесс MDM (`plutil/reg query`) | Побочный эффект импорта, строка 1 `main.tsx` | До `applySafeConfigEnvironmentVariables()` |
| Предварительное чтение Keychain (OAuth + API key) | Побочный эффект импорта, строка 3 `main.tsx` | `ensureKeychainPrefetchCompleted()` |
| TCP-предподключение к Claude API | `preconnectAnthropicApi()` в `init()` | Первый API-запрос автоматически переиспользует соединение |
| Загрузка plugin hooks | fire-and-forget в `setup()` | `processSessionStartHooks()` перед рендерингом |
| Чтение MCP configs | `getClaudeCodeMcpConfigs()` kick-off | `getMcpToolsCommandsAndResources()` в интерактивном режиме |

### Механизм ленивой загрузки

Claude Code применяет явную ленивую загрузку для крупных модулей на критическом пути запуска:

```typescript
// entrypoints/init.ts — комментарий к ленивой загрузке OpenTelemetry
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

Кроме того, `replLauncher.tsx` импортирует компоненты App и REPL в самый последний момент, предотвращая вычисление дерева React до завершения маршрутизации Commander.

Функция `feature()` из `bun:bundle` реализует DCE на этапе компиляции:

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

Во внешних сборках этот код полностью удаляется, уменьшая размер бандла.

### Детали шагов инициализации setup.ts

477 строк `setup.ts` структурированы вокруг следующих ключевых ограничений:

1. **`setCwd()` должен вызываться первым**: все последующие операции (хуки, настройки, загрузка плагинов) зависят от правильного cwd
2. **Снимок хуков должен сниматься после `setCwd()`**: чтобы читать `.claude/settings.json` из нужного каталога
3. **Создание Worktree должно предшествовать `getCommands()`**: иначе команда `/eject` будет недоступна
4. **`initSinks()` должен вызываться после регистрации всех фоновых задач**: чтобы очередь аналитических событий была готова

В режиме `--bare` (скриптовый/headless SDK-вызов) пропускается большинство интерактивных функций: проверка резервного копирования терминала, предварительная загрузка plugin hooks, attribution коммитов, наблюдатель командной памяти и др. — это минимизирует накладные расходы скриптовых вызовов.

### Построение состояния в bootstrap/state.ts

`state.ts` (1 758 строк) поддерживает глобальный синглтон состояния всей сессии. Основной тип `State` охватывает:

```typescript
// bootstrap/state.ts (частичное определение типа State)
type State = {
  originalCwd: string
  projectRoot: string          // Стабильный корень проекта, worktree его не меняет
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // Счётчики телеметрии
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // Провайдеры логов/трейсов
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... всего ~60 полей
}
```

**Ограничения проектирования**: комментарий `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` — архитектурная защита. Правило ESLint `custom-rules/bootstrap-isolation` предотвращает импорт state.ts модулями, которые могут создать циклические зависимости. Всё состояние доступно через функции-сеттеры/геттеры, изменяемые объекты не выставляются напрямую.

---

## 1.5 Анализ точек входа

### CLI-точка входа (интерактивный режим)

`entrypoints/cli.tsx` — самая сложная точка входа, обрабатывающая всю маршрутизацию пользовательских функций:

**Путь запуска**:
1. Commander.js парсит argv → определяет подкоманду или флаг
2. `await init()` — инициализация (memoized)
3. Обработка MCP configs, корпоративной политики, интеграции Chrome
4. `await setup(cwd, permissionMode, ...)` — инициализация сессии
5. Ветвление по режиму:
   - **Интерактивный режим**: `showSetupScreens()` → `launchRepl()` → React TUI
   - **Режим Print (`-p`)**: `runHeadless()` → `QueryEngine` → stdout
   - **Режим Resume**: `loadConversationForResume()` → восстановление исторической сессии
   - **Режим Teleport**: перехват удалённой сессии

**Ключевые CLI-опции** (частично):

| Флаг | Функция |
|------|---------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | Динамическая конфигурация MCP-сервера |
| `--worktree` | Создание git worktree для изоляции |
| `--tmux` | Запуск в tmux-сессии |
| `--model` | Переопределение модели основного цикла |
| `--resume` | Восстановление исторической сессии |

### SDK-точка входа (программный API)

При вызове через флаг `-p` или программный SDK API React TUI обходится, управление передаётся напрямую в `QueryEngine.ts`:

- `isNonInteractiveSession = true`
- Весь рендеринг UI (Ink) пропускается
- Потоковый вывод в stdout через тип `SDKMessage`
- Поддержка структурированного вывода `SDKStatus`, `SDKPermissionDenial`, `SDKCompactBoundaryMessage` и др.

В SDK-режиме также есть эксклюзивные beta-функции: `entrypoints/sdk/coreSchemas.ts` определяет структурированные JSON input/output-схемы, `entrypoints/agentSdkTypes.ts` — SDK-специфичные типы `HookEvent`, `ModelUsage` и другие.

### MCP-точка входа (режим MCP-сервера)

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools: все инструменты Claude Code выставляются как MCP tools
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool: проксирование выполнения к соответствующей реализации Tool
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

MCP-режим обратно выставляет весь набор инструментов Claude Code (BashTool, FileReadTool, GrepTool и др.) внешним MCP-клиентам, реализуя «Claude Code как MCP-сервер».

### Общая логика трёх точек входа

Независимо от точки входа, все они разделяют:
- Глобальное состояние `bootstrap/state.ts`
- Инициализацию `entrypoints/init.ts` (memoized — выполняется единожды)
- Реестр инструментов `Tool.ts`
- Все сервисы в `services/` (API-клиент, система прав и др.)
- Систему жизненного цикла хуков

Различие — в наличии/отсутствии React TUI и формате вывода (интерактивный текст vs. структурированный JSON).

---

## 1.6 Анализ проектных решений

### Почему Bun, а не Node.js

Из кода можно наблюдать следующие особенности использования Bun:

1. **Функция `feature()` из `bun:bundle`**: уникальный компиляторный механизм feature flags Bun с поддержкой Dead Code Elimination. Широко используется в `main.tsx` (COORDINATOR_MODE, KAIROS, CHICAGO_MCP, UDS_INBOX и др.) — экспериментальный код полностью удаляется во внешних сборках.

2. **WebView API Bun** (условная ссылка): `typeof Bun !== 'undefined' && 'WebView' in Bun` — часть функций зависит от API, специфичного для Bun.

3. **Single-file executable Bun**: в комментариях упоминается `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv` — артефакт выпуска является однофайловым исполняемым файлом, скомпилированным Bun.

4. **Производительность**: Bun значительно быстрее Node.js по скорости запуска и загрузки модулей, что критично для TTFR CLI-инструментов.

Совместимость с Node.js 18+ сохраняется (в `setup.ts` есть проверка версии Node) для поддержки сред без Bun (CI, корпоративные managed-машины).

### Почему React/Ink для TUI

81 546 строк в каталоге `components/` свидетельствуют об исключительно высокой сложности UI. Написанный вручную с помощью сырых ANSI-управляющих кодов, он был бы практически не поддерживаемым. Выбор React/Ink даёт:

1. **Декларативный UI**: потоковый вывод, статусы выполнения инструментов, диалоги подтверждения прав — всё управляется состоянием React, а не императивным управлением курсором
2. **Изоляция компонентов**: `screens/REPL.tsx` заботится только об общей компоновке, а каждая подфункция (поле ввода, список сообщений, прогресс инструментов) инкапсулирована отдельно
3. **Удобство горячей перезагрузки**: при разработке можно применять подход React DevTools
4. **Тестируемость**: React-компоненты можно покрывать юнит-тестами через `@testing-library/react` без реального терминала

### Подход к оптимизации производительности через параллельное предварительное получение данных

Оптимизация запуска Claude Code следует чёткой модели приоритетов: **TTFR (Time To First Render) — в первую очередь, а не «все инициализации завершены»**.

Конкретные проявления:
- Чтение Keychain (~65 мс) запускается при первом побочном эффекте импорта, а не в момент реальной потребности в API key
- Соединение с MCP-серверами происходит в фоне параллельно, не блокируя рендеринг REPL (пользователь видит интерфейс, пока MCP ещё подключается)
- Release notes, конфигурация GrowthBook, plugin hooks — всё в режиме fire-and-forget

Цена — необходимость аккуратно управлять race condition «предвыборка завершена до потребления» через точки ожидания типа `ensureKeychainPrefetchCompleted()`.

### Компромисс ленивой загрузки против предзагрузки

| Стратегия | Объект | Обоснование |
|-----------|--------|-------------|
| Предзагрузка (побочный эффект import) | Дочерний процесс MDM, Keychain | I/O-интенсивные, чем раньше — тем лучше |
| Ленивая загрузка (`await import()`) | OpenTelemetry (~400 КБ), gRPC (~700 КБ), React TUI-компоненты | Дорогостоящее вычисление модуля, не на критическом пути |
| Условная загрузка (DCE через `feature()`) | COORDINATOR_MODE, KAIROS, CHICAGO_MCP | Экспериментальные функции, внешним пользователям не нужны |
| Задержка через `setImmediate()` | Хук attribution коммита | Избежать блокировки event loop в микрозадачном окне setup() |

Эта многоуровневая стратегия позволяет Claude Code выполнять «только минимально необходимую работу для отображения интерфейса» при запуске.

---

## 1.7 Переносимые паттерны

### Универсальный паттерн оптимизации запуска

Последовательность запуска Claude Code демонстрирует трёхуровневый оптимизационный фреймворк «**параллельный прогрев + ленивая загрузка + DCE**»:

**Паттерн 1: I/O-прогрев через побочные эффекты ES-модулей**
```typescript
// Вставка fire-and-forget I/O между операторами import
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // Немедленный запуск без await
import { SomethingElse } from './other.js'  // Параллельная загрузка
```
Применимо для: любых инициализационных данных, которые «нужно прочитать, но чтение медленное» (файлы конфигурации, учётные данные, предварительное сетевое подключение).

**Паттерн 2: однократная инициализация через memoize**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
Применимо для: инициализационной логики, разделяемой несколькими точками входа, предотвращения повторного выполнения.

**Паттерн 3: слои режима `--bare`**
Скриптовые/API-вызовы не требуют UI, проверок терминала, аналитики и пр. — быстрый пропуск через `isBareMode()` поддерживает низкие накладные расходы headless-вызовов.

**Паттерн 4: разделение состояния**
`bootstrap/state.ts` как строгий листовой модуль (без циклических зависимостей), доступный через сеттеры/геттеры, с принудительным соблюдением через правило ESLint. Это позволяет безопасно импортировать модуль состояния из любого места.

### Что можно взять в Doramagic CLI

На основе приведённого анализа CLI Doramagic может применить следующие архитектурные паттерны:

1. **Разделение критического пути запуска**: чётко отделять инициализацию «должна завершиться до рендеринга» от «может завершиться после рендеринга» с пояснительными комментариями (в стиле комментариев Claude Code `// ~65ms on every macOS startup`)

2. **Глобальный синглтон состояния + паттерн аксессоров**: по образцу `bootstrap/state.ts` поддерживать состояние сессии в одном строгом листовом модуле, не давая состоянию рассыпаться по всему коду

3. **Инициализационные функции через `memoize`**: независимо от точки входа, инициализация выполняется единожды

4. **Разделение трёх режимов**: interactive (React TUI) / headless (-p flag) / server (MCP), разделяющих нижние уровни инструментов и сервисов

5. **Feature flags + DCE**: экспериментальные функции обёртываются в feature flags и автоматически удаляются в релизных сборках

---

## 1.8 Индекс исходного кода

| Файл | Строк | Ключевое содержание |
|------|-------|---------------------|
| `main.tsx` | 4 683 | Главная точка входа, маршрутизация Commander, инициализация состояния, ветвление interactive/headless |
| `setup.ts` | 477 | Инициализация сессии: cwd, хуки, worktree, предзагрузка плагинов |
| `bootstrap/state.ts` | 1 758 | Синглтон глобального состояния, определение типа `State`, все геттеры/сеттеры |
| `entrypoints/init.ts` | ~400 | Memoized глобальная инициализация: config, mTLS, proxy, ленивая загрузка OTel |
| `entrypoints/cli.tsx` | ~2 000 | Маршрутизация Commander.js, ветви interactive/print/resume/teleport |
| `entrypoints/mcp.ts` | ~200 | Режим MCP-сервера, выставление набора инструментов |
| `entrypoints/sdk/coreSchemas.ts` | — | Структурированные input/output-схемы SDK-режима |
| `entrypoints/agentSdkTypes.ts` | — | SDK-специфичные типы (HookEvent, ModelUsage и др.) |
| `replLauncher.tsx` | ~30 | Ленивая загрузка App + REPL, запуск React TUI |
| `QueryEngine.ts` | ~1 500 | Управление жизненным циклом сессии, ядро headless-пути |
| `Tool.ts` | — | Определение интерфейса инструмента (inputSchema, call, prompt и др.) |
| `tools/` | 50 828 | 30 реализаций инструментов (BashTool/FileEditTool/AgentTool и др.) |
| `services/api/` | — | Вызовы Claude API, повторы, статистика использования |
| `services/mcp/client.ts` | — | Управление MCP-клиентскими соединениями |
| `utils/startupProfiler.ts` | — | Точки замера производительности `profileCheckpoint()` |
| `utils/secureStorage/keychainPrefetch.ts` | — | Параллельное предварительное чтение macOS Keychain |
| `utils/settings/mdm/rawRead.ts` | — | Параллельное чтение конфигурации MDM |

### Ключевые места в коде

- **Начало параллельного прогрева**: `main.tsx:12-20` (3 побочных эффекта импорта)
- **Memoized инициализация**: `entrypoints/init.ts:57` (`export const init = memoize(...)`)
- **Тип глобального состояния**: `bootstrap/state.ts:30-200` (`type State = {...}`)
- **Определение MCP-сервера**: `entrypoints/mcp.ts:42` (`startMCPServer`)
- **Точка входа рендеринга REPL**: `replLauncher.tsx:14` (`launchRepl`)
- **Интерфейс инструмента**: `Tool.ts:1-30` (`ToolInputJSONSchema`, `ToolUseContext`)
- **Ключевой порядок setup**: `setup.ts:77-230` (setCwd → captureHooksConfigSnapshot → worktree → фоновые задачи)

---

*Количество символов в главе: ~9 800 | Дата снимка исходного кода: 31 марта 2026 г.*
