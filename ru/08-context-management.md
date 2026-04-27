# Глава 8: Управление контекстом

## 8.1 Обзор и позиционирование

Управление контекстом — одна из наиболее критичных подсистем в архитектуре Claude Code. Типичная сессия программирования может длиться несколько часов, задействовать сотни вызовов инструментов и порождать сотни тысяч токенов в истории диалога. Без управления контекстное окно истощается через 20–30 итераций, прерывая сессию.

Ключевая проблема, которую решает система управления контекстом Claude Code: **как в рамках ограниченного контекстного окна (обычно 200 тыс. токенов) поддерживать непрерывность сессии и целостность информации, одновременно минимизируя воспринимаемые пользователем потери информации?**

Система состоит из 11 файлов в директории `services/compact/`, суммарно около 3 900 строк TypeScript-кода, а также двух ключевых вспомогательных модулей: `utils/collapseReadSearch.ts` (1 109 строк) и `utils/toolResultStorage.ts` (1 040 строк). Архитектура всей подсистемы отражает три основных принципа:

1. **Постепенная деградация** (Graceful Degradation): от нулевой стоимости микросжатия до разрушительного полного сжатия — с постепенным усилением вмешательства
2. **Кеш в приоритете** (Cache-First): каждое решение о сжатии прежде всего учитывает сохранность prompt cache
3. **Гарантии безопасности** (Safety Invariants): пары tool_use/tool_result нельзя разрывать, защита от рекурсии, механизм автоматического выключателя (circuit breaker)

## 8.2 Теоретические основы

### 8.2.1 Информационно-теоретическая перспектива: сжатие с потерями и без потерь

Управление контекстом по существу является **задачей сжатия информации**. Многоуровневая система Claude Code соответствует различным стратегиям сжатия:

- **Без потерь** (Lossless): путь `cache_edits` микросжатия — через механизм cache editing API удаляет серверные копии кешированных результатов старых инструментов без изменения локального содержимого сообщений. Модель видит заполнитель `[Old tool result content cleared]`, но исходные данные сохраняются на диске (`toolResultStorage.ts`). Информация не теряется, лишь перемещается из горячего хранилища в холодное.
- **С потерями** (Lossy): полное сжатие через Fork Agent генерирует резюме, сжимая десятки тысяч токенов диалога до нескольких тысяч. Это необратимый процесс снижения размерности — детали кода, стек ошибок, промежуточные рассуждения могут быть утрачены.

С позиции Rate-Distortion Theory архитектура Claude Code имплицитно задаёт **функцию измерения искажений**: 9 разделов промпта для резюме (см. раздел 8.6) определяют, какие информационные измерения наименее терпимы к потерям — «user messages» (полное сохранение) приоритетнее, чем «key technical concepts» (допустимо обобщение).

### 8.2.2 Теория кеша: временная и пространственная локальность

Механизм белого списка в микросжатии воплощает предположение **временной локальности** (Temporal Locality) из классической теории кеша:

> Недавно использованные результаты инструментов с большей вероятностью будут нужны в будущем.

Белый список (`COMPACTABLE_TOOLS`) в `microCompact.ts` — это воплощение политики вытеснения (eviction policy): только результаты определённых инструментов (Read, Shell, Grep, Glob, WebFetch, WebSearch, Edit, Write) могут быть очищены, поскольку их вывод воспроизводим (инструмент можно выполнить повторно). Пользовательский ввод, изображения и другие невоспроизводимые данные никогда не очищаются.

```typescript
// microCompact.ts:30-41 — 可压缩工具白名单
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

Параметр `keepRecent` (по умолчанию сохраняет 5 последних) напрямую реализует политику вытеснения LRU (Least Recently Used).

### 8.2.3 Паттерн автоматического выключателя (Circuit Breaker Pattern)

Механизм circuit breaker в `autoCompact.ts` — это точная адаптация классического паттерна Circuit Breaker из распределённых систем (введённого Майклом Найгардом в книге «Release It!») для LLM-приложений. Трёхсостояниевая модель (Closed → Open → Half-Open) в реализации Claude Code:

```typescript
// autoCompact.ts:70-73 — 断路器阈值
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

Этот комментарий раскрывает реальные катастрофические данные до введения circuit breaker: **1 279 сессий застряли в петле из 50+ последовательных сбоев**, в одной самой тяжёлой сессии было 3 272 неудачных попытки, а в глобальном масштабе тратилось около 250 тыс. API-вызовов в день. Введение circuit breaker ограничивает максимальное число повторных попыток тремя.

| Состояние | Поведение | Соответствующий код |
|-----------|-----------|---------------------|
| Closed (нормальное) | `consecutiveFailures < 3`, обычная попытка сжатия | Путь по умолчанию `autoCompactIfNeeded` |
| Open (сработавший) | `consecutiveFailures >= 3`, пропуск сжатия | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open (зондирование) | После успешного сжатия `consecutiveFailures` сбрасывается в 0 | `consecutiveFailures: 0` при успехе |

## 8.3 Общая архитектура

### 8.3.1 Общая архитектура многоуровневой системы сжатия

Система управления контекстом Claude Code использует **5-уровневую защиту**. В порядке от минимального вмешательства к максимальному:

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户请求                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage（预防层）                           │
│   大工具结果 → 磁盘持久化 + 2KB 预览                              │
│   触发: 结果 > 阈值（默认 50K chars）                             │
│   代价: 零上下文代价（存磁盘，preview 在上下文）                    │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact（微压缩）                                  │
│   路径 A: 时间触发 — 清除旧工具结果内容                            │
│   路径 B: 缓存编辑 — cache_edits API 删除服务端缓存               │
│   触发: 每次 API 调用前                                          │
│   代价: 极低（工具结果被占位符替换，可通过磁盘恢复）                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact（自动压缩）                                │
│   Session Memory → Fork Agent → 全量摘要                         │
│   触发: token 超过 effectiveContextWindow - 13K                  │
│   代价: 高（有损摘要，丢失细节，消耗一次 API 调用）                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact（手动压缩）                              │
│   用户主动触发，支持 Partial Compact                               │
│   触发: 用户命令                                                  │
│   代价: 同上                                                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact（响应式压缩）                           │
│   API 返回 prompt_too_long → 截断重试                             │
│   触发: 413 错误                                                 │
│   代价: 最高（紧急截断 + 摘要，信息损失最大）                        │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 Сравнение условий срабатывания, стоимости и потерь информации по уровням

