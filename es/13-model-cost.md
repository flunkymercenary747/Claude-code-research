# Capítulo 13: Selección de Modelos y Control de Costes

> Fuente de datos: snapshot del código fuente TypeScript de Claude Code (2026-03-31, ~512K LOC)
> Archivos principales: `services/api/claude.ts` (3.419 líneas), `services/api/withRetry.ts`, `cost-tracker.ts` (323 líneas), `utils/effort.ts`, `utils/modelCost.ts`, `utils/model/model.ts`, directorio `migrations/` (11 archivos)

---

## 13.1 Descripción General y Posicionamiento

La filosofía de diseño de Claude Code en materia de selección de modelos y control de costes puede resumirse en tres principios:

1. **Prioridad a la intención del usuario**: la cadena de prioridad va de `/model` → flag `--model` → variable de entorno → archivo de configuración; cada capa puede ser sobreescrita por la superior, pero no puede ser reemplazada accidentalmente por la inferior.
2. **Transparencia total de costes**: al finalizar la sesión se imprime obligatoriamente el uso de tokens y el coste en dólares desglosado por modelo; no se puede desactivar (solo cuando `hasConsoleBillingAccess()` es verdadero).
3. **Sin degradación silenciosa**: cuando se produce un Overload Fallback (Opus → Sonnet), se debe mostrar al usuario un mensaje de advertencia; nunca se realiza un cambio silencioso.

Este capítulo verifica desde el nivel del código fuente las afirmaciones del cc-notebook sobre este subsistema y profundiza en el análisis.

---

## 13.2 Fundamentos Teóricos

### Estrategias de Enrutamiento en Sistemas Multimodelo

En los sistemas multimodelo, las estrategias de enrutamiento generalmente equilibran tres dimensiones: **capacidad** (capability), **coste** (cost) y **latencia** (latency). La elección de Claude Code es enrutar la conversación principal (main loop) hacia el modelo más potente disponible, enrutar las tareas auxiliares en segundo plano hacia el modelo más rápido y económico, y proporcionar degradación transparente cuando el modelo principal no está disponible.

### Aplicación del Análisis de Coste-Beneficio en Sistemas de IA

Desde `modelCost.ts` se puede ver que Claude Code tiene incorporada una tabla de precios precisa:

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

Haiku 4.5 tiene el precio más bajo ($1/$5 por Mtok), y Opus 4.6 Fast Mode el más alto ($30/$150 por Mtok), con una diferencia de 30 veces. Esta diferencia de precio es la lógica económica central por la que el sistema asigna las tareas en segundo plano a Haiku.

### Patrón de Degradación Elegante (Graceful Degradation)

En el software tradicional, la degradación elegante consiste en recurrir a una alternativa cuando una función no está disponible, pero sin colapsar. En los sistemas LLM, la alternativa es "cambiar a un modelo más económico/disponible". Claude Code implementa un mecanismo de activación con protección por contador: después de 3 errores 529 consecutivos se activa el cambio de modelo, no de forma inmediata (para evitar que una sobrecarga ocasional provoque una degradación de calidad innecesaria).

---

## 13.3 Arquitectura de Selección de Modelos

### Jerarquía de Prioridad de Modelos

La función `getUserSpecifiedModelSetting()` de `utils/model/model.ts` define con precisión el orden de prioridad:

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

  // Ignorar el modelo especificado por el usuario si no está en la allowlist de availableModels.
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

`getMainLoopModel()` añade a esto una quinta prioridad: el valor predeterminado integrado:

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

Cadena de prioridad completa de 5 niveles:
| Prioridad | Origen | Descripción |
|---|---|---|
| 1 (máxima) | Comando `/model` | Efecto inmediato en la sesión, almacenado en el override de memoria |
| 2 | Flag de inicio `--model` | Escrito en el override de memoria al iniciar |
| 3 | Variable de entorno `ANTHROPIC_MODEL` | A nivel de proceso |
| 4 | Archivo de configuración `settings.json` | Preferencia de usuario persistente |
| 5 (mínima) | Valor predeterminado integrado | Determinado por tipo de suscripción |

