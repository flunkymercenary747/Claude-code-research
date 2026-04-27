# Capítulo 2: Query Engine — Núcleo de Interacción LLM

## 2.1 Descripción General y Posicionamiento

**Posicionamiento en una oración:** El Query Engine es el "latido" de Claude Code: posee el ciclo de vida completo de la conversación con el LLM, transformando la entrada del usuario en interacciones agentic de múltiples rondas con control de permisos, ejecución de herramientas, compresión de contexto y seguimiento de costos.

**El problema central que resuelve:** Cómo orquestar de manera fiable en un pipeline de async generator el bucle "respuesta streaming del LLM → llamada a herramienta → inyección de resultado → nueva solicitud", manejando al mismo tiempo al menos 12 ramas de excepción: desbordamiento de ventana de contexto, presupuesto de tokens, fallos de API, interrupciones del usuario, aprobaciones de permisos, etc.

**Estadísticas de archivos involucrados y cantidad de código:**

| Archivo | Líneas | Responsabilidad |
|------|------|------|
| `QueryEngine.ts` | 1,295 | Gestión de estado a nivel de sesión, punto de entrada SDK/headless |
| `query.ts` | 1,729 | Bucle principal de consulta, orquestación de ejecución de herramientas, recuperación multinivel |
| `query/config.ts` | 46 | Instantánea de configuración inmutable |
| `query/tokenBudget.ts` | 93 | Decisión de auto-continue del presupuesto de tokens |
| `query/stopHooks.ts` | 473 | Hooks al detener (puertas de seguridad, post-procesamiento) |
| `query/deps.ts` | 40 | Interfaz de inyección de dependencias (amigable para pruebas) |
| `services/api/claude.ts` | 3,419 | Llamadas API, análisis streaming, reintentos, estrategias de caché |
| `cost-tracker.ts` | 323 | Seguimiento de costos y acumulación de uso |
| **Total** | **~7,418** | — |

Estas 7,400+ líneas de código constituyen aproximadamente el 1.4% del código de Claude Code, pero son la ruta más crítica de todo el producto: cada interacción entre el usuario y Claude debe pasar por aquí.

---

## 2.2 Fundamentos Teóricos

### 2.2.1 Async Generator Pipeline (Tubería de Corrutinas)

La arquitectura central del Query Engine se basa en **ES2018 Async Generator** como primitiva de procesamiento streaming. La firma de la función `query()` revela este diseño:

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

Esta no es una elección de sintaxis casual, sino una aplicación precisa de la **teoría de Corrutinas**. Un async generator tiene simultáneamente:
- **Evaluación perezosa**: impulsada por el consumidor, no precalcula todas las respuestas del API
- **Comunicación bidireccional**: yield para eventos de stream, return para la razón terminal
- **Seguridad de recursos**: el bloque finally garantiza la liberación del stream (`releaseStreamResources()` en `claude.ts`)

Los callbacks tradicionales o cadenas de Promise no pueden satisfacer simultáneamente "salida streaming para la UI" y "esperar los resultados de ejecución de herramientas". Los async generators soportan de forma natural la semántica de "el productor pausa esperando que el consumidor consuma".

### 2.2.2 State Machine (Máquina de Estados Implícita)

El bucle principal de `query.ts` no es una llamada recursiva (aunque los comentarios del código antiguo aún usan `query_recursive_call`), sino una **máquina de estados explícita impulsada por while(true) + continue**:

