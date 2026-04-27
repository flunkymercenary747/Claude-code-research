# Capítulo 8: Gestión de Contexto

## 8.1 Visión General y Posicionamiento

La gestión de contexto es uno de los subsistemas más críticos de la arquitectura de Claude Code. Una sesión de programación típica puede durar varias horas, involucrar cientos de llamadas a herramientas y generar cientos de miles de tokens de historial de conversación. Sin gestión, la ventana de contexto se agota después de 20-30 rondas de interacción, provocando la interrupción de la sesión.

El problema central que resuelve el sistema de gestión de contexto de Claude Code es: **¿Cómo mantener la continuidad de la sesión y la integridad de la información dentro de una ventana de contexto limitada (generalmente 200K tokens), minimizando al mismo tiempo la pérdida de información percibida por el usuario?**

El sistema está compuesto por 11 archivos en el directorio `services/compact/`, con aproximadamente 3,900 líneas de código TypeScript en total, complementados por dos módulos de herramientas clave: `utils/collapseReadSearch.ts` (1,109 líneas) y `utils/toolResultStorage.ts` (1,040 líneas). El diseño de todo el subsistema refleja tres principios fundamentales:

1. **Degradación Progresiva** (Graceful Degradation): desde la microcompresión de costo cero hasta la compresión completa con pérdida, aumentando gradualmente la intensidad de la intervención
2. **Caché Primero** (Cache-First): cada decisión de compresión prioriza la preservación del prompt cache
3. **Garantías de Seguridad** (Safety Invariants): el emparejamiento tool_use/tool_result no puede cortarse, protección contra recursión, mecanismo de circuit breaker

## 8.2 Fundamentos Teóricos

### 8.2.1 Perspectiva de la Teoría de la Información: Compresión con Pérdida vs. Sin Pérdida

La gestión de contexto es esencialmente un **problema de compresión de información**. El sistema multicapa de Claude Code corresponde a diferentes estrategias de compresión:

- **Compresión Sin Pérdida** (Lossless): la ruta `cache_edits` de la microcompresión — elimina las copias en caché del lado del servidor de los resultados de herramientas antiguas mediante el mecanismo de cache editing de la API, sin cambiar el contenido de los mensajes locales. El modelo ve el marcador `[Old tool result content cleared]`, pero los datos originales se guardan en disco (`toolResultStorage.ts`). La información no se pierde, solo se mueve del almacenamiento en caliente al almacenamiento en frío.
- **Compresión con Pérdida** (Lossy): la compresión completa genera un resumen mediante Fork Agent, comprimiendo decenas de miles de tokens de conversación en unos pocos miles. Este es un proceso de reducción de dimensionalidad irreversible — los detalles del código, las trazas de errores y el razonamiento intermedio pueden perderse.

Desde la perspectiva de la Rate-Distortion Theory, el diseño de Claude Code implica una **función de medida de distorsión**: los 9 capítulos del prompt de resumen (véase la sección 8.6) definen qué dimensiones de información son más intolerantes a la distorsión — "user messages" (preservados íntegramente) tienen mayor prioridad que "key technical concepts" (se permiten resúmenes).

### 8.2.2 Teoría de Caché: Localidad Temporal y Espacial

El mecanismo de lista blanca de la microcompresión refleja la suposición de **localidad temporal** (Temporal Locality) de la teoría clásica de caché:

> Los resultados de herramientas utilizados recientemente tienen más probabilidad de ser referenciados posteriormente.

La lista blanca (`COMPACTABLE_TOOLS`) en `microCompact.ts` es una manifestación de la política de desalojo — solo los resultados de herramientas específicas (Read, Shell, Grep, Glob, WebFetch, WebSearch, Edit, Write) pueden eliminarse, porque sus salidas son regenerables (se puede volver a ejecutar la herramienta para obtenerlos). El texto ingresado manualmente por el usuario, las imágenes y otros contenidos no regenerables nunca se eliminan.

```typescript
// microCompact.ts:30-41 — Lista blanca de herramientas comprimibles
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

El parámetro `keepRecent` (por defecto, conserva los últimos 5) implementa directamente la política de desalojo LRU (Least Recently Used).

### 8.2.3 Patrón Circuit Breaker (Disyuntor)

El mecanismo de circuit breaker en `autoCompact.ts` es una adaptación precisa del Circuit Breaker Pattern clásico de sistemas distribuidos a aplicaciones LLM. Este patrón se originó en *Release It!* de Michael Nygard, y su modelo de tres estados (Closed → Open → Half-Open) se implementa en Claude Code así:

```typescript
// autoCompact.ts:70-73 — Umbral del circuit breaker
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

Este comentario revela los datos reales de desastre antes de que existiera el circuit breaker: **1,279 sesiones quedaron atrapadas en bucles de fallos consecutivos de más de 50 veces**, con la sesión más grave alcanzando 3,272 intentos fallidos, desperdiciando aproximadamente 250K llamadas a la API globalmente cada día. La introducción del circuit breaker limitó el máximo de reintentos a 3.

| Estado | Comportamiento | Código correspondiente |
|------|------|---------|
| Closed (normal) | `consecutiveFailures < 3`, intento normal de compresión | Ruta por defecto de `autoCompactIfNeeded` |
| Open (disparado) | `consecutiveFailures >= 3`, omite la compresión | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open (sondeo) | Tras compresión exitosa, `consecutiveFailures` se reinicia a 0 | `consecutiveFailures: 0` en éxito |

## 8.3 Arquitectura General

### 8.3.1 Arquitectura General del Sistema Multicapa de Compresión