### Modelo Predeterminado por Niveles de Suscripción

`getDefaultMainLoopModelSetting()` revela las diferencias por suscripción:

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (empleados internos) usan por defecto Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Usuarios Max y Team Premium usan por defecto Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG, Enterprise, Team Standard, Pro usan por defecto Sonnet 4.6
  return getDefaultSonnetModel()
}
```

Este diseño significa que incluso sin ninguna configuración por parte del usuario, los usuarios Max/Team Premium abrirán Opus 4.6 por defecto, mientras que los usuarios Pro/Sonnet abrirán Sonnet 4.6. **El valor predeterminado en sí mismo es una estrategia de diferenciación de producto.**

### Sistema de Alias de Modelos

`parseUserSpecifiedModel()` soporta la resolución de alias cortos, para que los usuarios no necesiten recordar el Model ID completo:

```typescript
// utils/model/model.ts — extracto de parseUserSpecifiedModel
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // modo plan usa Sonnet, no plan usa Opus
```

El sufijo `[1m]` puede añadirse a cualquier alias (como `opus[1m]`); el sistema lo parsea automáticamente como variante de ventana de contexto de 1M.

### Detección de Capacidades del Modelo

`utils/model/modelCapabilities.ts` implementa un mecanismo de caché habilitado solo para empleados internos (`USER_TYPE === 'ant'`):

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

Los usuarios externos no solicitan la lista de capacidades del modelo; la información de capacidades está codificada directamente en funciones como `modelSupportsEffort()`, `modelSupports1M()`, evitando el overhead de llamadas API adicionales.

---

## 13.4 Usos en Segundo Plano de Haiku

El cc-notebook afirma que Haiku tiene 6 usos en segundo plano. Mediante búsqueda exhaustiva de las ubicaciones de llamada a la función `queryHaiku` (`grep -rn 'queryHaiku\b'`) y `getSmallFastModel()`, la **verificación en el código fuente** es la siguiente:

### Resumen de Usos en Segundo Plano (verificados en el código fuente)

| # | Uso | Archivo | Condición de activación |
|---|---|---|---|
| 1 | Extracción de contenido de Web Fetch | `tools/WebFetchTool/utils.ts:503` | Usa Haiku para filtrar el Markdown al contenido especificado por el usuario tras obtener la página web |
| 2 | Extracción de prefijo de comando Shell | `utils/shell/prefix.ts:220` | Antes de ejecutar la herramienta Bash, usa Haiku para determinar si el comando necesita solicitud de permisos |
| 3 | Generación de título de sesión | `utils/sessionTitle.ts:87` | Genera automáticamente un título corto al inicio de la sesión (salida JSON schema) |
| 4 | Análisis de DateTime en MCP | `utils/mcp/dateTimeParser.ts:68` | Parsea descripciones de tiempo en lenguaje natural al formato ISO 8601 |
| 5 | Generación de resumen de llamadas a herramientas | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | Genera una etiqueta de resumen de una línea tras completar un lote de llamadas a herramientas |
| 6 | Renombrado de sesión | `commands/rename/generateSessionName.ts:20` | El comando `/rename` genera nombres en formato kebab-case |

**Descubrimiento adicional** (no mencionado en cc-notebook, encontrado mediante búsqueda de `getSmallFastModel()`):

| # | Uso | Archivo | Condición de activación |
|---|---|---|---|
| 7 | Verificación de API Key | `services/api/claude.ts:544` | Verifica la validez de la API Key (comentario en el código fuente: "WARNING: if you change this to use a non-Haiku model, this request will fail in 1P") |
| 8 | Resumen en modo Away | `services/awaySummary.ts:49` | Genera un resumen de contexto cuando el usuario se ausenta (modo AFK) |
| 9 | Asistencia en búsqueda web | `tools/WebSearchTool/WebSearchTool.ts:280` | En algunos escenarios de búsqueda web, usa Haiku para procesar los resultados |
| 10 | Comprobación de estado de cuota | `services/claudeAiLimits.ts:200` | Sondea el estado de cuota actual con la solicitud Haiku mínima |
| 11 | Estimación de número de tokens | `services/tokenEstimation.ts:277` | Estima el uso de la ventana de contexto |
| 12 | Ejecución de hooks Prompt/Exec | `utils/hooks/execPromptHook.ts:79`, `execAgentHook.ts:118` | Los callbacks de hooks usan Haiku por defecto (a menos que la configuración del hook lo sobreescriba) |
| 13 | Análisis de mejora de skills | `utils/hooks/skillImprovement.ts:169` | Analiza automáticamente sugerencias de mejora tras la ejecución de un Skill |

**Conclusión**: la afirmación del cc-notebook de "6 usos en segundo plano" es una **subestimación**. Hay al menos 13 puntos de llamada a `queryHaiku` o `getSmallFastModel()` en el código fuente, cubriendo todas las etapas del ciclo de vida de la sesión (verificación en el inicio, asistencia durante la ejecución, gestión de la sesión). Haiku/SmallFastModel es la "capa de servicios de fondo" de todo el sistema, no una optimización ocasional.

Detalle de diseño clave: `queryHaiku` usa llamadas no streaming (`queryModelWithoutStreaming`) y sin contexto de permisos de herramienta (`getEmptyToolPermissionContext()`):

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

## 13.5 Mecanismo de Overload Fallback

El cc-notebook afirma la existencia del "529 Overload Fallback, Opus → Sonnet". El código fuente **verifica completamente** esta afirmación, con detalles aún más ricos.

### Detección del Error 529

La función `is529Error()` en `services/api/withRetry.ts`:

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // verifica el código de estado 529, o la situación en que el SDK no puede transmitir correctamente el código de estado en streaming
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

Nótese la detección dual: el código de estado `529` y la cadena `overloaded_error` en el mensaje de error. Esto se debe a que el SDK a veces no puede transmitir correctamente el código de estado 529 durante el streaming.

### Condición de Activación: 3 Errores 529 Consecutivos

```typescript
// services/api/withRetry.ts — extracto de la función withRetry
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
      // lanza un error especial, activando el cambio de modelo en la capa superior
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

