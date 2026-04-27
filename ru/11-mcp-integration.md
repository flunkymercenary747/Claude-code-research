# Глава 11: Интеграция MCP

## 11.1 Обзор и позиционирование

### Что такое MCP

MCP (Model Context Protocol) — открытый протокол, разработанный под руководством Anthropic, определяющий стандартизированный формат коммуникации между AI-приложениями и внешними сервисами инструментов. По сути, это протокол JSON-RPC 2.0, работающий поверх нескольких транспортных уровней (stdio, SSE, HTTP Streamable, WebSocket) и задающий стандартные форматы сообщений: обнаружение инструментов (`tools/list`), вызов инструментов (`tools/call`), управление ресурсами (`resources/list`/`resources/read`), шаблоны Prompt (`prompts/list`/`prompts/get`).

### Роль MCP в Claude Code

Встроенный набор инструментов Claude Code (Bash, Read, Edit и др.) охватывает файловую систему и локальные сценарии разработки. MCP позиционируется как **открытый интерфейс расширения инструментов**: любой сторонний сервис (Slack, GitHub, Jira, базы данных, браузерная автоматизация и т.д.) может реализовать MCP-сервер, и Claude Code, подключившись по стандартному протоколу, получает доступ к этим внешним возможностям без изменения кода ядра.

С архитектурной точки зрения Claude Code является чистым **MCP-клиентом** и не реализует никаких серверных возможностей MCP (за исключением ответов на запросы `roots/list` для сообщения серверу рабочей директории). Инструменты каждого подключённого MCP-сервера динамически регистрируются как объекты Tool в формате `mcp__<serverName>__<toolName>` и разделяют единый фреймворк исполнения со встроенными инструментами.

### Масштаб кода

Интеграция MCP охватывает около 12 310 строк TypeScript, распределённых по следующим файлам:

| Файл | Строк | Ответственность |
|------|-------|-----------------|
| `services/mcp/client.ts` | 3 348 | Управление соединениями, обнаружение инструментов, ядро исполнения |
| `services/mcp/config.ts` | 1 578 | Управление конфигурацией (слияние из нескольких источников, фильтрация политик) |
| `services/mcp/auth.ts` | 2 465 | Аутентификация OAuth 2.0 (включая XAA — межприложенчный доступ) |
| `services/mcp/utils.ts` | 575 | Фильтрация инструментов, хеширование имён, обнаружение устаревших |
| `services/mcp/types.ts` | 258 | Определения типов (Transport, ServerConfig, состояния соединения) |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | Классификация свёртывания UI (идентификация Search/Read инструментов) |
| `tools/MCPTool/UI.tsx` | 402 | Рендеринг результатов выполнения инструментов |
| `services/mcp/channelPermissions.ts` | 240 | Ретрансляция разрешений Channel |
| `services/mcp/channelNotification.ts` | 316 | Механизм отправки сообщений Channel |
| `services/mcp/elicitationHandler.ts` | 313 | Обработка Elicitation (форм/URL-взаимодействий) |
| `skills/mcpSkillBuilders.ts` | 44 | Реестр строителей Skill (развязка графа зависимостей) |

---

## 11.2 Теоретические основы

### Режим расширения инструментов на основе протокола

Традиционные системы плагинов обычно зависят от SDK, предоставляемого хост-приложением, и разработчики плагинов должны знать внутренние интерфейсы хоста. MCP использует **протокол-ориентированный (protocol-driven)** подход: все взаимодействия между хостом (Claude Code) и плагином (MCP-сервером) осуществляются через стандартные JSON-RPC сообщения, и обе стороны могут развиваться независимо.

Это концептуально совпадает с подходом LSP (Language Server Protocol):

| Измерение | LSP | MCP |
|-----------|-----|-----|
| Основная модель | Редактор ↔ Языковой сервер | AI Agent ↔ Сервер инструментов |
| Механизм обнаружения | Обмен capabilities при `initialize` | `tools/list`, `resources/list`, `prompts/list` |
| Транспортный уровень | stdio, LSP over TCP | stdio, SSE, HTTP Streamable, WebSocket |
| Двусторонняя коммуникация | Поддерживается | Поддерживается (notifications, elicitation) |
| Согласование версий | Поддерживается | Поддерживается (`protocolVersion`) |