La gestión de contexto de Claude Code utiliza un diseño de **5 capas de defensa**. Ordenadas de menor a mayor intervención:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Solicitud del usuario                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage（Capa preventiva）                  │
│   Resultados grandes de herramientas → persistencia en disco     │
│   + vista previa de 2KB                                          │
│   Activación: resultado > umbral (por defecto 50K chars)         │
│   Costo: cero costo de contexto (guardado en disco,              │
│   vista previa en contexto)                                      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact（Microcompresión）                         │
│   Ruta A: activación por tiempo — elimina contenido de           │
│   resultados de herramientas antiguas                            │
│   Ruta B: edición de caché — cache_edits API elimina caché       │
│   del servidor                                                   │
│   Activación: antes de cada llamada a la API                     │
│   Costo: muy bajo (resultados de herramientas reemplazados       │
│   por marcadores, recuperables desde disco)                      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact（Compresión automática）                   │
│   Session Memory → Fork Agent → resumen completo                 │
│   Activación: tokens supera effectiveContextWindow - 13K         │
│   Costo: alto (resumen con pérdida, pérdida de detalles,         │
│   consume una llamada a la API)                                  │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact（Compresión manual）                    │
│   Activación por el usuario, soporta Partial Compact             │
│   Activación: comando del usuario                                │
│   Costo: igual que arriba                                        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact（Compresión reactiva）                 │
│   API devuelve prompt_too_long → truncar y reintentar            │
│   Activación: error 413                                          │
│   Costo: máximo (truncado de emergencia + resumen,               │
│   mayor pérdida de información)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 Comparación de Condiciones de Activación, Costo y Pérdida de Información por Capa

| Capa | Condición de activación | Momento | Latencia | Pérdida de información | Costo de API |
|------|---------|------|------|---------|---------|
| L0: Tool Result Storage | Resultado individual > umbral | Tras ejecución de herramienta | I/O de disco | Cero (original en disco) | Cero |
| L1a: Time-based MC | > 60min desde último assistant | Antes de llamada a API | Cero (operación local) | Baja (resultados antiguos eliminados) | Cero |
| L1b: Cached MC | Herramientas comprimibles superan umbral | Antes de llamada a API | Cero (cache_edits) | Baja (igual) | Cero |
| L2: Auto-Compact | tokens > umbral | Entre rondas | 5-15s (llamada a API) | Alta (resumen con pérdida) | 1 llamada a API |
| L3: Manual Compact | Usuario /compact | Activado por usuario | Igual | Media-Alta (el usuario puede guiar) | 1 llamada a API |
| L4: Reactive Compact | prompt_too_long 413 | Tras fallo de API | 10-30s (reintento) | Máxima (truncado+resumen) | 1-4 llamadas a API |

### 8.3.3 Flujo de Datos

```
Array de mensajes (Message[])
    │
    ▼
microcompactMessages()  ──→ [¿activación por tiempo?] ──Y──→ Eliminar contenido → retornar
    │ N                      │
    │                  [¿edición de caché?] ──Y──→ pendingCacheEdits → retornar
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     No comprimir, retornar directamente
    │ Y
    ▼
trySessionMemoryCompaction() ──→ [¿hay session memory?]
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
Nuevo array de mensajes: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 Capa 1: Microcompresión (Microcompact)

La microcompresión es la primera línea de defensa de la gestión de contexto. Se ejecuta **antes de cada llamada a la API** (punto de entrada `microcompactMessages`), con el objetivo de liberar espacio de contexto con el mínimo costo.

### 8.4.1 Lista Blanca de Herramientas Comprimibles

La microcompresión solo opera sobre las salidas de herramientas específicas. El principio de diseño detrás de la lista blanca es: **solo eliminar contenido regenerable**.

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // Lectura de archivos — se puede releer
  ...SHELL_TOOL_NAMES,     // Comandos Shell — se pueden re-ejecutar
  GREP_TOOL_NAME,          // Búsqueda — se puede volver a buscar
  GLOB_TOOL_NAME,          // Coincidencia de archivos — se puede re-ejecutar
  WEB_SEARCH_TOOL_NAME,    // Búsqueda web — se puede volver a buscar
  WEB_FETCH_TOOL_NAME,     // Obtención web — se puede volver a obtener
  FILE_EDIT_TOOL_NAME,     // Edición de archivos — resultado guardado en disco
  FILE_WRITE_TOOL_NAME,    // Escritura de archivos — igual que arriba
])
```

Nótese que `apiMicrocompact.ts` también define una distinción más granular:

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // Elimina contenido de tool_result
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // Elimina entrada de tool_use
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

La distinción aquí es sutil: para Read/Grep/Shell, se elimina la **salida** (tool_result); para Edit/Write, se elimina la **entrada** (tool_use input), porque la entrada de las operaciones de edición (contenido diferencial) es voluminosa pero el resultado ya está persistido en disco.

### 8.4.2 Descripción Detallada de las Dos Subrutas

La microcompresión tiene dos rutas de ejecución mutuamente excluyentes, gestionadas de manera unificada por la función `microcompactMessages()`:

```typescript
// microCompact.ts:287-317 — Lógica de despacho
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // Ruta A: activación por tiempo — máxima prioridad, cortocircuita rutas posteriores
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // Ruta B: edición de caché — solo hilo principal, solo modelos compatibles
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

**Ruta A: Time-based Microcompact (activación por tiempo)**

Se activa cuando el usuario regresa a la sesión después de exceder el umbral de tiempo configurado (por defecto 60 minutos). El motivo de diseño se expresa claramente en `timeBasedMCConfig.ts`:

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

El TTL del prompt cache del servidor es 1 hora. Si el usuario se ausenta más de 1 hora, el **caché necesariamente ha expirado** y todo el prefijo del prompt debe reescribirse. Eliminar resultados de herramientas antiguas en ese momento es "gratuito" — no genera costo adicional de invalidación de caché.

La lógica clave de la activación por tiempo:

```typescript
// microCompact.ts:381-389 — Evaluación de la activación por tiempo
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

La estrategia de eliminación tras la activación por tiempo también usa LRU (`keepRecent` por defecto 5), pero con una protección de límite:

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