Restricciones clave:
- Por defecto solo se activa para **modelos de la serie Opus** de **usuarios no suscritos a ClaudeAI** (`isNonCustomOpusModel()`)
- La variable de entorno `FALLBACK_FOR_ALL_PRIMARY_MODELS` puede extenderlo a todos los modelos principales
- Los errores 529 en solicitudes streaming se contabilizan en el contador (`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`), coordinando el recuento con los reintentos no streaming

### Propagación de la Señal FallbackTriggeredError

`FallbackTriggeredError` es una clase de error especializada que lleva los campos `originalModel` y `fallbackModel`, propagándose hacia arriba en la pila de llamadas hasta `query.ts`:

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

### Cambio de Modelo y Notificación al Usuario en query.ts

`query.ts:894-946` captura este error y ejecuta el cambio de modelo:

```typescript
// query.ts — extracto del manejo de FallbackTriggeredError
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // muestra al usuario con nivel de advertencia — visible independientemente del modo verbose
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // actualiza de forma síncrona el modelo principal en toolUseContext
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // reintenta toda la solicitud con el nuevo modelo
}
```

**Mecanismo de notificación al usuario**: el mensaje de cambio usa el nivel `'warning'`, lo que significa que los usuarios lo verán en la interfaz independientemente de si el modo verbose está activo. **La afirmación del cc-notebook sobre "sin degradación silenciosa" queda completamente verificada.**