LSP решил проблему взрыва M×N «каждый редактор должен поддерживать каждый язык»; MCP решает аналогичную проблему «каждый AI-инструмент должен поддерживать каждый внешний сервис».

### Принципы проектирования клиент-серверного протокола

Два ключевых архитектурных решения MCP оказывают глубокое влияние на реализацию Claude Code:

**Согласование возможностей (Capability Negotiation)**: При подключении сервер через `ServerCapabilities` объявляет поддерживаемое подмножество функций (`tools`, `prompts`, `resources`, `elicitation`, `experimental`), и клиент вызывает только те функции, которые сервер уже объявил. Это означает, что Claude Code не нужно писать специальные ветки для каждого типа серверов — поведение определяется единой проверкой `capabilities`.

**Аннотации инструментов (Tool Annotations)**: В версии MCP 2025-03 добавлено поле `tool.annotations`, позволяющее серверу объявлять семантические маркеры `readOnlyHint`, `destructiveHint`, `openWorldHint` и др. Claude Code напрямую отображает эти маркеры на методы инструмента `isReadOnly()`, `isDestructive()`, `isOpenWorld()`, не требуя ведения статического белого списка имён инструментов для принятия решений о безопасности.

---

## 11.3 Архитектура MCP-клиента

### Основной интерфейс класса MCPClient

Claude Code не реализует MCP-клиент напрямую, а инкапсулирует класс `Client`, предоставляемый `@modelcontextprotocol/sdk`. `connectToServer` является главной точкой входа (`client.ts`) и использует `lodash/memoize` для кеширования на уровне соединения, ключ кеша: `${name}-${jsonStringify(serverRef)}`:

```typescript
// client.ts（约第 540 行）
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... 根据 serverRef.type 初始化 transport
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... 连接、超时、能力协商
  },
  getServerCacheKey,
)
```

### Управление соединениями (установка, поддержание, разрыв)

**Установка соединения**: `connectToServer` создаёт соответствующий transport в зависимости от `serverRef.type`, затем инициирует `client.connect(transport)` с таймаутом 30 секунд (`getConnectionTimeoutMs()`, переопределяется переменной окружения `MCP_TIMEOUT`):

```typescript
// client.ts（约第 1000 行）
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    transport.close().catch(() => {})
    reject(new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
      `MCP server "${name}" connection timed out after ${getConnectionTimeoutMs()}ms`,
      'MCP connection timeout',
    ))
  }, getConnectionTimeoutMs())
  connectPromise.then(() => clearTimeout(timeoutId), () => clearTimeout(timeoutId))
})
await Promise.race([connectPromise, timeoutPromise])
```

**Поддержание соединения**: Обнаружение ошибок и автоматическое переподключение реализованы через переопределение `client.onerror` и `client.onclose`. Для удалённых транспортов (SSE/HTTP) ведётся счётчик `consecutiveConnectionErrors`; после 3 последовательных терминальных ошибок (`ECONNRESET`/`ETIMEDOUT`/`EPIPE` и др.) срабатывает `closeTransportAndRejectPending`, который вызывает `client.close()`, отклоняя все ожидающие `callTool()`, и очищает кеш memoize — следующий запрос автоматически вызовет переподключение:

```typescript
// client.ts（约第 1250 行）
client.onclose = () => {
  // 清除所有相关缓存，下次调用时触发重连
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Обработка истечения сессии**: MCP-серверы на HTTP-транспорте могут возвращать HTTP 404 + код ошибки JSON-RPC `-32001` (Session Not Found). Claude Code обнаруживает этот конкретный шаблон ошибки, инициирует переподключение и прозрачно повторяет попытку в `fetchToolsForClient.call()` (максимум 1 раз):

```typescript
// client.ts（约第 150 行）
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**Разрыв соединения**: Для stdio-транспорта используется трёхэтапная эскалация сигналов: сначала `SIGINT` (ожидание 100 мс), затем `SIGTERM` (ожидание 400 мс), наконец `SIGKILL`. Общий лимит времени разрыва — 600 мс, что предотвращает блокировку завершения CLI.