| Уровень | Условие срабатывания | Момент | Задержка | Потери информации | Стоимость API |
|---------|---------------------|--------|----------|-------------------|---------------|
| L0: Tool Result Storage | Единственный результат инструмента > порога | После выполнения инструмента | Дисковый I/O | Ноль (оригинал на диске) | Ноль |
| L1a: Time-based MC | С момента последнего assistant прошло > 60 мин | Перед API-вызовом | Ноль (локальная операция) | Низкие (очистка старых результатов) | Ноль |
| L1b: Cached MC | Число сжимаемых инструментов превышает порог | Перед API-вызовом | Ноль (cache_edits) | Низкие (аналогично) | Ноль |
| L2: Auto-Compact | токены > порога | Между итерациями | 5–15 с (API-вызов) | Высокие (сжатие с потерями) | 1 API-вызов |
| L3: Manual Compact | Пользователь вводит /compact | По команде пользователя | Аналогично | Средние–высокие (пользователь может направлять) | 1 API-вызов |
| L4: Reactive Compact | prompt_too_long 413 | После сбоя API | 10–30 с (повтор) | Наивысшие (усечение + резюме) | 1–4 API-вызова |

### 8.3.3 Поток данных

```
消息数组 (Message[])
    │
    ▼
microcompactMessages()  ──→ [时间触发?] ──Y──→ 内容清除 → 返回
    │ N                      │
    │                  [缓存编辑?] ──Y──→ pendingCacheEdits → 返回
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     不做压缩，直接返回
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [有 session memory?]
    │ N                              │ Y
    ▼                                ▼
compactConversation()           calculateMessagesToKeepIndex()
    │                                │
    ▼                                ▼
streamCompactSummary()          buildPostCompactMessages()
    │ (Fork Agent)
    ▼
formatCompactSummary()
    │
    ▼
buildPostCompactMessages()
    │
    ▼
新消息数组: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 Уровень 1: Микросжатие (Microcompact)

Микросжатие — первая линия обороны в управлении контекстом. Оно выполняется **перед каждым API-вызовом** (точка входа `microcompactMessages`) с целью высвободить пространство в контексте с минимальными затратами.

### 8.4.1 Белый список сжимаемых инструментов

Микросжатие работает только с выводом определённых инструментов. Принцип проектирования белого списка: **очищать только воспроизводимый контент**.

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // 文件读取 — 可重新读取
  ...SHELL_TOOL_NAMES,     // Shell 命令 — 可重新执行
  GREP_TOOL_NAME,          // 搜索 — 可重新搜索
  GLOB_TOOL_NAME,          // 文件匹配 — 可重新匹配
  WEB_SEARCH_TOOL_NAME,    // 网络搜索 — 可重新搜索
  WEB_FETCH_TOOL_NAME,     // 网络抓取 — 可重新抓取
  FILE_EDIT_TOOL_NAME,     // 文件编辑 — 结果已保存到磁盘
  FILE_WRITE_TOOL_NAME,    // 文件写入 — 同上
])
```

Обратите внимание, в `apiMicrocompact.ts` определено более детальное разграничение:

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // 清除 tool_result 内容
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // 清除 tool_use 输入
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

Это разграничение тонко продумано: для Read/Grep/Shell очищается **вывод** (tool_result); для Edit/Write очищается **ввод** (tool_use input), поскольку ввод операции редактирования (содержимое diff) велик, но результат уже сохранён на диске.

### 8.4.2 Подробное описание двух подпутей

Микросжатие имеет два взаимоисключающих пути выполнения, управляемых функцией `microcompactMessages()`:

```typescript
// microCompact.ts:287-317 — 调度逻辑
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // 路径 A: 时间触发 — 优先级最高，短路后续路径
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // 路径 B: 缓存编辑 — 仅主线程，仅支持特定模型
  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    const model = toolUseContext?.options.mainLoopModel ?? getMainLoopModel()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  return { messages }
}
```

**Путь A: Time-based Microcompact (по временному триггеру)**

Срабатывает, когда пользователь возвращается в сессию после превышения настроенного временного порога (по умолчанию 60 минут). Обоснование чётко изложено в `timeBasedMCConfig.ts`:

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

TTL серверного prompt cache — 1 час. Отсутствие пользователя более часа означает **гарантированное истечение кеша**, и весь prompt prefix нужно записывать заново. В этот момент очистка старых результатов инструментов «бесплатна» — не влечёт дополнительных потерь кеша.

Ключевая логика временного триггера:

```typescript
// microCompact.ts:381-389 — 时间触发评估
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) {
    return null
  }
  const gapMinutes =
    (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  return { gapMinutes, config }
}
```

Стратегия очистки после временного триггера также использует LRU (`keepRecent` по умолчанию 5), но с граничной защитой:

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

`Math.max(1, ...)` защищает от ловушки JavaScript: при `keepRecent=0` `slice(-0)` возвращает полный массив — классический случай «защитного программирования против семантической неоднозначности».

После временного триггера необходимо также сбросить состояние кешированного редактирования:

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**Путь B: Cached Microcompact (кешированное редактирование)**

Это продвинутый оптимизационный путь (только для внутреннего использования Anthropic, `feature('CACHED_MICROCOMPACT')`), использующий механизм `cache_edits` API для удаления результатов инструментов из серверного кеша **без изменения локального содержимого сообщений**.