```typescript
// query.ts:218 (definición del tipo State)
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

En cada `continue`, el State completo se reconstruye (actualización inmutable), y el campo `transition` registra la razón de la transición. Esta es la clásica **Máquina de Mealy**: la salida depende de la combinación del estado actual y el evento de entrada.

### 2.2.3 Backpressure y Gestión de Recursos

El backpressure es crucial en el streaming de LLM. La solución de Claude Code:

1. **for-await-of como limitador natural**: el consumo del stream en `claude.ts` usa `for await (const part of stream)`, la velocidad de procesamiento del consumidor determina directamente la tasa de envío del productor
2. **Stream idle watchdog** (`claude.ts:2397`): si no se recibe ningún chunk en 90 segundos, se aborta activamente el stream y se hace fallback a non-streaming
3. **Garantía del ciclo de vida del generator**: el bloque `finally` garantiza que `releaseStreamResources()` se ejecute en todas las rutas de salida (incluyendo `.return()` y excepciones)

### 2.2.4 Por Qué Estas Teorías Son Especialmente Importantes en Escenarios LLM

Las llamadas a API HTTP tradicionales son modelos "solicitud-respuesta" con manejo de errores simple. El bucle agentic de LLM enfrenta desafíos únicos:

- **Una única llamada puede durar 10 minutos** (límite de non-streaming)
- **La respuesta puede activar nuevo I/O a mitad** (llamadas a herramientas)
- **La ventana de contexto es un recurso escaso con estado**, que requiere equilibrar "pérdida de información por compresión" y "colapso por desbordamiento"
- **Los costos se acumulan en tiempo real**, necesitando ser interrumpibles en cualquier momento

Estas restricciones hacen del Event Loop + máquina de estados + Backpressure un soporte teórico indispensable.

---

## 2.3 Arquitectura y Estructuras de Datos

### 2.3.1 Interfaz Central de la Clase QueryEngine

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

**Puntos clave de diseño:** QueryEngine es por conversación. Cada llamada a `submitMessage()` es un nuevo turno en la misma conversación, y el estado se mantiene entre turnos.

### 2.3.2 Definiciones de Tipos Clave

**QueryEngineConfig** (`QueryEngine.ts:95-153`) — configuración inmutable pasada en la construcción:

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

**QueryConfig** (`query/config.ts:18-31`) — instantánea de entorno inmutable por consulta:

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

Nótese la intención de diseño en el comentario del código fuente (`config.ts:9-12`): "Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." Esta es la demarcación clara entre la optimización de compilación de bun:bundle y la configuración en tiempo de ejecución.

### 2.3.3 Diagrama de Dependencias entre Módulos

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- gestión de estado de sesión
                    |  (1,295 líneas) |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- bucle principal & máquina de estados
                    |  (1,729 líneas) |  <-- State, bucle while(true)
                    +--------+--------+
                             |
              +--------------+------------------+
              |              |                  |
              v              v                  v
    +---------+---+  +-------+--------+  +------+-------+
    | query/      |  | query/         |  | query/       |
    | config.ts   |  | tokenBudget.ts |  | stopHooks.ts |
    | (46 líneas) |  | (93 líneas)    |  | (473 líneas) |
    +-------------+  +----------------+  +--------------+
              |              |
              v              v
    +---------+--------------+--------+
    |         query/deps.ts           |
    |   QueryDeps (interfaz DI)       |
    +---------+-----------------------+
              |
              |  callModel / autocompact / microcompact
              v
    +---------+-----------+     +----------+--------+
    | services/api/       |     | cost-tracker.ts   |
    | claude.ts           |     | (323 líneas)      |
    | (3,419 líneas)      |     | addToTotalSession |
    | queryModelWith      |     | Cost, formatCost  |
    | Streaming           |     +-------------------+
    +---------+-----------+
              |
              v
    +---------+-----------+
    | withRetry / stream  |
    | análisis SSE        |
    | Fallback            |
    | non-streaming       |
    +---------------------+
```

**El diseño de inyección de dependencias de deps.ts** (`deps.ts:18-37`) merece especial atención:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" — esta es la mejor práctica de inyección de dependencias en TypeScript: sin necesidad de escribir interfaces manualmente, las firmas siguen automáticamente las implementaciones.

---

## 2.4 Algoritmos Centrales y Flujos

### 2.4.1 Flujo de Ejecución Completo del Bucle Principal de query()

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    entrada query()
                        |
                +=======+========+  <--- inicio del bucle principal while(true)
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        verificación             |
        blocking-limit           |
                |                |
                v                |
        callModel() [streaming]  |
                |                |
        +-------+--------+       |
        | consumo eventos |      |
        | streaming        |     |
        | recopilación     |     |
        | tool_use         |     |
        | streaming exec   |     |
        +-------+--------+       |
                |                |
        needsFollowUp?           |
        /          \             |
      NO           SÍ            |
       |              \          |
       v               v         |
  +---------+   runTools() o    |
  | lógica  |   getRemainingR() |
  | recup.  |         |         |
  +---------+         v         |
       |     getAttachments()   |
       |     precarga memoria   |
       |     descubrim. skill   |
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