Este `Math.max(1, ...)` previene la trampa de JavaScript donde `slice(-0)` devuelve el array completo cuando `keepRecent=0` — un caso típico de "programación defensiva para evitar ambigüedad semántica".

Tras la activación por tiempo también hay que reiniciar el estado de edición de caché:

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**Ruta B: Cached Microcompact (edición de caché)**

Esta es una ruta de optimización avanzada interna de Anthropic (`feature('CACHED_MICROCOMPACT')`), que utiliza el mecanismo `cache_edits` de la API para eliminar los resultados de herramientas del caché del servidor **sin modificar el contenido de los mensajes locales**.

```typescript
// microCompact.ts:327-370 — Núcleo de la ruta de edición de caché
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // Registrar resultados de herramientas
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
      messages,  // ¡Los mensajes no cambian!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

Decisión de diseño clave: **el array de mensajes no cambia** — `return { messages }` devuelve la referencia original. La edición de caché ocurre en la capa de API (a través del parámetro `cache_edits`), el estado local permanece íntegro. Esto significa que si la llamada a la API falla o se reintenta, no hay efectos secundarios locales.

### 8.4.3 Gestión de Estado de la Edición de Caché

La ruta de edición de caché mantiene tres grupos de estados clave:

```typescript
// microCompact.ts:43-49 — Estado a nivel de módulo
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

La gestión del ciclo de vida de estos tres estados es delicada:

- `pendingCacheEdits` es de un solo uso — `consumePendingCacheEdits()` lo limpia al leerlo (`microCompact.ts:80-84`), el llamador debe anclarlo (pin) después de enviarlo en la solicitud de API.
- `pinnedCacheEdits` es acumulativo — cada cache edit exitoso se ancla en la posición específica del mensaje de usuario, las solicitudes posteriores deben reenviarse en la misma posición para garantizar el acierto de caché.
- `cachedMCState` se reinicia tras la compresión (`resetMicrocompactState()`) o tras la activación por tiempo.

```typescript
// microCompact.ts:78-105 — Consumo de estado y pin
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

### 8.4.4 Funciones Auxiliares de Estimación de Tokens

El módulo de microcompresión proporciona funciones de estimación de tokens compartidas por todo el sistema:

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
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // Fijo en 2000
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
  return Math.ceil(totalTokens * (4 / 3))  // Relleno conservador 4/3
}
```

La fórmula central de `roughTokenCountEstimation` es extremadamente concisa: `Math.round(content.length / 4)` (`tokenEstimation.ts:203-207`). `estimateMessageTokens` aplica además un factor conservador de 4/3 a este resultado, equivalente a `text.length / 3`. Esta doble estrategia conservadora garantiza una probabilidad muy baja de subestimación.

## 8.5 Capa 2: Compresión Automática (Auto-Compact)

### 8.5.1 Fórmula de Cálculo del Umbral

El umbral de activación de la compresión automática se calcula con la siguiente fórmula:

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

Derivación de valores concretos (usando Claude Opus 200K como ejemplo):

```
contextWindow = 200,000
maxOutputTokens = 16,384 (o valor específico del modelo)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (basado en p99.99 = 17,387)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — Constantes clave
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

La elección de `AUTOCOMPACT_BUFFER_TOKENS = 13,000` es un compromiso de ingeniería: demasiado pequeño implica compresión demasiado frecuente (cada compresión consume 5-15 segundos y una llamada a la API), demasiado grande desperdicia contexto disponible. 13K es aproximadamente el espacio de 3-5 rondas de conversación ordinaria.

### 8.5.2 Árbol de Decisión de shouldAutoCompact

```typescript
// autoCompact.ts:127-178 — Cadena de decisión completa
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. Protección contra recursión: session_memory y compact no activan
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. Protección contra colapso de contexto: marble_origami (ctx-agent) no activa
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. Verificación de configuración: ¿ha habilitado el usuario?
  if (!isAutoCompactEnabled()) return false

  // 4. Modo reactivo: si está habilitado, suprime la compresión proactiva
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. Modo de colapso de contexto: el colapso ES la gestión de contexto,
  //    la compresión no debe interferir
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Conteo de tokens + comparación con umbral
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

Este árbol de decisión expone múltiples estrategias de gestión de contexto que Claude Code está experimentando en paralelo:
- **Reactive Compact** (`tengu_cobalt_raccoon`): no comprime proactivamente, espera a que la API reporte prompt_too_long
- **Context Collapse** (`CONTEXT_COLLAPSE`): gestiona el contexto en modo streaming con 90% de commit / 95% de bloqueo
- **Auto Compact** (default actual): comprime proactivamente al alcanzar el umbral

Los tres son mutuamente excluyentes, controlados por feature flags.

### 8.5.3 Mecanismo del Circuit Breaker