### Динамическое обнаружение и регистрация инструментов

`fetchToolsForClient` (с LRU-кешем, ёмкость 20) отправляет серверу `tools/list` и оборачивает каждый инструмент в объект, соответствующий внутреннему интерфейсу `Tool`:

- **Правило именования**: `mcp__${normalizeNameForMCP(serverName)}__${toolName}` (формат соединения через подчёркивание)
- **Усечение описания**: описания длиннее `MAX_MCP_DESCRIPTION_LENGTH = 2048` символов усекаются с добавлением `… [truncated]`, предотвращая засорение контекста сверхдлинной документацией серверов, генерирующих OpenAPI
- **Отображение разрешений**: `tool.annotations.readOnlyHint` → `isReadOnly()`, `tool.annotations.destructiveHint` → `isDestructive()`
- **Классификация свёртывания**: вызов `classifyMcpToolForCollapse(serverName, toolName)` определяет, является ли инструмент Search/Read типа

Аналогично, `fetchCommandsForClient` отправляет `prompts/list` и преобразует MCP Prompt в объекты `/команд`; `fetchResourcesForClient` отправляет `resources/list` и для серверов, поддерживающих ресурсы, инжектирует инструменты `ListMcpResourcesTool` и `ReadMcpResourceTool`.

### Транспортный уровень передачи сообщений

Claude Code поддерживает 6 типов транспорта:

| Тип | Сценарий применения | Класс Transport |
|-----|---------------------|----------------|
| `stdio` | Локальный подпроцесс (большинство серверов сообщества) | `StdioClientTransport` |
| `sse` | Удалённый SSE-сервер (с OAuth) | `SSEClientTransport` |
| `sse-ide` | Внутренний SSE расширения IDE (без OAuth) | `SSEClientTransport` (упрощённая конфигурация) |
| `http` | MCP Streamable HTTP (новейшая спецификация) | `StreamableHTTPClientTransport` |
| `ws` | WebSocket транспорт | Пользовательский `WebSocketTransport` |
| `ws-ide` | Внутренний WebSocket расширения IDE | `WebSocketTransport` (с `X-Claude-Code-Ide-Authorization`) |

В специальных сценариях MCP-серверы Chrome Extension и Computer Use работают во **внутрипроцессном режиме (In-Process)**: через `createLinkedTransportPair()` создаётся канал в памяти, избегая ~325 МБ накладных расходов подпроцесса.

Важная инженерная деталь HTTP-транспорта: каждый POST-запрос должен содержать заголовок `Accept: application/json, text/event-stream` (требование спецификации MCP Streamable HTTP). Claude Code единообразно инжектирует этот заголовок через `wrapFetchWithTimeout`, предотвращая его потерю в некоторых средах выполнения:

```typescript
// client.ts（约第 460 行）
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// wrapFetchWithTimeout 中：
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 Управление конфигурацией MCP

### Формат конфигурации сервера

`types.ts` использует Zod для определения 7 схем конфигурации серверов, агрегированных через `z.union([...])` в `McpServerConfigSchema`:

```typescript
// types.ts（第 28-115 行，概要）
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // 向后兼容：无 type 字段 = stdio
  command: z.string().min(1),
  args: z.array(z.string()).default([]),
  env: z.record(z.string(), z.string()).optional(),
}))