```typescript
// microCompact.ts:327-370 — 缓存编辑路径核心
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // 注册工具结果
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && 
            compactableToolIds.has(block.tool_use_id) &&
            !state.registeredTools.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  const toolsToDelete = mod.getToolResultsToDelete(state)
  if (toolsToDelete.length > 0) {
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits
    }
    // ...
    return {
      messages,  // 消息不变!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

Ключевое архитектурное решение: **массив сообщений остаётся неизменным** — `return { messages }` возвращает оригинальную ссылку. Редактирование кеша происходит на уровне API (через параметр `cache_edits`), локальное состояние сохраняется в полном объёме. Это означает, что при сбое или повторной попытке API-вызова нет никаких локальных побочных эффектов.

### 8.4.3 Управление состоянием в кешированном редактировании

Путь кешированного редактирования поддерживает три группы ключевых состояний:

```typescript
// microCompact.ts:43-49 — 模块级状态
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

Управление жизненным циклом этих трёх состояний тонко организовано:

- `pendingCacheEdits` одноразовое — `consumePendingCacheEdits()` читает и очищает его (`microCompact.ts:80-84`); вызывающий код должен закрепить (pin) его после отправки в API-запросе.
- `pinnedCacheEdits` накопительное — каждое успешное редактирование кеша закрепляется на конкретной позиции user message, и последующие запросы должны повторно отправлять его с той же позиции для обеспечения попадания в кеш.
- `cachedMCState` сбрасывается после сжатия (`resetMicrocompactState()`) или после временного триггера.

```typescript
// microCompact.ts:78-105 — 状态消费与 pin
export function consumePendingCacheEdits() {
  const edits = pendingCacheEdits
  pendingCacheEdits = null
  return edits
}

export function getPinnedCacheEdits() {
  if (!cachedMCState) return []
  return cachedMCState.pinnedEdits
}

export function pinCacheEdits(
  userMessageIndex: number,
  block: import('./cachedMicrocompact.js').CacheEditsBlock,
): void {
  if (cachedMCState) {
    cachedMCState.pinnedEdits.push({ userMessageIndex, block })
  }
}
```

### 8.4.4 Вспомогательные функции оценки числа токенов

Модуль микросжатия предоставляет общесистемную функцию оценки числа токенов:

```typescript
// microCompact.ts:155-194 — estimateMessageTokens
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue
    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 固定 2000
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'tool_use') {
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {}),
        )
      }
      // ...
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // 4/3 保守填充
}
```

Базовая формула `roughTokenCountEstimation` предельно лаконична: `Math.round(content.length / 4)` (`tokenEstimation.ts:203-207`). `estimateMessageTokens` затем умножает результат на консервативный коэффициент 4/3, что эквивалентно `text.length / 3`. Эта двойная консервативная стратегия обеспечивает крайне низкую вероятность недооценки.

## 8.5 Уровень 2: Автоматическое сжатие (Auto-Compact)

### 8.5.1 Формула расчёта порога

Порог срабатывания автоматического сжатия вычисляется по следующей формуле:

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

Вывод конкретных значений (на примере Claude Opus 200K):

```
contextWindow = 200,000
maxOutputTokens = 16,384 (или значение, специфичное для модели)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (на основе p99.99 = 17,387)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — 关键常量
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

Выбор `AUTOCOMPACT_BUFFER_TOKENS = 13 000` — инженерный компромисс: слишком малое значение приводит к слишком частому сжатию (каждое потребляет 5–15 с и один API-вызов), слишком большое — к потере доступного контекста. 13 тыс. — это примерно пространство для 3–5 обычных итераций диалога.

### 8.5.2 Дерево решений shouldAutoCompact

```typescript
// autoCompact.ts:127-178 — 完整决策链
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. 递归保护：session_memory 和 compact 查询源不触发
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. 上下文折叠保护：marble_origami（ctx-agent）不触发
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. 配置检查：用户是否启用
  if (!isAutoCompactEnabled()) return false

  // 4. 响应式模式：如果启用，抑制主动压缩
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. 上下文折叠模式：折叠 IS 上下文管理，压缩不应干预
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Token 计数 + 阈值比较
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

Это дерево решений обнажает несколько стратегий управления контекстом, параллельно тестируемых в Claude Code:
- **Reactive Compact** (`tengu_cobalt_raccoon`): не сжимать проактивно, дождаться сообщения API о prompt_too_long
- **Context Collapse** (`CONTEXT_COLLAPSE`): управлять контекстом потоковым способом с порогами 90% submit / 95% block
- **Auto Compact** (текущий по умолчанию): проактивно сжимать при достижении порога

Три стратегии взаимно исключают друг друга и управляются через feature flags.

### 8.5.3 Механизм circuit breaker

```typescript
// autoCompact.ts:219-272 — autoCompactIfNeeded 含断路器
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // 断路器检查
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // 熔断状态，直接跳过
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // 优先尝试 Session Memory 压缩
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 传统压缩
  try {
    const compactionResult = await compactConversation(...)
    runPostCompactCleanup(querySource)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const nextFailures = (tracking?.consecutiveFailures ?? 0) + 1
    if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
      logForDebugging(
        `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`,
        { level: 'warn' })
    }
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

### 8.5.4 Порядок выполнения autoCompactIfNeeded

Полный порядок выполнения:

1. **Проверка переменной окружения**: `DISABLE_COMPACT` → глобальное отключение
2. **Проверка circuit breaker**: `consecutiveFailures >= 3` → пропуск
3. **Проверка порога**: `shouldAutoCompact()` → многоуровневые шлюзы
4. **Сжатие Session Memory** (приоритетный путь): использование существующей session memory вместо API-вызова
5. **Традиционное сжатие через Fork Agent** (резервный путь): полная генерация резюме на основе API
6. **Обработка сбоя**: инкремент счётчика circuit breaker, передача в следующую итерацию

## 8.6 Уровень 3: Полное сжатие (Full Compact)

### 8.6.1 Механизм Fork Agent

Ядром традиционного сжатия является генерация резюме диалога через Fork Agent. Функция `streamCompactSummary()` (`compact.ts:1136-1396`) реализует двухуровневую стратегию отказоустойчивости:

**Первый уровень: Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — 缓存共享 fork
if (promptCacheSharingEnabled) {
  try {
    const result = await runForkedAgent({
      promptMessages: [summaryRequest],
      cacheSafeParams,
      canUseTool: createCompactCanUseTool(),
      querySource: 'compact',
      forkLabel: 'compact',
      maxTurns: 1,
      skipCacheWrite: true,
      overrides: { abortController: context.abortController },
    })
    // ...
  }
}
```