### 2.4.2 Bucle de Llamadas a Herramientas (Tool-call Loop)

La innovación central de las llamadas a herramientas es la **streaming tool execution**: comenzar a ejecutar herramientas mientras la respuesta streaming del LLM *todavía está en progreso*:

```typescript
// query.ts:443 (inicialización de StreamingToolExecutor)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

En el bucle de consumo streaming (`query.ts:536`):

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

Las herramientas se envían para ejecución en `content_block_stop`, sin esperar que toda la respuesta del asistente finalice. Esto significa que si el LLM produce 3 bloques tool_use, los primeros dos pueden haber terminado su ejecución mientras el tercero aún está en transmisión.

### 2.4.3 Implementación Concreta del Procesamiento Streaming

El `queryModel()` de `claude.ts` implementa manualmente el análisis del stream SSE, **evitando deliberadamente el BetaMessageStream del SDK de Anthropic**:

```typescript
// claude.ts comentario (aproximadamente línea 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

Modelo de acumulación de estado streaming:

```
message_start → partialMessage = part.message, usage = inicial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → yield AssistantMessage (¡por bloque!)
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (terminación)
```

Diseño clave: **cada content block emite de forma independiente un AssistantMessage**. Esto significa que cuando una respuesta del LLM contiene texto + tool_use, la UI puede mostrar el texto inmediatamente después de completarse, sin esperar a que el JSON de tool_use se complete.

### 2.4.4 Mecanismo de 5 Capas para Reintentos y Recuperación de Errores

La arquitectura de recuperación de errores de Claude Code es defensa en profundidad, con 5 capas:

**Capa 1: withRetry (nivel API)** — `withRetry()` en `claude.ts` maneja errores reintentables como 429 (rate limit), 529 (overload), 5xx, con retroceso exponencial y fallback de modelo.

**Capa 2: Fallback Streaming → Non-streaming** — cuando la conexión streaming se interrumpe (`claude.ts:2592`):

```typescript
// Retroceder al modo non-streaming con reintentos
const result = yield* executeNonStreamingRequest(...)
```

También incluye stream idle watchdog (timeout de 90s sin datos) y fallback de creación de stream 404.

**Capa 3: Recuperación de max_output_tokens** — recuperación progresiva en 3 pasos en `query.ts`:

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// Paso 1: escalar a 64k tokens (ESCALATED_MAX_TOKENS)
// Pasos 2-4: inyectar meta message pidiendo "Resume directly — no apology, no recap"
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**Capa 4: Recuperación de Prompt-too-long** — three-tier en cascada:
1. Context collapse drain (vaciar el colapso almacenado)
2. Reactive compact (compresión completa de emergencia)
3. Mostrar error y salir (evitar bucle infinito)

El código fuente tiene guardias explícitos para prevenir bucles infinitos (comentario de `query.ts`):

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**Capa 5: Model Fallback** — en `query.ts:673-719`, cuando se captura `FallbackTriggeredError`, se cambia al modelo de fallback y se reintenta toda la solicitud.

### 2.4.5 Conteo de Tokens y Gestión de Presupuesto

El sistema de presupuesto de tokens tiene dos mecanismos independientes:

**Mecanismo A: task_budget del API** — presupuesto de tokens percibido por el servidor, rastreado a través de los límites de compaction:

```typescript
// query.ts:270-280 (seguimiento de taskBudgetRemaining a través de compresión)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**Mecanismo B: auto-continue del presupuesto de tokens del lado cliente** (`tokenBudget.ts`) — continúa automáticamente cuando la salida del turno no alcanza el 90% del presupuesto:

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
  // Los sub-agentes no hacen auto-continue
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

Nótese la **detección de rendimientos decrecientes**: cuando hay 3 continuaciones consecutivas con cada incremento < 500 tokens, se detiene automáticamente, evitando que el modelo desperdicie presupuesto en salidas ineficientes.

### 2.4.6 Manejo del Thinking Mode

Los "comentarios mágicos" en el código fuente resumen las 3 reglas de hierro del thinking mode (`query.ts:105-118`):

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

Lógica de construcción de parámetros thinking en `claude.ts` (aproximadamente línea 2242):

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

Los modelos que soportan adaptive thinking lo usan preferentemente (sin necesidad de presupuesto predefinido); de lo contrario, se retrocede a enabled + budget_tokens.

### 2.4.7 Gestión de Límites de Prompt Cache

La estrategia de caché de Claude Code es impresionante. El diseño central está en `addCacheBreakpoints()` (`claude.ts:3045`):

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**Solo se coloca un marcador cache_control**, en el último mensaje (o en el penúltimo cuando `skipCacheWrite`), siendo el resultado de una optimización coordinada con el gestor de páginas KV del equipo de inferencia (Mycro).

El caché con TTL de 1h también tiene un mecanismo fino de "session stability latching" (`claude.ts:380-420`): una vez que la elegibilidad se determina, queda fija para toda la sesión, evitando que los cambios de configuración de GrowthBook a mitad de sesión inviertan el TTL de cache_control y rompan el caché.

---

## 2.5 Análisis de Decisiones de Diseño

### 2.5.1 Tradeoffs Clave

**Tradeoff 1: Ejecución streaming vs. garantías de completitud**

El StreamingToolExecutor comienza a ejecutar herramientas mientras el LLM aún produce salida, lo que conlleva una optimización significativa de latencia pero también introduce complejidad: si el streaming hace fallback a mitad de camino, las herramientas ya ejecutadas necesitan descartarse:

```typescript
// query.ts:534-538 (limpieza en fallback de streaming)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

Este problema ya ha causado bugs (ver comentario en `claude.ts:2575` referenciando inc-4258: double tool execution).

**Tradeoff 2: Estabilidad de caché vs. funciones dinámicas**

Múltiples beta headers usan el patrón "sticky-on latch" (`claude.ts:2102-2126`): una vez activados se mantienen para toda la sesión, incluso si la función se desactiva:

```typescript
// claude.ts:2104 comentario
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

Esta es la compensación explícita de "prioridad a la tasa de aciertos de caché sobre la flexibilidad de funciones".

**Tradeoff 3: Máquina de estados vs. recursión**

El bucle principal evolucionó de llamadas recursivas `query()` tempranas a `while(true)` + reconstrucción de State. Los nombres de checkpoint `query_recursive_call` aún existen en los comentarios del código fuente, pero en realidad ya es iterativo. Las ventajas son:
- Sin riesgo de desbordamiento de pila (las conversaciones largas pueden tener cientos de turnos)
- La reconstrucción de State es explícita, fácil de depurar
- El campo `transition` proporciona una pista de auditoría completa de transiciones de estado

### 2.5.2 Problemas Conocidos Expuestos por Comentarios del Código Fuente

1. **Duplicación de text delta del SDK** (`claude.ts:2350`):

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Conflicto entre fallback non-streaming y streaming tool execution** (`claude.ts:2575`):

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Sesgo de conteo de tokens de task budget a través de compaction** (`query.ts:268`):

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 Diferencias de Diseño con Soluciones como LangChain

| Dimensión | Claude Code Query Engine | LangChain AgentExecutor |
|------|--------------------------|-------------------------|
| Primitiva streaming | ES Async Generator (nativo) | Callback + Stream wrapper |
| Gestión de estado | Struct State explícito + actualización inmutable | Dict AgentState mutable |
| Ejecución de herramientas | Paralela en streaming (StreamingToolExecutor) | await serial |
| Reintentos | 5 capas de defensa en profundidad + model fallback | max_iterations simple |
| Inyección de dependencias | QueryDeps + sincronización de firma typeof | Duck typing en tiempo de ejecución |
| Caché | Coordinación profunda con inference KV cache | Ninguno (llamadas API de caja negra) |

La diferencia más fundamental: Claude Code es **inference-aware**: comprende el mecanismo físico del Prompt Cache (gestor de páginas Mycro) y optimiza en consecuencia, mientras que los frameworks de código abierto solo pueden tratar el API como una caja negra.

---

## 2.6 Patrones Transferibles

### 2.6.1 Patrones de Ingeniería Generales Extraídos del Query Engine

**Patrón 1: Estado Inmutable + Etiqueta de Transición**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

Registrar la **razón** de cada transición de estado en el state convierte la depuración y la telemetría en una preocupación de primer nivel. Cualquier sistema que requiera decisiones en múltiples pasos puede adoptar esto.

**Patrón 2: Inyección de Dependencias Tipada mediante `typeof`**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

Sin necesidad de escribir interfaces manualmente, las firmas se sincronizan automáticamente con las implementaciones. Aplicable a cualquier sistema que necesite simular I/O pesado.

**Patrón 3: Withholding Pattern (Exposición Diferida de Errores)**

Para errores recuperables (prompt-too-long, max_output_tokens), no hacer yield al consumidor inicialmente; decidir si exponerlos después de ejecutar la lógica de recuperación:

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

Esto evita que los consumidores SDK reciban señales de error falsas en casos donde "el error ya se recuperó".

**Patrón 4: Session-stable Latching**

Para elementos de configuración que afectan las claves de caché, bloquearlos una vez activados para toda la sesión:

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // escribir en bootstrap state
}
```