export const McpSSEServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('sse'),
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: McpOAuthConfigSchema().optional(),
}))
// + HTTP / WebSocket / SDK / claudeai-proxy ...
```

`ScopedMcpServerConfig` добавляет к базовой конфигурации поля `scope` (источник конфигурации) и `pluginSource` (идентификатор источника, предоставляющего плагин) для проверки разрешений Channel.

### Слияние конфигураций из нескольких источников (enterprise > local > project > user > dynamic)

`getClaudeCodeMcpConfigs` (`config.ts`) реализует многоуровневое слияние конфигураций, приоритет от высшего к низшему:

1. **enterprise** (`managed-mcp.json`): монопольный режим предприятия — при наличии этого файла все остальные источники блокируются
2. **local** (приватный для проекта, хранится в глобальной конфигурации пользователя, привязан к CWD)
3. **project** (`.mcp.json`, обход директорий вверх, ближайший имеет приоритет)
4. **user** (поле `mcpServers` в глобальном `~/.claude/config.json`)
5. **dynamic** (инъекция в рантайме через параметр CLI `--mcp-config`)

Конфигурация project требует дополнительного **шлюза пользовательского подтверждения**: при первом обнаружении сервера в `.mcp.json` отображается диалог подтверждения. `getProjectMcpServerStatus()` читает настройки `enabledMcpjsonServers`/`disabledMcpjsonServers` и возвращает `approved`/`rejected`/`pending`. В неинтерактивном режиме (параметр `-p`, вызов через SDK) при включённом `isSettingSourceEnabled('projectSettings')` подтверждение происходит автоматически.

После слияния конфигураций выполняется также **дедупликация**: Plugin-серверы дедуплицируются по «сигнатуре» (для stdio-серверов — массив команд, для удалённых — URL), предотвращая двойное подключение к одному серверу; claude.ai Connector аналогично дедуплицируется, чтобы не дублировать ручную конфигурацию.

### Расширение переменных окружения

В файлах конфигурации можно использовать синтаксис `${ENV_VAR}`, который `expandEnvVarsInString` (`config.ts`/`envExpansion.ts`) раскрывает при чтении конфигурации. Неопределённые переменные собираются в список `missingVars` и сообщаются пользователю.

---

## 11.5 Система аутентификации MCP

### Интеграция OAuth 2.0

`ClaudeAuthProvider` (`auth.ts`) реализует интерфейс `OAuthClientProvider` MCP SDK и управляет всем жизненным циклом OAuth. Процесс аутентификации следует RFC 6749 (поток Authorization Code) + PKCE (Proof Key for Code Exchange) с приёмом callback через локальный HTTP-сервер:

1. **Обнаружение метаданных**: сначала исследуется RFC 9728 (`/.well-known/oauth-protected-resource`), при неудаче — откат к RFC 8414 (`/.well-known/oauth-authorization-server`), затем попытка пути-осведомлённого обнаружения (для обратной совместимости)
2. **DCR (динамическая регистрация клиента)**: при первой аутентификации автоматически регистрируется OAuth-клиент, `clientId`/`clientSecret` сохраняются в системном Keychain
3. **Обмен токенами**: локальный случайный порт принимает код авторизации, обменивает на access_token + refresh_token
4. **Обновление токенов**: `checkAndRefreshOAuthTokenIfNeeded()` проверяет истечение срока перед вызовом и при необходимости обновляет, при неудаче выполняет умную повторную попытку

**Уровень совместимости Slack**: Некоторые OAuth-серверы (в частности, Slack) возвращают HTTP 200 с телом ошибки на endpoint токена, нарушая ожидания RFC 6749. Claude Code переписывает такие ответы в HTTP 400 через `normalizeOAuthErrorBody`, позволяя логике классификации ошибок SDK работать корректно:

```typescript
// auth.ts（约第 250 行）
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // 检测是否是 OAuthErrorResponse 伪装成 200
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // 将 Slack 非标准错误码 'invalid_refresh_token' 标准化为 'invalid_grant'
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### Поддержка нескольких методов аутентификации

Помимо стандартного OAuth, Claude Code поддерживает:

- **Step-Up Auth**: Некоторые операции требуют повышенного охвата разрешений — сервер возвращает HTTP 403 с указанием новых требований к scope, Claude Code обнаруживает это и повторно инициирует поток OAuth
- **XAA (Cross-App Access / SEP-990)**: В корпоративных сценариях через единый IdP (поддерживающий OIDC) одна авторизация предоставляет доступ к нескольким MCP-серверам; используется составной поток RFC 8693 (Token Exchange) + RFC 7523 (JWT Bearer), устраняя необходимость открывать браузер для каждого сервера отдельно
- **Static Headers**: Инъекция статических заголовков аутентификации через конфигурационный файл или скрипт `headersHelper` (подходит для аутентификации по API Key)

### Управление токенами