Fork Agent повторно использует полный prompt cache основного диалога (системный промпт + инструменты + сообщения контекста), добавляя лишь один запрос на резюме. Ключевые решения:

1. `maxTurns: 1` — многоитерационное взаимодействие не допускается
2. `canUseTool: createCompactCanUseTool()` — все вызовы инструментов отклоняются
3. `skipCacheWrite: true` — без записи в кеш (временное ответвление)
4. **maxOutputTokens не задаётся** — комментарий поясняет: задание изменило бы thinking config, вызывая несовпадение cache key

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**Второй уровень: Streaming Fallback**

При сбое Fork Agent происходит откат к прямому потоковому API-вызову, при котором **можно** задать `maxOutputTokensOverride`:

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

Потоковый откат также поддерживает настраиваемые повторные попытки (`tengu_compact_streaming_retry`), не более `MAX_COMPACT_STREAMING_RETRIES = 2` раз.

### 8.6.2 Конвейер предварительной обработки

Сообщения перед сжатием проходят трёхэтапную предобработку:

```typescript
// compact.ts:1293-1300 — 预处理链
normalizeMessagesForAPI(
  stripImagesFromMessages(
    stripReinjectedAttachments([
      ...getMessagesAfterCompactBoundary(messages),
      summaryRequest,
    ]),
  ),
  context.options.tools,
)
```

1. `getMessagesAfterCompactBoundary` — берёт только сообщения после последнего сжатия
2. `stripReinjectedAttachments` — удаляет вложения `skill_discovery` / `skill_listing` (они будут повторно инжектированы после сжатия)
3. `stripImagesFromMessages` — заменяет блоки изображений текстовым маркером `[image]` (`compact.ts:144-199`)

Причина существования `stripImagesFromMessages` практична:

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

Пользователи CCD (Claude Code Desktop) часто прикрепляют скриншоты; без удаления изображений сам API-вызов сжатия может упасть из-за слишком длинного промпта.

### 8.6.3 Формат резюме из 9 разделов

`prompt.ts` определяет 9 структурированных разделов, которым должно следовать резюме:

```
1. Primary Request and Intent    — намерение пользователя
2. Key Technical Concepts        — технические концепции
3. Files and Code Sections       — файлы и фрагменты кода
4. Errors and fixes              — ошибки и исправления
5. Problem Solving               — решение задач
6. All user messages             — все сообщения пользователя (не результаты инструментов)
7. Pending Tasks                 — ожидающие задачи
8. Current Work                  — текущая работа
9. Optional Next Step            — следующий шаг (опционально)
```

Раздел 6 особенно важен: «List ALL user messages that are not tool results». Это гарантирует, что даже после сжатия диалога оригинальные формулировки пользователя сохраняются полностью. Это **гарантия нулевых потерь пользовательской обратной связи**.

Раздел 9 имеет тщательно спроектированное ограничение:

```
// prompt.ts — 第 9 章的约束
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

Это предотвращает «самодеятельность» модели после сжатия — только явно запрошенные пользователем следующие шаги будут зафиксированы.

### 8.6.4 Трёхуровневая защита NO_TOOLS_PREAMBLE от обходов

Fork Agent наследует полный набор инструментов основного диалога (для совпадения cache key), но агент сжатия не должен использовать ни один инструмент. Это создаёт противоречие: инструменты присутствуют в схеме, но вызываться не должны.

Решение — **трёхуровневое отклонение инструментов**:

```typescript
// prompt.ts:16-24 — 第一层：prompt 开头的强力声明
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — 第二层：prompt 结尾的重复提醒
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — 第三层：代码级拒绝
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

Комментарий раскрывает реальную причину этих трёх уровней:

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

На Sonnet 4.6 только с помощью инструкции промпта модель всё равно с вероятностью 2,79% пытается вызвать инструмент (на 4.5 — лишь 0,01%). `createCompactCanUseTool` — последняя программная защита.

### 8.6.5 Постобработка (formatCompactSummary)

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // 剥离 <analysis> 草稿区 — 提升摘要质量的中间推理，不需要保留
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // 提取 <summary> 内容
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${summaryMatch[1].trim()}`)
  }
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')
  return formattedSummary.trim()
}
```

Тег `<analysis>` — это приём Chain-of-Thought: модель сначала делает «черновик» в зоне анализа, а финальный результат выдаёт в `<summary>`. Наличие зоны анализа повышает качество резюме, но в финальный вывод она не включается — она содержит избыточные промежуточные рассуждения, которые расходовали бы контекст последующих итераций.

### 8.6.6 Последовательность сообщений после сжатия и повторная инъекция вложений