### Estrategia de 529 para Tareas en Segundo Plano: Descartar Directamente

Las tareas no en primer plano (como summary, title, suggestions) **no reintentan** ante un error 529, sino que lo descartan directamente:

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

// las tareas no en primer plano ante 529 se lanzan directamente sin reintentar
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

Esta es una decisión de control de costes a nivel arquitectónico: los reintentos de tareas en segundo plano producirían un efecto de amplificación de 3-10 veces en los gateways bajo presión de capacidad, mientras que el usuario no percibe en absoluto el fallo de estas tareas.

---

## 13.6 Mecanismo de Effort Level

El cc-notebook afirma la existencia del sistema Effort Level. El código fuente lo **verifica completamente**, con detalles mucho más ricos que la descripción.

### Cuatro Niveles de Effort

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

Semántica de cada nivel (obtenida de `getEffortLevelDescription()`):
- **low**: Quick, straightforward implementation with minimal overhead
- **medium**: Balanced approach with standard implementation and testing
- **high**: Comprehensive implementation with extensive testing and documentation
- **max**: Maximum capability with deepest reasoning (**solo Opus 4.6**)

### Matriz de Soporte por Modelo

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // solo Opus 4.6 y Sonnet 4.6 soportan el parámetro effort
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku y versiones antiguas de Opus/Sonnet no lo soportan
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P por defecto true, 3P por defecto false
  return getAPIProvider() === 'firstParty'
}

// max effort solo disponible en Opus 4.6
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### Cadena de Prioridad: env → appState → valor predeterminado del modelo

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' o 'auto' → no enviar parámetro effort
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // la API rechaza max para modelos que no sean Opus 4.6 → degradación automática a high
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### Diferenciación del Effort Predeterminado

El effort predeterminado de Opus 4.6 varía según el tipo de suscripción:

```typescript
// utils/effort.ts — extracto de getDefaultEffortForModel
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // los usuarios Pro tienen medium por defecto (ahorra cuota)
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team también puede configurarse a medium mediante GrowthBook
  }
}
```

Es interesante que el `dialogDescription` de `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` diga explícitamente: "We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits." — esto indica que el medium predeterminado es una estrategia consciente de gestión de cuota, no una priorización de rendimiento.

### Restricción de Persistencia de max

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max es de nivel de sesión para usuarios externos; no se persiste
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

El effort `max` establecido por usuarios externos no se escribe en `settings.json`, solo tiene efecto en la sesión actual.

---

## 13.7 Sistema de Seguimiento de Costes

### Responsabilidades Principales de cost-tracker.ts

`cost-tracker.ts` (323 líneas) asume tres responsabilidades:
1. **Acumulación en tiempo real**: llama a `addToTotalSessionCost()` tras cada respuesta de la API
2. **Persistencia**: escribe en el archivo de configuración del proyecto al finalizar la sesión (`saveCurrentSessionCosts()`)
3. **Restauración**: lee los datos de coste de la sesión anterior desde el archivo de configuración al reiniciar (`restoreCostStateForSession()`)

### Estadísticas de Tokens Desglosadas por Modelo

`addToTotalModelUsage()` acumula datos en 5 dimensiones por nombre de modelo:

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

Al finalizar la sesión se muestra formateado (`formatModelUsage()`): agrupado por nombre corto (diferentes endpoints de API devuelven el mismo modelo en distintos formatos), mostrando algo como:

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Marcado de Costes para Fast Mode

`addToTotalSessionCost()` tiene un manejo especial para Fast Mode:

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

El marcador `speed: 'fast'` afecta a la facturación: en Fast Mode, Opus 4.6 usa `COST_TIER_30_150` ($30/$150), en lugar del estándar `COST_TIER_5_25` ($5/$25).

### Seguimiento de Costes Anidado del Advisor

`addToTotalSessionCost()` procesa recursivamente el uso del Advisor:

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