Структуры данных токенов хранятся в системном защищённом хранилище (macOS Keychain / Linux Secret Service) с ключом `${serverName}|${SHA256(config)[:16]}`, гарантируя, что серверы с одинаковым именем, но разной конфигурацией используют отдельные слоты токенов.

`auth-cache` (`mcp-needs-auth-cache.json`) записывает серверы, недавно вернувшие 401, с TTL 15 минут, предотвращая повторное зондирование гарантированно неудачных серверов при каждом запуске. Чтение кеша осуществляется через общий Promise (`authCachePromise`), предотвращая N параллельных чтений одного файла при пакетном подключении.

---

## 11.6 Выполнение инструментов MCP

### Поток выполнения MCPTool

Когда LLM решает вызвать `mcp__slack__send_message`, поток выполнения следующий:

1. **Маршрутизация**: вызывается функция `call()`, зарегистрированная `fetchToolsForClient`, с входными данными JSON, сгенерированными LLM
2. **Проверка переподключения**: `ensureConnectedClient(client)` проверяет, действительно ли соединение, при необходимости переподключается
3. **Уведомление о прогрессе**: через callback `onProgress` испускается событие `mcp_progress: started`
4. **Вызов инструмента**: `callMCPToolWithUrlElicitationRetry` (обёртка над `callMCPTool`) отправляет запрос `tools/call` серверу
5. **Обработка результата**: изображения и крупный бинарный контент обрабатываются особо (сохраняются на диск, передаётся ссылка); слишком большой текстовый контент усекается
6. **Уведомление о прогрессе**: испускается событие `mcp_progress: completed` (с временем выполнения)

Прозрачная логика повтора при истечении сессии:

```typescript
// client.ts（约第 2100 行）
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // 自动重试一次
    }
    throw error
  }
}
```

### classifyForCollapse — классификация свёртывания результатов инструментов в контексте

`classifyForCollapse.ts` поддерживает два статических Set: `SEARCH_TOOLS` (около 100 имён инструментов) и `READ_TOOLS` (около 300 имён инструментов), охватывающих 40+ популярных MCP-серверов (Slack, GitHub, Linear, Datadog, Sentry, Jira, Asana, Gmail, Grafana, PagerDuty и др.).

Правила классификации: имя инструмента сначала проходит через `normalize()` (унифицированное преобразование camelCase/kebab-case в snake_case), затем проверяется принадлежность к двум Set:

```typescript
// classifyForCollapse.ts（第 587-598 行）
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')
    .replace(/-/g, '_')
    .toLowerCase()
}

export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean } {
  const normalized = normalize(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
  }
}
```

**Замысел проектирования**: Результаты инструментов типа Search/Read, как правило, велики, но их ценность для последующих рассуждений LLM ограничена (промежуточное состояние поиска). После маркировки уровень UI может свернуть эти результаты в истории диалога, экономя видимое пространство и context window. Обратите внимание, что классификация **консервативна** (неизвестные инструменты не сворачиваются) и **основана только на имени инструмента**, без различия по имени сервера — поскольку имена инструментов популярных серверов являются стабильными кросс-инстансными идентификаторами.

### Управление разрешениями и изолированная среда

Перед выполнением инструмента MCP вызывается `checkPermissions()`, который возвращает статус `passthrough` (то есть всегда требует отображения запроса разрешений), включая быстрое действие с предложением добавить имя инструмента в список правил `allow`.

Таймаут вызова инструмента управляется переменной окружения `MCP_TOOL_TIMEOUT`, по умолчанию `100_000_000` миллисекунд (около 27,8 часов, фактически «бесконечно»), позволяя MCP-серверам с длительными операциями завершаться нормально.

---

## 11.7 Система Channel в MCP

Система Channel является расширенным применением MCP: она позволяет внешним мессенджерам (Telegram, Discord, iMessage, Slack и др.) отправлять сообщения в текущую сессию Claude Code (feature flag: `KAIROS`/`KAIROS_CHANNELS`, runtime gate: `tengu_harbor`).

### Управление разрешениями Channel