После завершения сжатия новая последовательность сообщений строится функцией `buildPostCompactMessages()`:

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // 系统消息：标记压缩边界
    ...result.summaryMessages,    // 用户消息：摘要内容
    ...(result.messagesToKeep ?? []),  // 保留的原始消息
    ...result.attachments,        // 文件附件 + 技能 + 计划
    ...result.hookResults,        // SessionStart 钩子结果
  ]
}
```

Повторная инъекция вложений — сложный процесс (`compact.ts:532-585`), включающий:

1. **Файловые вложения**: последние 5 файлов по истории доступа, с ограничением в 50 тыс. токенов, не более 5 тыс. токенов на файл
2. **Файл плана**: при наличии активного плана
3. **Инструкции режима планирования**: при нахождении в plan mode
4. **Содержимое скиллов**: содержимое использованных скиллов, отсортированное по последнему использованию, не более 5 тыс. токенов на скилл, общий бюджет 25 тыс. токенов
5. **Deferred Tools Delta**: повторное объявление схем отложенных инструментов
6. **Agent Listing Delta**: повторное объявление списка агентов
7. **MCP Instructions Delta**: повторное объявление инструкций MCP-серверов

### 8.6.7 Механизм PTL-повтора (Prompt-Too-Long Recovery)

Когда API-вызов сжатия сам падает из-за слишком длинного промпта, система повторяет его с прогрессивным усечением:

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // 先清除之前重试留下的标记消息
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // 精确截断：根据 API 返回的 token 差值
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // 模糊截断：丢弃 20% 的消息组
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // 至少保留一组
  if (dropCount < 1) return null

  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

Лимит повторных попыток — `MAX_PTL_RETRIES = 3`. Стратегия усечения имеет два пути:
- **Точный путь**: ошибка API содержит разницу токенов → отбрасывать группы по очереди до достижения нужного числа
- **Приближённый путь** (нестандартные форматы ошибок Vertex/Bedrock и др.): каждый раз отбрасывать 20%

Граничная обработка в строке 283: после отбрасывания группы 0 последовательность сообщений может начинаться с assistant-сообщения, нарушая ограничения API (первое сообщение должно быть user). Система вставляет синтетическое маркерное user-сообщение для исправления.

### 8.6.8 Два направления частичного сжатия (Partial Compact)

`partialCompactConversation()` (`compact.ts:772-1106`) поддерживает два направления:

```
Direction 'from': 
  [压缩后保留] | pivot | [被摘要的消息]
  → 保留 prompt cache（保留的在前，cache prefix 不变）

Direction 'up_to':
  [被摘要的消息] | pivot | [压缩后保留]
  → prompt cache 失效（摘要在前，前缀改变）
```

Направление `up_to` имеет дополнительную логику очистки — из сохраняемых сообщений необходимо удалить старые compact boundary и резюме:

```typescript
// compact.ts:791-799
const messagesToKeep =
  direction === 'up_to'
    ? allMessages.slice(pivotIndex)
        .filter(m =>
          m.type !== 'progress' &&
          !isCompactBoundaryMessage(m) &&
          !(m.type === 'user' && m.isCompactSummary))
    : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

Комментарий объясняет причину: в режиме `up_to` резюме стоит перед сохраняемыми сообщениями, а старое boundary вводит в заблуждение обратное сканирование `findLastCompactBoundaryIndex`.

## 8.7 Уровень 4: Сжатие Session Memory

### 8.7.1 Ключевая идея и преимущества

Сжатие Session Memory (`sessionMemoryCompact.ts`) является оптимизированной альтернативой традиционному сжатию. Ключевая идея: использовать session memory, непрерывно извлекаемую фоновым агентом (инкрементные резюме диалога), вместо резюме, генерируемых Fork Agent в реальном времени.

Преимущества:
- **Ноль дополнительных API-вызовов**: session memory непрерывно поддерживается фоновым агентом, при сжатии используется напрямую
- **Более низкая задержка**: не нужно ждать 5–15 с ответа API
- **Более точное сохранение**: возможность точно рассчитать, сколько последних сообщений сохранить

### 8.7.2 Подробное описание алгоритма calculateMessagesToKeepIndex

Это ядро алгоритма сжатия Session Memory (`sessionMemoryCompact.ts:262-327`), определяющее, сколько сообщений сохранить после сжатия:

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // 从 lastSummarizedIndex + 1 开始（session memory 已覆盖之前的内容）
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // 计算当前保留范围的 token 和文本消息数
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // 已达上限 → 不扩展
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 已满足两个最低要求 → 不扩展
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 向前扩展，但不越过最后一个 compact boundary
  const idx = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  const floor = idx === -1 ? 0 : idx + 1

  for (let i = startIndex - 1; i >= floor; i--) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
    startIndex = i
    if (totalTokens >= config.maxTokens) break
    if (totalTokens >= config.minTokens && 
        textBlockMessageCount >= config.minTextBlockMessages) break
  }

  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

Параметры конфигурации (могут быть переопределены через удалённую конфигурацию GrowthBook):

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // 至少保留 10K token
  minTextBlockMessages: 5,     // 至少保留 5 条有文本的消息
  maxTokens: 40_000,           // 最多保留 40K token
}
```

Двойное ограничение алгоритма (`minTokens` И `minTextBlockMessages`) гарантирует:
- Расширение не остановится из-за нескольких очень крупных сообщений (требование по токенам выполнено, но сообщений слишком мало)
- Не будет сохранено слишком много мелких сообщений при реально недостаточном числе токенов

**Механизм Floor**: при расширении вперёд нельзя пересекать последнее compact boundary (`floor = lastBoundaryIndex + 1`). Комментарий объясняет причину:

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

Цепочка сообщений на уровне дискового хранения имеет разрыв на месте compact boundary; пересечение его приводит к тому, что обратный обход загрузчика пропускает сохранённые сообщения.

### 8.7.3 Исправление ошибок в adjustIndexToPreserveAPIInvariants

Эта функция (`sessionMemoryCompact.ts:172-260`) — наиболее тонко проработанный фрагмент кода во всей системе сжатия; она решает два нарушения инвариантов API:

**Сценарий ошибки 1: Осиротевший tool_result**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

Если startIndex = N+2:
  Старый код проверяет только tool_results сообщения N+2 → не находит → возвращает N+2
  После объединения normalizeMessagesForAPI по message.id:
    msg[1]: assistant с [tool_use: VALID_ID]  (ORPHAN tool_use исключён!)
    msg[2]: user с [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → Ошибка API: orphan tool_result ссылается на несуществующий tool_use
```