Aplicable a cualquier escenario donde "un cambio de configuración causaría la invalidación de recursos costosos".

### 2.6.2 Lecciones para Doramagic

El pipeline `flow_controller` de Doramagic enfrenta problemas de orquestación similares al Query Engine; se puede aprender de:

1. **Patrón State + Transition**: la máquina de 12 estados de Doramagic puede usar una estructura similar de `{ state, transition: { reason } }`, que simplifica la depuración y soporta naturalmente el registro de auditoría
2. **Dependency Injection via typeof**: la capa de llamadas LLM de Doramagic puede usar el patrón QueryDeps, inyectando modelos falsos durante las pruebas sin simular todo el módulo
3. **Detección de rendimientos decrecientes** (`tokenBudget.ts`): el Soul Extractor de Doramagic puede usar la misma estrategia de "parar si el incremento es inferior al umbral en N iteraciones consecutivas" durante el refinamiento iterativo, evitando que el LLM desperdicie tokens en salidas de baja calidad

---

## 2.7 Índice del Código Fuente

| Archivo | Líneas | Responsabilidad en una oración |
|------|------|----------|
| `QueryEngine.ts` | 1,295 | Propietario a nivel de sesión: mantiene mutableMessages, totalUsage; traduce submitMessage() en llamadas query(); maneja el enrutamiento de mensajes SDK |
| `query.ts` | 1,729 | Máquina de estados del bucle principal: while(true) orquesta compaction → llamada API → ejecución de herramientas → stop hooks → verificación de presupuesto |
| `query/config.ts` | 46 | Instantánea inmutable de QueryConfig: sessionId + 4 runtime gates (los feature() gates se excluyen deliberadamente para preservar el tree-shaking) |
| `query/tokenBudget.ts` | 93 | Auto-continue del presupuesto de tokens del lado cliente: umbral de completitud del 90% + parada anticipada por rendimientos decrecientes |
| `query/stopHooks.ts` | 473 | Orquestación de hooks al finalizar turno: Stop hooks → TaskCompleted hooks → TeammateIdle hooks, con soporte para reinyección de errores de bloqueo |
| `query/deps.ts` | 40 | Interfaz de inyección para 4 dependencias I/O: callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3,419 | Ciclo de vida completo del API: construcción de parámetros → creación de stream → análisis SSE → acumulación de content block → cálculo de costo → fallback non-streaming → gestión de breakpoints de caché |
| `cost-tracker.ts` | 323 | Acumulación de costos a nivel de sesión: seguimiento de uso por modelo, persistencia/recuperación de sesión, formato de salida |