`channelPermissions.ts` реализует механизм **делегирования разрешений**: когда Claude Code сталкивается с операцией, требующей подтверждения пользователя, он может одновременно через Channel-сервер отправить уведомление на телефон пользователя; пользователь отвечает `yes <5-буквенный_ID>`, сервер разбирает ответ и уведомляет Claude Code о подтверждении через событие `notifications/claude/channel/permission`.

5-буквенный ID использует алфавит из 25 символов (буква `l` исключена во избежание путаницы с `1`/`I`), генерируется через FNV-1a хеш с фильтрацией нецензурных слов (список `ID_AVOID_SUBSTRINGS`, около 24 слов), гарантируя отсутствие нежелательного контента в рабочих сообщениях:

```typescript
// channelPermissions.ts（第 86-110 行）
export function shortRequestId(toolUseID: string): string {
  let candidate = hashToId(toolUseID)
  for (let salt = 0; salt < 10; salt++) {
    if (!ID_AVOID_SUBSTRINGS.some(bad => candidate.includes(bad))) {
      return candidate
    }
    candidate = hashToId(`${toolUseID}:${salt}`)
  }
  return candidate
}
```

Channel-сервер должен одновременно объявить обе возможности `capabilities.experimental['claude/channel']` и `capabilities.experimental['claude/channel/permission']`, чтобы стать ретранслятором разрешений, предотвращая случайное открытие границы безопасности.

### Механизм уведомлений Channel

`channelNotification.ts` определяет полную логику шлюза входящих сообщений (`gateChannelServer`), последовательно проверяя:

1. Объявление возможностей сервера (`claude/channel`)
2. Runtime-переключатель (`tengu_harbor`)
3. OAuth-аутентификация (поддерживается только авторизация через аккаунт claude.ai, не API Key)
4. Командная/корпоративная политика (`channelsEnabled: true`)
5. Параметр сессии `--channels` (явно объявленные пользователем доверенные Channel)
6. Проверка источника Marketplace (предотвращение имитации `slack@anthropic` через `slack@evil`)

Входящие сообщения оборачиваются в формат `<channel source="serverName" meta_key="value">content</channel>` и инжектируются в очередь сессии; при пробуждении после опроса `SleepTool` (интервал около 1 секунды) модель может решить, как ответить.

### Обработка Elicitation

`elicitationHandler.ts` обрабатывает интерактивные запросы, инициируемые сервером (спецификация MCP Elicitation). Поддерживается два режима:

- **Режим form**: сервер запрашивает заполнение формы пользователем (поле `requestedSchema` определяет JSON Schema)
- **Режим url**: сервер запрашивает переход пользователя по URL для выполнения операции (например, авторизации OAuth)

Поток обработки: сначала запускается система Hook (программный ответ); если Hook не отвечает, запрос ставится в очередь `AppState.elicitation.queue`, ожидая рендеринга формы UI или открытия браузера; после действия пользователя callback `respond()` инициирует ответ:

```typescript
// elicitationHandler.ts（第 69-90 行）
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. 先尝试 Hook（程序化响应）
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. 显示 UI，等待用户响应
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

Режим URL также поддерживает `ElicitationCompleteNotificationSchema`: после завершения операции сервер активно уведомляет Claude Code, соответствующий элемент очереди помечается `completed: true`, и UI обновляет отображаемый статус.

---

## 11.8 Построение Skill в MCP

`skills/mcpSkillBuilders.ts` — это минималистичный модуль развязки графа зависимостей (44 строки), решающий проблему циклических зависимостей:

```
client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts  (循环)
```

Решение — введение реестра с однократной записью (Write-Once Registry):

```typescript
// mcpSkillBuilders.ts（全文）
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) throw new Error('MCP skill builders not registered')
  return builders
}
```

Модуль `loadSkillsDir.ts` при инициализации вызывает `registerMCPSkillBuilders()` — это происходит при запуске приложения (через статический import `commands.ts`) заблаговременно, гарантируя готовность реестра к моменту подключения MCP-сервера.

Механизм обнаружения Skill (Feature Flag: `MCP_SKILLS`): `fetchMcpSkillsForClient` читает ресурсы `skill://` сервера, парсит файлы skill в формате Markdown (с метаданными frontmatter) в Skill Command и регистрирует их как slash-команды в формате `/serverName:skillName`, позволяя MCP-серверам автоматически предоставлять повторно используемые AI-рабочие процессы.