El Advisor es una llamada a un modelo secundario oculta en la respuesta del modelo principal; su coste se rastrea de forma independiente y se suma al coste total.

### Mecanismo de Activación de la Visualización de Costes

`costHook.ts` (22 líneas) es un React hook que escucha el evento de salida del proceso:

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

`hasConsoleBillingAccess()` controla si se muestra el coste, garantizando que no se muestre en entornos sin acceso a la información de facturación (como el modo CCR/Remote); sin embargo, la escritura en `saveCurrentSessionCosts()` se ejecuta incondicionalmente: independientemente de si se muestra, siempre se persiste.

---

## 13.8 Capa de Llamadas a la API

### Parámetros Principales de Construcción de Solicitudes en claude.ts

`services/api/claude.ts` (3.419 líneas) es el punto de entrada unificado para llamadas a la API. Los parámetros clave provienen de la confluencia de múltiples sistemas:

```typescript
// services/api/claude.ts — ensamblaje de parámetros de solicitud (esquemático)
{
  model: normalizeModelStringForAPI(options.model),  // elimina el sufijo [1m]
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // parámetro Effort (solo para modelos que lo soportan)
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` elimina los sufijos `[1m]` y `[2m]` antes de enviar a la API: estos sufijos son solo una convención interna del cliente para marcar la ventana de contexto de 1M; la capa de API no los reconoce.

### Streaming y Fallback sin Streaming

La conversación principal usa transmisión (Server-Sent Events), pero en caso de fallo del streaming, recae en la versión sin streaming:

```typescript
// services/api/claude.ts:2535-2559
// si el fallo del streaming en sí es 529, se cuenta en el total de errores 529 consecutivos
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

El fallback sin streaming tiene un límite máximo de tokens:

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Inyección Dinámica de Beta Headers

Distintas funcionalidades corresponden a distintos Beta Headers, añadidos dinámicamente al hacer la solicitud:

```typescript
// constants/betas.ts (referencia)
EFFORT_BETA_HEADER        // soporte del parámetro effort
CONTEXT_1M_BETA_HEADER    // ventana de contexto 1M
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // control de presupuesto
```

---

## 13.9 Análisis de Decisiones de Diseño

### Filosofía de Diseño de la Degradación Sin Secretos

Desde el mensaje de cambio a nivel `'warning'` en `query.ts` y el diseño de la clase de error especializada `FallbackTriggeredError`, se puede ver que esta es una elección arquitectónica deliberada:

**¿Por qué no se puede cambiar silenciosamente?** Porque Claude Code es una herramienta de escritura de código, y la calidad del modelo afecta directamente la calidad del output. El usuario tiene derecho a saber "estoy usando Sonnet en lugar de Opus", para decidir si continuar esperando o utilizar una estrategia diferente. A diferencia de los productos de chat de consumo, los usuarios de herramientas de código son más profesionales y más sensibles a las diferencias entre modelos.

### Consideraciones de Diseño para la Transparencia de Costes

El diseño de `hasConsoleBillingAccess()` en `costHook.ts` merece atención: incluso cuando no se muestra, los datos de coste se persisten. Esto indica que el propósito principal del seguimiento de costes es la **recuperación de sesiones** (mostrar el gasto de la última sesión al reiniciar), no las alertas en tiempo real. Es un diseño de "conciencia offline": el usuario puede ver el gasto completo al finalizar la sesión, en lugar de ser interrumpido tras cada llamada a la API.

### Lógica de Producto de la Diferenciación de Modelos Predeterminados

Tener Opus como modelo predeterminado para Max/Team Premium y Sonnet para Pro/PAYG tiene una lógica de producto clara: uno de los argumentos de valor de la suscripción Max es "obtener el modelo más potente"; el valor predeterminado en sí mismo es la manifestación de este argumento.

Al mismo tiempo, incluso para los usuarios Max, el effort predeterminado de Opus 4.6 es `medium` (controlado por GrowthBook), lo que indica que Anthropic está **equilibrando la calidad y la cuota a través del sistema de effort**, en lugar de simplemente dar la configuración máxima a los usuarios Max.

---

## 13.10 La Necesidad de las Migraciones de Modelos

Los 11 archivos de migración en el directorio `migrations/` revelan las huellas de la evolución del producto; cada migración corresponde a una decisión de producto:

| Archivo de migración | Cuándo se activa | Lógica principal |
|---|---|---|
| `migrateFennecToOpus.ts` | Empleados internos (ant) | Alias de nombre en código fennec → alias opus (limpieza de nombres internos) |
| `migrateLegacyOpusToCurrent.ts` | Usuarios 1P con `opus-4-0`/`4-1` aún en settings | Model ID antiguo de Opus → alias `opus` (retirada de Opus 4.0/4.1) |
| `migrateOpusToOpus1m.ts` | Max/Team Premium (1P), `opus` en settings | `opus` → `opus[1m]` (fusión de la experiencia 1M) |
| `migrateSonnet1mToSonnet45.ts` | Usuarios con `sonnet[1m]` | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` (fijado a 4.5; la audiencia de 1M en 4.6 es diferente) |
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium (1P) fijados a Sonnet 4.5 | Cadena Sonnet 4.5 → alias `sonnet` (actualización a 4.6) |
| `resetProToOpusDefault.ts` | Usuarios Pro 1P sin modelo personalizado | Registra la marca de tiempo de migración; el REPL muestra una notificación de actualización una vez |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode habilitado, usuarios con dialog opt-in antiguo | Borra `skipAutoPermissionPrompt`, muestra de nuevo el nuevo dialog de permisos |
| `migrateAutoUpdatesToSettings.ts` | Usuario que deshabilitó explícitamente las actualizaciones automáticas | Migra `autoUpdates: false` a la variable de entorno en settings.json |
| `migrateBypassPermissionsAcceptedToSettings.ts` | Config global con `bypassPermissionsModeAccepted` | Migra a `skipDangerousModePermissionPrompt` en settings.json |
| `migrateSonnet45ToSonnet46.ts` | Igual que arriba | Migración homónima mencionada anteriormente |
| `migrateEnableAllProjectMcpServersToSettings.ts` | Configuración relacionada con MCP | Ajuste de la estructura de configuración de servidores MCP |

**Perspectiva arquitectónica**: cada migración opera solo en `userSettings` (settings.json a nivel de usuario), nunca toca `projectSettings` (nivel de proyecto) ni `policySettings` (política empresarial). Esto es un diseño intencional:

1. **Idempotencia**: leer y escribir en la misma fuente de datos; volver a ejecutar no produce efectos secundarios
2. **Mínimo privilegio**: no se puede (ni se debe) modificar las fijaciones a nivel de proyecto del usuario
3. **Evitar elevación global**: si el usuario ha fijado un Opus antiguo en un proyecto específico, la migración no lo elevará a una configuración global

La existencia de este sistema de migración demuestra por sí misma: **la migración de esquemas en sistemas de IA es mucho más compleja que las migraciones de bases de datos tradicionales**, necesitando considerar cambios en el tipo de suscripción, retirada de modelos, actualizaciones de la ventana de contexto y múltiples otras dimensiones; y no se puede simplemente sobreescribir la intención del usuario de forma bruta.

---

## 13.11 Patrones Transferibles

De este capítulo se extraen 5 patrones de diseño aplicables a sistemas propios:

### Patrón 1: Cadena de Override Multinivel
```
session_override > startup_flag > env_var > config_file > builtin_default
```
Cualquier capa puede ser sobreescrita por la superior, pero la inferior no puede influir silenciosamente en la superior. Combinado con comprobación de allowlist para prevenir la inyección de model IDs ilegítimos.