**Сценарий ошибки 2: Потерянный thinking-блок**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

Если startIndex = N+1:
  Thinking-блок в N исключён
  normalizeMessagesForAPI не может объединить (нет сообщения с тем же ID для объединения)
  → Thinking-блок потерян навсегда
```

Исправляющий код выполняет два этапа корректировки:

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Step 1: 处理 tool_use/tool_result 配对
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... 收集保留范围内已有的 tool_use IDs
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // 向前搜索缺失的 tool_use
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // 删除已找到的 ID
      }
    }
  }

  // Step 2: 处理共享 message.id 的 thinking 块
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id)
      messageIdsInKeptRange.add(messages[i]!.message.id)
  }
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id &&
        messageIdsInKeptRange.has(messages[i]!.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

Ключевой инсайт этого кода: потоковые ответы Claude API разбивают один API-ответ на несколько assistant-сообщений (с общим `message.id`, но разными UUID), в которых thinking-блоки и tool_use-блоки разделены. `normalizeMessagesForAPI` объединяет эти сообщения по `message.id` — если сжатие разрезает группу сообщений с одинаковым ID, после объединения возникает несогласованность.

### 8.7.4 Полный поток trySessionMemoryCompaction

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. 门禁检查
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. 初始化远程配置（仅首次）
  await initSessionMemoryCompactConfig()

  // 3. 等待正在进行的 session memory 提取完成
  await waitForSessionMemoryExtraction()

  // 4. 获取 session memory 内容
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. 确定边界
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID 不存在 → 回退
  } else {
    // Resumed session: 没有边界 → 从末尾开始
    lastSummarizedIndex = messages.length - 1
  }

  // 6. 计算保留范围
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // 过滤旧 boundary

  // 7. 运行 session start hooks
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. 构建结果
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. 阈值检查（仅 autocompact）
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // 压缩后仍超阈值 → 回退到传统压缩
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 Параметры конфигурации (удалённая конфигурация GrowthBook)

Все ключевые параметры сжатия Session Memory могут быть переопределены через удалённую конфигурацию GrowthBook:

```typescript
// sessionMemoryCompact.ts:91-109 — 远程配置初始化
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // 防御性编码：只使用正数值，忽略 0 值
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

Шлюз функции управляется двумя независимыми feature flags:

```typescript
// sessionMemoryCompact.ts:333-349
export function shouldUseSessionMemoryCompaction(): boolean {
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) return true
  if (isEnvTruthy(process.env.DISABLE_CLAUDE_CODE_SM_COMPACT)) return false
  
  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory', false)
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact', false)
  return sessionMemoryFlag && smCompactFlag
}
```

## 8.8 Свёртывание контекста и хранение результатов инструментов

### 8.8.1 Механизм collapseReadSearch

`utils/collapseReadSearch.ts` (1 109 строк) реализует свёртывание сообщений на уровне UI — группирует последовательные операции поиска/чтения в однострочное резюме (например, «Read 5 files, searched 3 patterns»).

Основная логика классификации (`getToolSearchOrReadInfo`, `collapseReadSearch.ts:142-237`) делит вызовы инструментов на:

| Категория | Сворачиваемая | Поведение свёртывания |
|-----------|---------------|----------------------|
| Read (file_path) | Да | «Read N files» |
| Search (Grep/Glob) | Да | «Searched N patterns» |
| Shell (Bash) | Да в полноэкранном режиме | «Ran N bash commands» |
| REPL | Да (тихо поглощается) | Независимый счётчик внутренних инструментов |
| Memory Write | Да | Специальный маркер |
| ToolSearch | Да (тихо поглощается) | Счётчик не увеличивается |
| Edit/Write (не memory) | Нет | Прерывает группу свёртывания |

«Тихое поглощение» (`isAbsorbedSilently`) — тонко спроектированный механизм: REPL и ToolSearch не увеличивают счётчик, но и не прерывают текущую группу свёртывания. Это означает, что `[Read, ToolSearch, Read]` сворачивается в «Read 2 files», а не разрезается ToolSearch на две группы.

Свёртывание — **оптимизация только для UI**: оно не изменяет содержимое сообщений, отправляемых в API, и влияет лишь на отображение в терминале.

### 8.8.2 Стратегия дискового хранения в toolResultStorage

`utils/toolResultStorage.ts` (1 040 строк) является «нулевым уровнем» управления контекстом — обрабатывает сверхкрупные результаты до их попадания в историю диалога.