```typescript
// autoCompact.ts:219-272 — autoCompactIfNeeded con circuit breaker
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // Verificación del circuit breaker
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // Estado disparado, omitir directamente
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // Intentar primero la compresión de Session Memory
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // Compresión tradicional
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

### 8.5.4 Flujo de Ejecución de autoCompactIfNeeded

Secuencia de ejecución completa:

1. **Verificación de variable de entorno**: `DISABLE_COMPACT` → desactivación global
2. **Verificación del circuit breaker**: `consecutiveFailures >= 3` → omitir
3. **Verificación de umbral**: `shouldAutoCompact()` → múltiples comprobaciones
4. **Compresión de Session Memory** (ruta prioritaria): utiliza session memory existente en lugar de una llamada a la API
5. **Compresión Fork Agent tradicional** (ruta de respaldo): generación completa de resumen impulsada por API
6. **Manejo de fallos**: incrementa el contador del circuit breaker, lo pasa a la siguiente ronda

## 8.6 Capa 3: Compresión Tradicional (Full Compact)

### 8.6.1 Mecanismo Fork Agent

El núcleo de la compresión tradicional es la generación de resúmenes de conversación mediante Fork Agent. La función `streamCompactSummary()` (`compact.ts:1136-1396`) implementa una estrategia de dos niveles de respaldo:

**Primer nivel: Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — Fork con caché compartido
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

Fork Agent reutiliza el prompt cache completo de la conversación principal (system prompt + tools + context messages), solo añade una solicitud de resumen. Decisiones de diseño clave:

1. `maxTurns: 1` — no se permiten interacciones de múltiples rondas
2. `canUseTool: createCompactCanUseTool()` — rechaza todas las llamadas a herramientas
3. `skipCacheWrite: true` — no escribe en caché (bifurcación temporal)
4. **No se establece maxOutputTokens** — el comentario explica: establecerlo cambiaría la configuración de thinking, causando discordancia en la cache key

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**Segundo nivel: Streaming Fallback**

Cuando Fork Agent falla, se recurre a una llamada directa de API en streaming, donde **sí** se puede establecer `maxOutputTokensOverride`:

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

El streaming fallback también soporta reintentos impulsados por configuración (`tengu_compact_streaming_retry`), con un máximo de `MAX_COMPACT_STREAMING_RETRIES = 2` reintentos.

### 8.6.2 Tubería de Preprocesamiento

Los mensajes antes de la compresión pasan por un preprocesamiento de tres pasos:

```typescript
// compact.ts:1293-1300 — Cadena de preprocesamiento
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

1. `getMessagesAfterCompactBoundary` — toma solo los mensajes posteriores a la última compresión
2. `stripReinjectedAttachments` — elimina adjuntos `skill_discovery` / `skill_listing` (se reinyectarán después de la compresión)
3. `stripImagesFromMessages` — reemplaza bloques de imagen con la etiqueta de texto `[image]` (`compact.ts:144-199`)

La razón de existir de `stripImagesFromMessages` es práctica:

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

Los usuarios de CCD (Claude Code Desktop) frecuentemente adjuntan capturas de pantalla; si no se eliminan las imágenes, la propia llamada a la API de compresión puede fallar por prompt demasiado largo.

### 8.6.3 Formato de 9 Capítulos de la Salida del Resumen

`prompt.ts` define los 9 capítulos estructurados que debe seguir el resumen:

```
1. Primary Request and Intent    — Intención del usuario
2. Key Technical Concepts        — Conceptos técnicos
3. Files and Code Sections       — Archivos y fragmentos de código
4. Errors and fixes              — Errores y correcciones
5. Problem Solving               — Resolución de problemas
6. All user messages             — Todos los mensajes del usuario (no tool result)
7. Pending Tasks                 — Tareas pendientes
8. Current Work                  — Trabajo actual
9. Optional Next Step            — Siguiente paso (opcional)
```

El diseño del capítulo 6 es especialmente importante — "List ALL user messages that are not tool results". Esto garantiza que, incluso si la conversación se comprime, las expresiones originales del usuario se conserven íntegramente. Es la garantía de **pérdida cero de información de retroalimentación del usuario**.

El capítulo 9 tiene una restricción cuidadosamente diseñada:

```
// prompt.ts — Restricción del capítulo 9
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

Esto evita que el modelo "actúe por su cuenta" después de la compresión — solo se registran los próximos pasos que el usuario ha solicitado explícitamente.

### 8.6.4 Diseño Anti-Elusión NO_TOOLS_PREAMBLE

Fork Agent hereda el conjunto completo de herramientas de la conversación principal (para coincidir con la cache key), pero el agente de compresión no debería usar ninguna herramienta. Esto crea una contradicción: las herramientas existen en el schema, pero no deberían ser invocadas.

La solución es un **rechazo de herramientas de tres capas**:

```typescript
// prompt.ts:16-24 — Primera capa: declaración fuerte al inicio del prompt
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — Segunda capa: recordatorio repetido al final del prompt
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — Tercera capa: rechazo a nivel de código
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

Los comentarios revelan la razón real de este diseño de tres capas:

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

En Sonnet 4.6, solo con instrucciones de prompt hay un 2.79% de probabilidad de que aún intente llamadas a herramientas (en 4.5 es solo el 0.01%). `createCompactCanUseTool` es la última protección a nivel de código.

### 8.6.5 Postprocesamiento (formatCompactSummary)

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // Eliminar el borrador <analysis> — razonamiento intermedio para mejorar
  // la calidad del resumen, no necesita conservarse
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // Extraer contenido de <summary>
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

El diseño de la etiqueta `<analysis>` es una técnica de Chain-of-Thought: permite al modelo "hacer un borrador" primero en el área de análisis, luego producir el resultado final en `<summary>`. La existencia del área de análisis mejora la calidad del resumen, pero se elimina en la salida final — porque contiene razonamiento intermedio redundante que desperdiciaría espacio de contexto en rondas posteriores.

### 8.6.6 Secuencia de Mensajes Post-Compresión y Reinyección de Adjuntos