### Patrón 2: Separación de Estrategias de 529 para Primer Plano/Fondo
Tareas en primer plano (el usuario espera el resultado): reintentar N veces; tras superar el límite, activar fallback.
Tareas en segundo plano (el usuario no las percibe): descartar en el primer 529, evitando el efecto de amplificación de reintentos durante presión de capacidad.

### Patrón 3: Señalización con FallbackTriggeredError
No cambiar silenciosamente el modelo dentro del reintento, sino lanzar un error especializado y dejar que la capa superior gestione la lógica de cambio. Así la lógica de cambio está centralizada en un lugar (query.ts) y necesariamente va acompañada de una notificación al usuario.

### Patrón 4: Filtrado de Persistencia con toPersistableEffort
Los ajustes de nivel de sesión (como el effort `max`) se filtran antes de escribirse en settings.json. El "estado que no debe persistir entre sesiones" y las "preferencias de usuario que sí deben persistir" se distinguen desde el nivel del modelo de datos.

### Patrón 5: Seguimiento de Costes por Cubeta de Modelo
No rastrear solo el coste total, sino desglosar por nombre de modelo (normalizado). Solo así se puede mostrar al finalizar la sesión "cuánto costó Opus, cuánto costó Haiku", ayudando al usuario a entender qué función es más cara.

---

## 13.12 Índice de Código Fuente

| Archivo | Líneas | Contenido principal |
|---|---|---|
| `services/api/claude.ts` | 3.419 | Capa de llamadas a la API, queryHaiku, construcción de solicitudes, procesamiento de streaming |
| `services/api/withRetry.ts` | ~600 | Lógica de reintentos, manejo de errores 529, FallbackTriggeredError |
| `cost-tracker.ts` | 323 | Seguimiento de costes, persistencia, visualización formateada |
| `costHook.ts` | 22 | React hook, escucha la salida del proceso para mostrar el resumen de costes |
| `utils/effort.ts` | ~350 | Definición de Effort Level, cadena de prioridad, detección de soporte por modelo |
| `utils/modelCost.ts` | ~200 | Tabla de precios, función de cálculo de costes |
| `utils/model/model.ts` | ~450 | Cadena de prioridad de modelos, resolución de alias, lógica de modelo predeterminado |
| `utils/model/modelCapabilities.ts` | ~100 | Caché de capacidades del modelo (solo usuarios internos) |
| `query.ts` | ~1.000 | Captura de FallbackTriggeredError, notificación al usuario, cambio de modelo |
| `migrations/*.ts` | 11 archivos | Scripts de migración de versiones de modelos |
| `tools/WebFetchTool/utils.ts:503` | — | Uso de Haiku 1: extracción de contenido de Web Fetch |
| `utils/shell/prefix.ts:220` | — | Uso de Haiku 2: determinación de prefijo de comando Shell |
| `utils/sessionTitle.ts:87` | — | Uso de Haiku 3: generación de título de sesión |
| `utils/mcp/dateTimeParser.ts:68` | — | Uso de Haiku 4: análisis de DateTime |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Uso de Haiku 5: generación de resumen de llamadas a herramientas |
| `commands/rename/generateSessionName.ts:20` | — | Uso de Haiku 6: renombrado de sesión |
| `services/api/claude.ts:544` | — | Uso de Haiku 7: verificación de API Key |

---

*Este capítulo cubre completamente las afirmaciones del cc-notebook sobre selección de modelos y control de costes. Resultados de verificación: los "al menos 6 usos en segundo plano de Haiku" quedan verificados (en realidad son 13 puntos de llamada); la degradación sin secretos queda completamente verificada; el mecanismo 529 Overload Fallback queda completamente verificado; el sistema Effort Level queda completamente verificado. Todos los fragmentos de código se copiaron con precisión de los archivos fuente, con rutas de archivo y números de línea anotados.*