**Разбор порога сохранения**:

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Read 工具特殊：Infinity → 不持久化（Read 自身有 maxTokens 限制）
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // GrowthBook 覆盖（tengu_satin_quoll）
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // 默认：min(工具声明值, 全局 50K 默认值)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**Оптимизация дедупликации**: `tool_use_id` уникален, используется `flag: 'wx'` (exclusive write) для предотвращения повторной записи:

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: 已在之前的轮次持久化，跳过
}
```

**Обработка пустых результатов**:

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

Это исправление устраняет ошибку в поведении модели: пустой tool_result заставлял некоторые модели распознавать шаблон `\n\nHuman:` как конец диалога.

**Per-Message Aggregate Budget** (`enforceToolResultBudget`, `toolResultStorage.ts:769-908`):

Это наиболее сложная функция в `toolResultStorage.ts` — принудительное соблюдение суммарного бюджета размера результатов инструментов на уровне user-сообщений API (после объединения `normalizeMessagesForAPI`).

Ключевые аспекты проектирования:
- **Заморозка состояния** (`ContentReplacementState`): после того, как tool_result был «увиден» (отправлен модели), решение о нём (заменить/не заменять) замораживается и не изменяется в течение жизненного цикла — это гарантирует стабильность prompt cache
- **Трёхзонная** стратегия: `mustReapply` (ранее заменённые → повторно применить кешированную замену), `frozen` (ранее просмотренные, но не заменённые → не трогать), `fresh` (новые → возможна замена)
- **Приоритет максимального**: при необходимости замены выбирается наибольший fresh-результат с наивысшим приоритетом

## 8.9 Роль сжатия в пятиуровневом восстановлении после ошибок

### 8.9.1 Полный пятиуровневый механизм восстановления после ошибок

Система сжатия играет множество ролей в механизме восстановления после ошибок Claude Code:

| Уровень | Условие срабатывания | Действие сжатия | Источник |
|---------|---------------------|-----------------|---------|
| L1 | API возвращает prompt_too_long (413) | Reactive Compact: усечение + резюме заново | `compactMessages.ts` |
| L2 | Сам API-вызов сжатия возвращает 413 | PTL Retry: усечение старейших групп сообщений × 3 раза | `compact.ts:truncateHeadForPTLRetry` |
| L3 | После сжатия всё ещё выше порога | Re-compaction: автоматическое повторное сжатие | `autoCompact.ts:recompactionInfo` |
| L4 | 3 последовательных сбоя сжатия | Circuit Breaker: прекращение попыток | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent не выдал текстовый вывод | Streaming Fallback: прямой потоковый API-вызов | `compact.ts:streamCompactSummary` |

### 8.9.2 Реактивное сжатие vs проактивное сжатие

Компромиссы двух стратегий:

**Проактивное сжатие** (Auto-Compact, текущий вариант по умолчанию):
- Срабатывает при достижении порога токенов
- Плюс: более плавный пользовательский опыт, не возникает ошибок 413
- Минус: возможно преждевременное сжатие с потерей доступного контекста

**Реактивное сжатие** (Reactive Compact, эксперимент `tengu_cobalt_raccoon`):
- Ждёт сообщения API prompt_too_long
- Плюс: максимизирует использование контекста
- Минус: заметное прерывание для пользователя из-за ожидания повтора

Взаимоисключающая связь в коде:

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // 响应式模式下不主动压缩
  }
}
```

## 8.10 Группировка сообщений и оценка числа токенов

### 8.10.1 Алгоритм groupMessagesByApiRound

`grouping.ts` (63 строки) группирует сообщения по API-итерациям — каждая группа соответствует одному полному циклу API:

```typescript
// grouping.ts:28-62
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (msg.type === 'assistant' && 
        msg.message.id !== lastAssistantId && 
        current.length > 0) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }
  if (current.length > 0) groups.push(current)
  return groups
}
```

Единственный критерий границы группы — изменение `message.id`: несколько потоковых блоков одного API-ответа разделяют одинаковый `message.id`, поэтому они естественно попадают в одну группу.

Этот подход заменил предыдущую группировку «по человеческим итерациям» (разбивка только на реальных пользовательских сообщениях), которая не могла обрабатывать долгие одноитерационные agent-сессии в SDK/CCR/eval-сценариях.

### 8.10.2 roughTokenCountEstimation и консервативное заполнение

Оценка числа токенов использует двухуровневую консервативную стратегию:

**Первый уровень**: Базовая оценка

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

По умолчанию 4 байта/токен; для JSON-файлов 2 байта/токен (JSON содержит много однобайтных токенов: `{`, `}`, `:`, `,`).

**Второй уровень**: Заполнение на уровне сообщений

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

Совокупный эффект: для обычного текста эффективная оценка равна `text.length / 4 * 4/3 = text.length / 3`.

### 8.10.3 Смешанная стратегия: точность vs оценка

Система использует разную точность в разных сценариях:

| Сценарий | Точность | Источник | Задержка |
|----------|---------|---------|---------|
| shouldAutoCompact | Смешанная: приоритет точному значению из ответа API | `tokenCountWithEstimation` | 0 (кешировано) |
| estimateMessageTokens | Грубая оценка (`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | Грубая оценка | `estimateMessageTokens` | 0 |
| Подсчёт токенов после сжатия | Точный | `tokenCountFromLastAPIResponse` | 0 (API уже ответил) |

Смешанная стратегия `tokenCountWithEstimation`: приоритет отдаётся `usage.input_tokens` из последнего ответа API (точное значение); если недоступно (например, до первого запроса) — откат к оценке.

## 8.11 Анализ архитектурных решений

### 8.11.1 Философия постепенной деградации

Управление контекстом Claude Code следует принципу **постепенной деградации без пропуска уровней**: каждый уровень пытается решить проблему с минимальными затратами и только при неудаче переходит на следующий. Это предотвращает распространённую проблему «чрезмерной реакции» — например, полного сжатия из-за одного большого результата Read файла.

Сравнение с отраслевыми практиками:
- **ChatGPT**: усечение старых сообщений (просто, но грубо)
- **GitHub Copilot Chat**: фиксированное окно контекста + последние N сообщений (без сжатия)
- **Claude Code**: 5-уровневое нарастание (профилактика → тонкая настройка → резюме → аварийное восстановление)

### 8.11.2 Проектирование с приоритетом кеша

Prompt cache — жизненная артерия Claude Code: для запроса в 200 тыс. токенов, если 180 тыс. — cache read ($0,30/млн токенов), стоимость в 10 раз ниже, чем полный cache miss ($3/млн токенов). Практически все архитектурные решения служат этому экономическому ограничению:

1. **Fork Agent разделяет кешированный prefix**: API-вызов сжатия повторно использует кеш основного диалога
2. **maxOutputTokens не задаётся во Fork**: избегает несовпадения thinking config → cache miss
3. **Cached MC не изменяет локальные сообщения**: сохраняет неизменный prompt prefix
4. **ContentReplacementState замораживает просмотренные ID**: гарантирует неизменность решения о замене tool_result в течение жизненного цикла
5. **sentSkillNames не сбрасывается**: предотвращает повторную инъекцию ~4 тыс. токенов skill_listing
6. **pinnedCacheEdits повторно отправляются на фиксированной позиции**: гарантирует постоянство позиции cache edit

### 8.11.3 Гарантии безопасности

Система поддерживает три класса инвариантов:

**Нераздельность пар**: `adjustIndexToPreserveAPIInvariants` гарантирует, что tool_use и tool_result никогда не окажутся по разные стороны. Это требование не только функциональной корректности (API выдаст ошибку), но и семантической (модель должна видеть результаты вызванных ею инструментов).

**Защита от рекурсии**: проверка `querySource` в `shouldAutoCompact` гарантирует, что агенты session_memory, compact, context collapse не вызывают автосжатие — эти агенты сами являются частью управления контекстом, и рекурсивное сжатие привело бы к дедлоку.

**Механизм circuit breaker**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` установлено на основе реальных данных (петли сбоев в 1 279 сессиях), превращая бесконечные повторы в ограниченные + срабатывание защиты.

### 8.11.4 Сравнение с API-native управлением контекстом

`apiMicrocompact.ts` раскрывает направление исследований Claude Code — передача части управления контекстом на уровень API:

```typescript
// apiMicrocompact.ts:37-47
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

Эти стратегии `context_management.edits` объявляются непосредственно в API-запросе и исполняются на стороне сервера. Преимущество — меньшая задержка (не требует клиентской обработки) и точное согласование с серверным подсчётом токенов. Стратегия очистки инструментов пока доступна только для внутренних пользователей (`USER_TYPE === 'ant'`), внешние используют только очистку thinking.

## 8.12 Переносимые паттерны

### 8.12.1 Общие паттерны проектирования для многоуровневых систем сжатия

Из управления контекстом Claude Code выведены следующие переносимые общие паттерны:

**Паттерн 1: Многоуровневое вытеснение (Tiered Eviction)**
- Применять разные стратегии вытеснения для разных типов контента
- Воспроизводимый контент (вывод инструментов) вытесняется в первую очередь; невоспроизводимый (пользовательский ввод) — в последнюю
- Реализация: белый список + приоритетная сортировка

**Паттерн 2: Гибридная оценка (Hybrid Estimation)**
- Быстрые решения — с грубой оценкой (`text.length / 3`); точный расчёт — с данными из ответа API
- Грубая оценка всегда консервативная (лучше переоценить и сжать раньше, чем недооценить и получить ошибку API)

**Паттерн 3: Заморозка-воспроизведение (Freeze-Replay)**
- После того, как контент «увиден» моделью, решение о его обработке замораживается
- В последующих итерациях к замороженному контенту применяется только «воспроизведение» (повторное применение кешированной замены), без новых решений
- Гарантирует побитовую стабильность prompt prefix → попадание в кеш

**Паттерн 4: Усечение с учётом границ (Boundary-Aware Truncation)**
- Никогда не усекать посередине семантической единицы (пара tool_use/tool_result, группа сообщений с одинаковым ID)
- Активное исправление после усечения (вставка синтетических сообщений, корректировка индексов)

**Паттерн 5: Защита circuit breaker (Circuit Breaker Protection)**
- Ввести счётчик сбоев для операций, которые могут уйти в бесконечный повтор
- Устанавливать пороги на основе реальных операционных данных (а не интуиции)

### 8.12.2 Что можно заимствовать для Doramagic

В конвейере Soul Extractor Doramagic процесс извлечения может порождать большое количество промежуточных результатов (фрагменты кода, API-документация, обсуждения в сообществах). Применимые паттерны:

1. **Многоуровневый кеш извлечения**: аналогично механизму белого списка microcompact — классифицировать промежуточные API-ответы и результаты анализа кода по воспроизводимости, вытесняя в первую очередь повторно получаемый контент
2. **Инкрементные резюме**: аналогично Session Memory Compact — поддерживать инкрементные резюме извлечённых знаний вместо полной истории
3. **Заморозка решений**: после подтверждения блока знаний как «ценного» или «незначимого» решение необратимо — исключает повторную переоценку в разных итерациях извлечения

## 8.13 Индекс исходного кода

| Файл | Строк | Ключевая ответственность |
|------|-------|--------------------------|
| `services/compact/compact.ts` | ~1 705 | Основная логика традиционного сжатия: Fork Agent, PTL-повтор, повторная инъекция вложений, частичное сжатие |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Сжатие Session Memory: calculateMessagesToKeepIndex, adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | Микросжатие: временной триггер, кешированное редактирование, оценка токенов |
| `services/compact/prompt.ts` | ~374 | Промпт сжатия: шаблон из 9 разделов, NO_TOOLS_PREAMBLE, formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | Автосжатие: расчёт порога, цепочка решений shouldAutoCompact, circuit breaker |
| `services/compact/apiMicrocompact.ts` | ~153 | API-native управление контекстом: clear_tool_uses, clear_thinking |
| `services/compact/grouping.ts` | ~63 | Группировка сообщений: groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | Очистка после сжатия: сброс кеша, состояний модулей, классификаторов |
| `services/compact/timeBasedMCConfig.ts` | ~43 | Конфигурация временного триггера: удалённая конфигурация GrowthBook |
| `services/compact/compactWarningHook.ts` | ~16 | React hook: подписка на состояние подавления предупреждения о сжатии |
| `services/compact/compactWarningState.ts` | ~18 | Хранилище состояний: флаг подавления предупреждения о сжатии |
| `services/cost-tracker.ts` | ~323 | Отслеживание затрат: тарификация токенов, статистика использования моделей |
| `utils/collapseReadSearch.ts` | ~1 109 | Свёртывание контекста: группировка и свёртывание сообщений на уровне UI |
| `utils/toolResultStorage.ts` | ~1 040 | Хранение результатов инструментов: дисковое сохранение, per-message budget, ContentReplacementState |
| `services/tokenEstimation.ts` | ~350+ | Оценка токенов: roughTokenCountEstimation (text.length/4) |