Tras completarse la compresión, la nueva secuencia de mensajes se construye con `buildPostCompactMessages()`:

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // Mensaje de sistema: marca el límite de compresión
    ...result.summaryMessages,    // Mensaje de usuario: contenido del resumen
    ...(result.messagesToKeep ?? []),  // Mensajes originales conservados
    ...result.attachments,        // Adjuntos de archivos + skills + planes
    ...result.hookResults,        // Resultados del hook SessionStart
  ]
}
```

La reinyección de adjuntos es un proceso complejo (`compact.ts:532-585`), que incluye:

1. **Adjuntos de archivos**: los 5 archivos accedidos más recientemente, con un presupuesto de 50K tokens, máximo 5K tokens por archivo
2. **Archivos de plan**: si hay un plan activo
3. **Instrucciones de modo plan**: si se está en plan mode
4. **Contenido de skills**: contenido de skills invocados, ordenados por uso más reciente, máximo 5K tokens cada uno, presupuesto total de 25K tokens
5. **Deferred Tools Delta**: re-declara el schema de herramientas de carga diferida
6. **Agent Listing Delta**: re-declara la lista de agentes
7. **MCP Instructions Delta**: re-declara las instrucciones del servidor MCP

### 8.6.7 Mecanismo de Reintento PTL (Prompt-Too-Long Recovery)

Cuando la propia llamada a la API de compresión falla por prompt demasiado largo, el sistema reintenta con truncado progresivo:

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // Primero limpiar los mensajes marcadores dejados por reintentos anteriores
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // Truncado preciso: basado en el diferencial de tokens devuelto por la API
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // Truncado aproximado: descartar 20% de los grupos de mensajes
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // Conservar al menos un grupo
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

El límite de reintentos es `MAX_PTL_RETRIES = 3`. La estrategia de truncado tiene dos caminos:
- **Ruta precisa**: el error de la API contiene el diferencial de tokens → descartar grupos según el diferencial
- **Ruta aproximada** (Vertex/Bedrock y otros formatos de error no estándar): descartar 20% en cada intento

Nótese el manejo del límite en la línea 283: al descartar el grupo 0, la secuencia de mensajes podría comenzar con un mensaje assistant, violando la restricción de la API (el primer mensaje debe ser user). El sistema inserta un mensaje de usuario marcador sintético para corregirlo.

### 8.6.8 Dos Direcciones de la Compresión Parcial (Partial Compact)

`partialCompactConversation()` (`compact.ts:772-1106`) soporta dos direcciones:

```
Direction 'from': 
  [mensajes conservados tras compresión] | pivot | [mensajes resumidos]
  → Conserva el prompt cache (los conservados van primero, el prefijo de caché no cambia)

Direction 'up_to':
  [mensajes resumidos] | pivot | [mensajes conservados tras compresión]
  → El prompt cache se invalida (el resumen va primero, el prefijo cambia)
```

La dirección `up_to` tiene una lógica de limpieza adicional — se deben eliminar el compact boundary y el resumen antiguos de los mensajes conservados:

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

El comentario explica la razón: en el modo `up_to`, el resumen está antes que los mensajes conservados, y el boundary antiguo induciría a error el escaneo inverso de `findLastCompactBoundaryIndex`.

## 8.7 Capa 4: Compresión de Session Memory

### 8.7.1 Idea Central y Ventajas

La compresión de Session Memory (`sessionMemoryCompact.ts`) es una alternativa optimizada a la compresión tradicional. La idea central: utilizar la session memory (resumen incremental de la conversación) mantenida continuamente en segundo plano en lugar del resumen generado en tiempo real por Fork Agent.

Ventajas:
- **Cero llamadas adicionales a la API**: la session memory es mantenida continuamente por un agente en segundo plano, usada directamente al comprimir
- **Menor latencia**: no es necesario esperar 5-15 segundos de respuesta de la API
- **Retención más granular**: se puede calcular con precisión cuántos mensajes recientes conservar

### 8.7.2 Descripción Detallada del Algoritmo calculateMessagesToKeepIndex

Este es el algoritmo central de la compresión de Session Memory (`sessionMemoryCompact.ts:262-327`), que determina cuántos mensajes conservar después de la compresión:

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // Empezar desde lastSummarizedIndex + 1 (la session memory ya cubre el contenido anterior)
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // Calcular tokens y conteo de mensajes con texto en el rango actual de retención
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // Ya se alcanzó el límite máximo → no expandir
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Ya se cumplen los dos requisitos mínimos → no expandir
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Expandir hacia atrás, pero sin cruzar el último compact boundary
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

Parámetros de configuración (pueden sobreescribirse mediante configuración remota de GrowthBook):

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // Conservar mínimo 10K tokens
  minTextBlockMessages: 5,     // Conservar mínimo 5 mensajes con texto
  maxTokens: 40_000,           // Conservar máximo 40K tokens
}
```

El diseño de doble restricción del algoritmo (`minTokens` AND `minTextBlockMessages`) garantiza:
- No detenerse de expandir por un número pequeño de mensajes muy grandes (tokens suficientes pero muy pocos mensajes)
- No conservar demasiados mensajes pequeños pero con tokens insuficientes

**Mecanismo Floor**: al expandir hacia atrás no se puede cruzar el último compact boundary (`floor = lastBoundaryIndex + 1`). El comentario explica la razón:

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

La cadena de mensajes en la capa de almacenamiento en disco tiene una discontinuidad en el compact boundary; cruzarla causaría que el recorrido inverso del cargador saltara los mensajes conservados.

### 8.7.3 Corrección de Bugs en adjustIndexToPreserveAPIInvariants

Esta función (`sessionMemoryCompact.ts:172-260`) es el fragmento de código más sofisticado de todo el sistema de compresión, y resuelve dos problemas de invariantes de la API:

**Escenario de bug 1: tool_result huérfano**

```
Índice N:   assistant, message.id: X, contenido: [thinking]
Índice N+1: assistant, message.id: X, contenido: [tool_use: ORPHAN_ID]
Índice N+2: assistant, message.id: X, contenido: [tool_use: VALID_ID]
Índice N+3: user, contenido: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

Si startIndex = N+2:
  Código antiguo solo verifica tool_results del mensaje N+2 → no encuentra → devuelve N+2
  Después de la fusión por message.id en normalizeMessagesForAPI:
    msg[1]: assistant con [tool_use: VALID_ID]  (¡tool_use ORPHAN excluido!)
    msg[2]: user con [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → Error de API: tool_result huérfano referencia un tool_use inexistente
```

**Escenario de bug 2: bloque thinking perdido**

```
Índice N:   assistant, message.id: X, contenido: [thinking]
Índice N+1: assistant, message.id: X, contenido: [tool_use: ID]
Índice N+2: user, contenido: [tool_result: ID]

Si startIndex = N+1:
  El bloque thinking en N queda excluido
  normalizeMessagesForAPI no puede fusionar (sin mensaje con el mismo ID para fusionar)
  → El bloque thinking se pierde permanentemente
```