---

## 11.9 Анализ архитектурных решений

### Почему MCP, а не кастомный протокол плагинов

**Проблема**: Claude Code требует поддержки большого количества сторонних интеграций инструментов; кастомный Plugin API создаёт экосистемную привязку.

**Решение**: Принятие открытого стандарта MCP с участием в экосистеме сообщества MCP.

**Обоснование**:
- В начале 2025 года экосистема MCP насчитывала сотни готовых к использованию серверов с открытым исходным кодом
- Разработчики плагинов могут реализовывать MCP-серверы на любом языке (Python, Go, Java и др.), не ограничиваясь экосистемой TypeScript
- Anthropic является одновременно разработчиком Claude Code и ведущим автором спецификации MCP, что позволяет решать потребности на уровне спецификации, а не патчить клиент

**Цена**: Вся сложность протокола (аутентификация, различия транспортных уровней, совместимость версий) ложится на клиента; 2 465 строк `auth.ts` во многом являются результатом обработки дефектов спецификации OAuth и несовместимых реализаций разных вендоров.

### Управление сложностью аутентификации MCP

**Проблема**: Описание аутентификации в спецификации MCP носит «рекомендательный» характер, и реализации серверов кардинально различаются (Slack возвращает ошибки с кодом 200, некоторые серверы не реализуют отзыв токенов и т.д.).

**Решение**: Создание полного уровня совместимости в `auth.ts` для обработки известных несовместимых поведений.

Ключевые стратегии:
- `normalizeOAuthErrorBody`: обработка ответов с ошибкой, замаскированной под 200
- `NONSTANDARD_INVALID_GRANT_ALIASES`: нормализация нестандартных кодов ошибок Slack и аналогичных
- Двойная попытка RFC 7009 revocation (сначала стандартным способом, при получении 401 — повтор с Bearer)
- Два пути обнаружения auth (RFC 9728 → RFC 8414 → откат с осведомлённостью о пути)

### Соображения проектирования классификации свёртывания инструментов

**Проблема**: Результаты инструментов MCP могут быть чрезвычайно длинными (результаты поиска, вывод логов), но массовое включение в историю диалога снижает читаемость и расходует context window.

**Решение**: Использование явного белого списка имён инструментов вместо эвристической классификации; результаты известных Search/Read инструментов сворачиваются в UI по умолчанию.

**Компромиссы**:
- Плюс: высокая детерминированность, нет ложных срабатываний, согласованное поведение для известных инструментов
- Минус: необходимость поддержки статического списка (текущий `classifyForCollapse.ts` охватывает 40+ серверов), новые серверы требуют ручного обновления
- Консервативная стратегия (неизвестные инструменты не сворачиваются) гарантирует, что новые серверы не потеряют информацию из-за неправильного свёртывания

---

## 11.10 Переносимые паттерны

Следующие паттерны взяты из инженерной практики интеграции MCP в Claude Code и применимы к другим системам, требующим интеграции внешних инструментов или сервисов:

**1. Согласование возможностей в приоритете перед проверкой типа**
При реализации клиента протокола всегда определяйте поведение через проверку `capabilities`, а не через ветвление if-else по типу или имени сервера. Это позволяет добавлять поддержку новых возможностей инкрементально, не затрагивая существующую логику.

**2. Паттерн Memoize + Cache Invalidation**
Дорогостоящие операции (подключение, обнаружение инструментов и др.) кешируются через memoize, но кеш должен немедленно инвалидироваться при разрыве соединения (очистка всех связанных записей кеша в `client.onclose`). Используйте LRU-кеш (ёмкость 20) для предотвращения утечек памяти.

**3. Реестр с однократной записью для решения циклических зависимостей**
Когда модуль A должен зависеть от функции модуля B, а модуль B косвенно зависит от модуля A, введите модуль реестра с нулевыми внешними зависимостями; при инициализации приложения модуль B инжектирует реализацию в реестр, модуль A читает из реестра. `mcpSkillBuilders.ts` — минимально воспроизводимый шаблон.