El código de corrección realiza dos ajustes:

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Paso 1: Manejar el emparejamiento tool_use/tool_result
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... Recopilar IDs de tool_use existentes en el rango conservado
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // Buscar hacia atrás los tool_use faltantes
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // Eliminar los IDs ya encontrados
      }
    }
  }

  // Paso 2: Manejar bloques thinking con message.id compartido
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

La intuición clave de este código es: la respuesta en streaming de la API de Claude divide una misma respuesta de la API en múltiples mensajes assistant (comparten `message.id`, pero con UUIDs distintos), donde los bloques thinking y tool_use están separados. `normalizeMessagesForAPI` fusiona estos mensajes por `message.id` — si la compresión corta un grupo de mensajes con el mismo ID, la fusión resultante será inconsistente.

### 8.7.4 Flujo Completo de trySessionMemoryCompaction

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. Verificación de acceso
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. Inicializar configuración remota (solo la primera vez)
  await initSessionMemoryCompactConfig()

  // 3. Esperar que finalice la extracción de session memory en curso
  await waitForSessionMemoryExtraction()

  // 4. Obtener contenido de la session memory
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. Determinar el límite
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID no existe → retroceder
  } else {
    // Sesión reanudada: sin límite → empezar desde el final
    lastSummarizedIndex = messages.length - 1
  }

  // 6. Calcular el rango de retención
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // Filtrar boundaries antiguos

  // 7. Ejecutar hooks de session start
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. Construir el resultado
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. Verificación de umbral (solo para autocompact)
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // Aún por encima del umbral tras la compresión → retroceder a compresión tradicional
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 Parámetros de Configuración (Configuración Remota GrowthBook)

Todos los parámetros clave de la compresión de Session Memory pueden sobreescribirse mediante configuración remota de GrowthBook:

```typescript
// sessionMemoryCompact.ts:91-109 — Inicialización de configuración remota
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // Programación defensiva: usar solo valores positivos, ignorar valores 0
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

La habilitación del feature está controlada por dos feature flags independientes:

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

## 8.8 Colapso de Contexto y Almacenamiento de Resultados de Herramientas

### 8.8.1 Mecanismo collapseReadSearch

`utils/collapseReadSearch.ts` (1,109 líneas) implementa el plegado de mensajes en la capa de UI — plegando operaciones consecutivas de búsqueda/lectura en un resumen de una línea (como "Read 5 files, searched 3 patterns").

La lógica de clasificación central (`getToolSearchOrReadInfo`, `collapseReadSearch.ts:142-237`) divide las llamadas a herramientas en:

| Categoría | Plegable | Comportamiento de plegado |
|------|-------|---------|
| Read (file_path) | Sí | "Read N files" |
| Search (Grep/Glob) | Sí | "Searched N patterns" |
| Shell (Bash) | Sí en modo pantalla completa | "Ran N bash commands" |
| REPL | Sí (absorción silenciosa) | Conteo independiente de herramientas internas |
| Memory Write | Sí | Marcador especial |
| ToolSearch | Sí (absorción silenciosa) | No incrementa contador |
| Edit/Write (no memory) | No | Interrumpe el grupo de plegado |

"Absorción silenciosa" (`isAbsorbedSilently`) es un diseño sutil: REPL y ToolSearch no incrementan el contador pero tampoco interrumpen el grupo de plegado actual. Esto significa que `[Read, ToolSearch, Read]` se pliega como "Read 2 files" en lugar de ser cortado en dos grupos por ToolSearch.

El plegado es una optimización **solo de la capa UI** — no cambia el contenido de los mensajes enviados a la API, solo afecta la visualización en el terminal.

### 8.8.2 Estrategia de Almacenamiento en Disco de toolResultStorage

`utils/toolResultStorage.ts` (1,040 líneas) es la "capa cero" de la gestión de contexto — maneja los resultados sobredimensionados antes de que entren en el historial de conversación.

**Análisis del umbral de persistencia**:

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // Excepción para la herramienta Read: Infinity → no persistir (Read tiene su propio límite maxTokens)
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // Sobreescritura de GrowthBook (tengu_satin_quoll)
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // Por defecto: min(valor declarado por la herramienta, valor por defecto global 50K)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**Optimización de deduplicación**: `tool_use_id` es único, usando `flag: 'wx'` (exclusive write) para evitar escrituras duplicadas:

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: ya persistido en rondas anteriores, omitir
}
```

**Manejo de resultados vacíos**:

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

Esta corrección resuelve un bug de comportamiento del modelo: un tool_result vacío causaba que ciertos modelos coincidieran el patrón `\n\nHuman:` como fin de conversación.

**Per-Message Aggregate Budget** (`enforceToolResultBudget`, `toolResultStorage.ts:769-908`):

Esta es la funcionalidad más compleja de `toolResultStorage.ts` — aplica un presupuesto total de tamaño de resultados de herramientas por mensaje de usuario a nivel de API (tras la fusión de `normalizeMessagesForAPI`).

Puntos clave de diseño:
- **Congelación de estado** (`ContentReplacementState`): una vez que un tool_result es "visto" (enviado al modelo), su decisión (reemplazar/no reemplazar) se congela y nunca cambia — esto garantiza la estabilidad del prompt cache
- **Estrategia de tres particiones**: `mustReapply` (reemplazado anteriormente → reaplicar el reemplazo en caché), `frozen` (visto anteriormente pero no reemplazado → no tocar), `fresh` (nuevo → posiblemente reemplazado)
- **Máximo primero**: cuando es necesario reemplazar, se selecciona primero el resultado fresh más grande para reemplazar

## 8.9 El Rol de la Compresión en la Recuperación de Errores de 5 Capas

### 8.9.1 Mecanismo Completo de Recuperación de Errores de 5 Capas

El sistema de compresión juega múltiples roles en el mecanismo de recuperación de errores de Claude Code:

| Capa | Condición de activación | Comportamiento de compresión | Origen |
|------|---------|---------|------|
| L1 | API devuelve prompt_too_long (413) | Reactive Compact: truncar + re-resumir | `compactMessages.ts` |
| L2 | La propia API de compresión devuelve 413 | PTL Retry: truncar grupos de mensajes más antiguos × 3 veces | `compact.ts:truncateHeadForPTLRetry` |
| L3 | Aún por encima del umbral tras compresión | Re-compaction: comprimir automáticamente de nuevo | `autoCompact.ts:recompactionInfo` |
| L4 | 3 fallos consecutivos de compresión | Circuit Breaker: dejar de intentar | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent sin salida de texto | Streaming Fallback: llamada directa a la API en streaming | `compact.ts:streamCompactSummary` |

### 8.9.2 Compresión Reactiva vs. Compresión Proactiva

El equilibrio entre las dos estrategias:

**Compresión Proactiva** (Auto-Compact, default actual):
- Se activa proactivamente cuando los tokens alcanzan el umbral
- Ventajas: experiencia de usuario más fluida, no se encuentran errores 413
- Desventajas: puede comprimir demasiado pronto, desperdiciando contexto disponible

**Compresión Reactiva** (Reactive Compact, experimento `tengu_cobalt_raccoon`):
- Espera a que la API reporte prompt_too_long para activarse
- Ventajas: maximiza la utilización del contexto
- Desventajas: la experiencia de usuario tiene interrupciones obvias, requiere esperar reintentos

En el código se puede ver la relación de exclusión mutua entre ambas:

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // En modo reactivo, no comprimir proactivamente
  }
}
```

## 8.10 Agrupación de Mensajes y Estimación de Tokens

### 8.10.1 Algoritmo groupMessagesByApiRound

`grouping.ts` (63 líneas) agrupa los mensajes por ronda de API — cada grupo corresponde a un viaje de ida y vuelta completo de la API:

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

El único criterio para el límite de agrupación es el cambio en `message.id` — múltiples bloques en streaming de la misma respuesta de la API comparten el mismo `message.id`, por lo que naturalmente caen en el mismo grupo.

Este diseño reemplazó la agrupación anterior basada en "turnos humanos" (agrupando solo en los mensajes reales del usuario), que no podía manejar sesiones de agente de turno único prolongadas en escenarios SDK/CCR/eval.

### 8.10.2 roughTokenCountEstimation y Relleno Conservador

La estimación de tokens adopta una estrategia conservadora de dos niveles:

**Primer nivel**: estimación básica

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

Por defecto 4 bytes/token; los archivos JSON usan 2 bytes/token (porque JSON tiene muchos tokens de un solo carácter como `{`, `}`, `:`, `,`).

**Segundo nivel**: relleno a nivel de mensaje

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

Efecto combinado: para texto ordinario, la estimación efectiva es `text.length / 4 * 4/3 = text.length / 3`.

### 8.10.3 Estrategia Híbrida de Precisión vs. Estimación

El sistema usa diferentes precisiones en distintos escenarios:

| Escenario | Precisión | Origen | Latencia |
|------|------|------|------|
| shouldAutoCompact | Híbrido: prioriza valor preciso devuelto por la API | `tokenCountWithEstimation` | 0 (ya en caché) |
| estimateMessageTokens | Estimación aproximada (`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | Estimación aproximada | `estimateMessageTokens` | 0 |
| Estadísticas de tokens tras compresión | Preciso | `tokenCountFromLastAPIResponse` | 0 (ya devuelto por la API) |

La estrategia híbrida de `tokenCountWithEstimation` es: priorizar `usage.input_tokens` devuelto por la última llamada a la API (valor preciso); si no está disponible (como antes de la primera solicitud), recaer en la estimación.

## 8.11 Análisis de Decisiones de Diseño

### 8.11.1 Filosofía de Degradación Progresiva

La gestión de contexto de Claude Code adopta una degradación progresiva **sin saltar niveles**: cada capa intenta resolver el problema con el mínimo costo, y solo si falla la capa actual se sube a la siguiente. Esto evita el problema común de "reacción excesiva" — por ejemplo, activar una compresión completa simplemente porque el resultado de un Read de un archivo grande es demasiado grande.

Comparación con la práctica del sector:
- **ChatGPT**: truncar mensajes antiguos (simple pero brusco)
- **GitHub Copilot Chat**: ventana de contexto fija + últimos N mensajes (sin compresión)
- **Claude Code**: 5 niveles progresivos (prevención → microajuste → resumen → recuperación de emergencia)

### 8.11.2 Diseño Caché-Primero

El prompt cache es el sustento de vida de Claude Code — una solicitud de 200K tokens, si 180K son cache read ($0.30/M tokens), el costo es 10 veces menor que un total cache miss ($3/M tokens). Casi todas las decisiones de diseño sirven a esta restricción económica:

1. **Fork Agent comparte el prefijo de caché**: la llamada a la API de compresión reutiliza la caché de la conversación principal
2. **No establecer maxOutputTokens en Fork**: evita discordancias en la configuración de thinking que invaliden la caché
3. **Cached MC no modifica mensajes locales**: mantiene el prefijo del prompt sin cambios
4. **ContentReplacementState congela IDs ya vistos**: garantiza que la decisión de reemplazo para el mismo tool_result no cambie durante su ciclo de vida
5. **sentSkillNames no se reinicia**: evita reinyectar ~4K tokens de skill_listing
6. **pinnedCacheEdits se reenvía en posición fija**: garantiza la consistencia de la posición de la cache edit

### 8.11.3 Garantías de Seguridad

El sistema mantiene tres clases de invariantes:

**Emparejamiento no debe cortarse**: `adjustIndexToPreserveAPIInvariants` garantiza que tool_use y tool_result nunca queden en lados distintos. Esto no es solo un requisito de corrección funcional (la API reportaría error), sino también un requisito de corrección semántica (el modelo necesita ver el resultado de la herramienta que invocó anteriormente).

**Protección contra recursión**: la verificación de `querySource` en `shouldAutoCompact` garantiza que el agente session_memory, el agente compact y el agente de colapso de contexto no activen la compresión automática — estos agentes son ellos mismos parte de la gestión de contexto, y la compresión recursiva causaría un deadlock.

**Mecanismo Circuit Breaker**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` se estableció basándose en datos reales (bucle de fallos de 1,279 sesiones), cambiando el reintento infinito por un reintento limitado + disparo.

### 8.11.4 Comparación con la Gestión de Contexto API-Native

`apiMicrocompact.ts` revela que Claude Code está explorando la dirección de descargar parte de la gestión de contexto a la capa de API:

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

Estas estrategias `context_management.edits` se declaran directamente en la solicitud de API y son ejecutadas por el servidor. Las ventajas son menor latencia (sin procesamiento del cliente) y alineación precisa con el conteo de tokens del servidor. Actualmente, la estrategia de limpieza de herramientas solo está disponible para usuarios internos (`USER_TYPE === 'ant'`); los usuarios externos solo usan la limpieza de thinking.

## 8.12 Patrones Transferibles

### 8.12.1 Patrones de Diseño Generales del Sistema Multicapa de Compresión

La gestión de contexto de Claude Code destila los siguientes patrones generales transferibles:

**Patrón 1: Desalojo por Niveles (Tiered Eviction)**
- Aplicar diferentes estrategias de desalojo a diferentes tipos de contenido
- El contenido regenerable (salida de herramientas) se desaloja primero, el contenido no regenerable (entrada del usuario) se desaloja al final
- Implementación: lista blanca + ordenación por prioridad

**Patrón 2: Estimación-Precisión Híbrida (Hybrid Estimation)**
- Las decisiones rápidas usan estimación aproximada (`text.length / 3`), la contabilidad precisa usa los valores devueltos por la API
- La estimación aproximada siempre es conservadora (mejor sobreestimar y comprimir antes que subestimar y causar errores de la API)

**Patrón 3: Congelar-Reproducir (Freeze-Replay)**
- Una vez que el contenido es "visto" por el modelo, su decisión de procesamiento se congela
- Las rondas posteriores para contenido congelado solo hacen "reproducción" (re-apply cached replacement), sin nuevas decisiones
- Garantiza la estabilidad a nivel de bits del prefijo del prompt → acierto de caché

**Patrón 4: Truncado con Conciencia de Límites (Boundary-Aware Truncation)**
- Nunca truncar a mitad de una unidad semántica (par tool_use/tool_result, grupo de mensajes con el mismo ID)
- Reparar activamente tras el truncado (insertar mensajes sintéticos, ajustar índices)

**Patrón 5: Protección Circuit Breaker (Circuit Breaker Protection)**
- Establecer un contador de fallos para operaciones que podrían reintentar indefinidamente
- Establecer umbrales basados en datos operacionales reales (no en intuición)

### 8.12.2 Lecciones para Doramagic

En la tubería Soul Extractor de Doramagic, el proceso de extracción puede generar grandes cantidades de resultados intermedios (fragmentos de código, documentación de API, discusiones comunitarias). Patrones aplicables:

1. **Caché de extracción por capas**: similar al mecanismo de lista blanca de microcompact, clasificar las respuestas intermedias de la API y los resultados del análisis de código por regenerabilidad, priorizando el desalojo del contenido que se puede volver a obtener
2. **Resumen incremental**: similar a Session Memory Compact, mantener un resumen incremental del conocimiento extraído en lugar del historial completo
3. **Decisiones congeladas**: una vez que un bloque de conocimiento se confirma como "valioso" o "sin valor", la decisión es irreversible — evitar reevaluar repetidamente entre diferentes rondas de extracción

## 8.13 Índice de Código Fuente

| Archivo | Líneas | Responsabilidad principal |
|------|------|---------|
| `services/compact/compact.ts` | ~1,705 | Lógica principal de compresión tradicional: Fork Agent, reintento PTL, reinyección de adjuntos, compresión parcial |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Compresión de Session Memory: calculateMessagesToKeepIndex, adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | Microcompresión: activación por tiempo, edición de caché, estimación de tokens |
| `services/compact/prompt.ts` | ~374 | Prompt de compresión: plantilla de 9 capítulos, NO_TOOLS_PREAMBLE, formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | Compresión automática: cálculo de umbral, cadena de decisión shouldAutoCompact, circuit breaker |
| `services/compact/apiMicrocompact.ts` | ~153 | Gestión de contexto API-native: clear_tool_uses, clear_thinking |
| `services/compact/grouping.ts` | ~63 | Agrupación de mensajes: groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | Limpieza post-compresión: reinicio de caché, estado de módulos, clasificadores |
| `services/compact/timeBasedMCConfig.ts` | ~43 | Configuración de activación por tiempo: configuración remota de GrowthBook |
| `services/compact/compactWarningHook.ts` | ~16 | React hook: suscripción al estado de supresión de avisos de compact |
| `services/compact/compactWarningState.ts` | ~18 | Almacenamiento de estado: indicador de supresión de avisos de compact |
| `services/cost-tracker.ts` | ~323 | Seguimiento de costos: facturación de tokens, estadísticas de uso del modelo |
| `utils/collapseReadSearch.ts` | ~1,109 | Colapso de contexto: agrupación y plegado de mensajes en la capa UI |
| `utils/toolResultStorage.ts` | ~1,040 | Almacenamiento de resultados de herramientas: persistencia en disco, presupuesto por mensaje, ContentReplacementState |
| `services/tokenEstimation.ts` | ~350+ | Estimación de tokens: roughTokenCountEstimation (text.length/4) |