**4. Централизованное управление уровнем совместимости протоколов**
Несовместимые реализации спецификаций OAuth/HTTP должны обрабатываться централизованно в одном месте (например, `auth.ts`), а не рассыпаться по местам вызова. `normalizeOAuthErrorBody` — типичный пример этого паттерна: чистая функция, унифицированно обрабатывающая на транспортном уровне, не требуя от вызывающего кода знать, является ли сервер совместимым.

**5. Многоуровневое ограничение скорости параллельных соединений**
Разные типы операций потребляют ресурсы по-разному: stdio-серверы требуют fork подпроцесса (интенсивно по CPU + памяти), сетевые серверы требуют только TCP-соединения (интенсивно по I/O). Использование разных ограничений параллелизма для двух типов операций (local: 3, remote: 20) позволяет максимизировать пропускную способность при защите системы.

**6. Двойная проверка состояния needs-auth**
Для удалённых сервисов, требующих аутентификации, комбинируйте **кеш на основе времени** (TTL 15 минут) и **проверку на основе состояния** (есть состояние discovery, но нет токена) для двойного определения и пропуска заведомо неудачных подключений, избегая задержек бесполезного зондирования при каждом запуске.

---

## 11.11 Индекс исходного кода

| Ключевая реализация | Файл:строка |
|--------------------|-------------|
| Определение объединённого типа `MCPServerConnection` | `services/mcp/types.ts:170-200` |
| Перечисление `ConfigScope` (7 источников) | `services/mcp/types.ts:10-22` |
| Главная функция `connectToServer` (memoized) | `services/mcp/client.ts:540` |
| Ветви инициализации Transport (6 типов) | `services/mcp/client.ts:570-930` |
| Обработка таймаута соединения | `services/mcp/client.ts:1000-1040` |
| Обнаружение ошибок разрыва и инициирование переподключения | `services/mcp/client.ts:1200-1320` |
| Трёхэтапное закрытие stdio (SIGINT/SIGTERM/SIGKILL) | `services/mcp/client.ts:1370-1490` |
| `fetchToolsForClient` (регистрация инструментов) | `services/mcp/client.ts:1830-2050` |
| `getMcpToolsCommandsAndResources` (точка входа пакетного подключения) | `services/mcp/client.ts:2580` |
| `isMcpSessionExpiredError` | `services/mcp/client.ts:150-165` |
| `wrapFetchWithTimeout` (инъекция HTTP Accept) | `services/mcp/client.ts:450-510` |
| Слияние источников конфигурации `getClaudeCodeMcpConfigs` | `services/mcp/config.ts:1050` |
| Определение монопольного режима предприятия | `services/mcp/config.ts:1080-1090` |
| Шлюз подтверждения Project-сервера | `services/mcp/utils.ts:210-250` |
| Дедупликация Plugin-серверов (хеш сигнатуры) | `services/mcp/config.ts:215-270` |
| `ClaudeAuthProvider` (ядро OAuth) | `services/mcp/auth.ts:500+` |
| `normalizeOAuthErrorBody` (совместимость Slack) | `services/mcp/auth.ts:250-290` |
| `performMCPXaaAuth` (межприложенчная аутентификация) | `services/mcp/auth.ts:700+` |
| `getServerKey` (генерация ключа хранения токена) | `services/mcp/auth.ts:390-405` |
| `hasMcpDiscoveryButNoToken` (быстрый отказ) | `services/mcp/auth.ts:420-435` |
| `classifyMcpToolForCollapse` (классификация свёртывания) | `tools/MCPTool/classifyForCollapse.ts:587-598` |
| Белые списки `SEARCH_TOOLS` / `READ_TOOLS` | `tools/MCPTool/classifyForCollapse.ts:20-585` |
| `shortRequestId` (ID разрешения Channel) | `services/mcp/channelPermissions.ts:140-160` |
| `gateChannelServer` (6-уровневый шлюз) | `services/mcp/channelNotification.ts:190-310` |
| `registerElicitationHandler` | `services/mcp/elicitationHandler.ts:65-150` |
| `registerMCPSkillBuilders` (развязка циклических зависимостей) | `skills/mcpSkillBuilders.ts:30-44` |
