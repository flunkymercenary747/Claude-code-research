# Capítulo 3: Sistema de Herramientas

## 3.1 Descripción General y Posicionamiento

El sistema de herramientas de Claude Code es la capa de ejecución de todo el producto. El LLM es responsable del razonamiento y la toma de decisiones, pero los efectos secundarios reales —leer archivos, ejecutar comandos, buscar código, acceder a la red— se realizan todos a través del sistema de herramientas. El sistema de herramientas es el único canal entre la intención del LLM y el mundo real.

En términos de escala, este es un subsistema bastante extenso:
- En la instantánea del código fuente, el directorio de herramientas contiene **40+ subdirectorios**, que abarcan operaciones de archivos, ejecución de código, coordinación de agentes, integración MCP, gestión de tareas y otras categorías
- El archivo de abstracción central `Tool.ts` tiene 792 líneas, el archivo de registro de herramientas `tools.ts` tiene 389 líneas, y el motor de ejecución de herramientas `services/tools/toolExecution.ts` tiene 1,745 líneas
- El módulo de almacenamiento de resultados de herramientas `utils/toolResultStorage.ts` tiene 1,040 líneas, manejando independientemente los problemas de presupuesto de tokens

Esta escala señala un hecho: **el sistema de herramientas no es un accesorio de Claude Code, sino su activo de ingeniería central**. La fiabilidad, seguridad y extensibilidad de todo el producto están determinadas en gran medida por la calidad del diseño del sistema de herramientas.

En el análisis competitivo (cc-notebook) no hay un capítulo separado sobre el sistema de herramientas, lo que es un punto ciego de análisis obvio — este capítulo llena ese vacío.

---

## 3.2 Fundamentos Teóricos

### Patrón de Self-Describing Tools (Herramientas Auto-Descriptivas)

En las llamadas a API tradicionales, el llamador necesita conocer las especificaciones de la interfaz de antemano. El sistema de herramientas de Claude Code adopta una filosofía de diseño diferente: **cada herramienta se auto-describe con sus capacidades, formato de entrada y restricciones de uso**.

Esto se refleja en varios campos centrales del tipo `Tool`:

```typescript
// Tool.ts:300-310 (simplificado)
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // resumen de capacidad de 3-10 palabras para coincidencia de palabras clave de ToolSearch
  description(input, options): Promise<string>   // genera descripción dinámicamente
  prompt(options): Promise<string>               // prompt de sistema completo de la herramienta
  inputSchema: Input           // schema Zod, tanto documentación como validador
  outputSchema?: z.ZodType
  // ...
}
```

`description()` y `prompt()` son métodos asíncronos, lo que significa que la auto-descripción de las herramientas puede **generarse dinámicamente**: ajustando el contenido del prompt según el contexto de permisos actual, las herramientas instaladas y el estado del entorno. No es documentación estática, sino descripciones conscientes del contexto generadas en tiempo de ejecución.

### Arquitectura de Plugins e Inyección de Dependencias

El sistema de herramientas es esencialmente una arquitectura de plugins. Cada herramienta se construye usando la función factory `buildTool()`, implementa la interfaz `Tool` unificada, pero están completamente desacopladas entre sí. Para agregar una nueva herramienta solo se necesita:

1. Crear un directorio de herramienta (ej. `tools/MyTool/`)
2. Implementar la interfaz `ToolDef`
3. Registrarla en `getAllBaseTools()` de `tools.ts`

Las herramientas no dependen entre sí (las dependencias circulares se rompen mediante lazy require), pero todas dependen de `ToolUseContext`: un objeto de contexto que atraviesa toda la cadena de ejecución, conteniendo estado de permisos, historial de mensajes, estado de la aplicación, etc.

```typescript
// Tool.ts:167-172 (simplificado)
export type ToolUseContext = {
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  contentReplacementState?: ContentReplacementState
  // ...
}
```

El diseño de `ToolUseContext` es inyección de dependencias clásica: todas las dependencias externas necesarias durante la ejecución de la herramienta se pasan a través del contexto, y las herramientas mismas son componentes funcionales sin estado. Esto hace posibles las pruebas, el aislamiento y la ejecución en sub-agentes.

### El Papel de Function Calling en LLM

Claude Code sigue el protocolo de Function Calling del API de Anthropic. El LLM puede producir bloques `tool_use` durante el razonamiento, especificando el nombre de la herramienta a llamar y sus parámetros; los resultados de ejecución se devuelven al LLM como bloques `tool_result`, como entrada para la siguiente ronda de razonamiento.

La restricción clave de este bucle es: **las definiciones de herramientas (nombre + input schema) deben enviarse al LLM en el system prompt**, consumiendo valiosos tokens de contexto. Cuando el número de herramientas supera 40, más las herramientas de terceros MCP, este costo se vuelve considerable — esto es lo que dio origen al mecanismo de carga diferida de ToolSearch descrito en la sección 3.6.

---

## 3.3 Arquitectura y Estructuras de Datos

### Abstracción Unificada buildTool()

`buildTool()` es la función factory central del sistema de herramientas, definida en `Tool.ts:756-769`:

```typescript
// Tool.ts:756-769
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

Hace una cosa: combinar el `ToolDef` proporcionado por el usuario (que permite omitir campos opcionales) con `TOOL_DEFAULTS` (valores predeterminados seguros), devolviendo un `Tool` completo.

Los valores predeterminados (`Tool.ts:729-742`) reflejan la filosofía de diseño **fail-closed (fallo seguro)**:

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // por defecto no es seguro para concurrencia
  isReadOnly: (_input?) => false,            // por defecto asume escritura
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // por defecto permite, manejado por sistema de permisos general
  toAutoClassifierInput: (_input?) => '',    // por defecto omite clasificador de seguridad
  userFacingName: (_input?) => '',
}
```

Vale la pena notar que `isConcurrencySafe` es `false` por defecto — lo que significa que el sistema prefiere ejecutar dos herramientas en serie antes de arriesgarse a ejecutar en paralelo operaciones con posibles efectos secundarios. Solo las herramientas que declaran explícitamente `isConcurrencySafe: () => true` (como GrepTool, GlobTool y otras herramientas de solo lectura) serán programadas en paralelo.

### Definición del Tipo Central de Herramientas

Los métodos de la interfaz `Tool` se pueden dividir en varios dominios funcionales (`Tool.ts:297-580`):

**Dominio de ejecución**
- `call(args, context, canUseTool, parentMessage, onProgress)` — método de ejecución central, devuelve `Promise<ToolResult<Output>>`
- `validateInput(input, context)` — validación antes de ejecutar, devuelve `ValidationResult`
- `checkPermissions(input, context)` — verificación de permisos, independiente del sistema de permisos general

**Dominio de descripción** (capacidad de auto-descripción de herramientas)
- `description(input, options)` — descripción corta, para la lista de tools en el API
- `prompt(options)` — prompt de sistema completo, informa al modelo cómo usar esta herramienta
- `searchHint` — resumen de capacidad de 3-10 palabras, exclusivamente para coincidencia de palabras clave de ToolSearch

**Dominio de renderizado** (componentes React, solo en modo REPL)
- `renderToolUseMessage(input, options)` — UI cuando comienza la llamada a herramienta
- `renderToolResultMessage(content, progressMessages, options)` — UI para los resultados de herramienta
- `renderToolUseProgressMessage(progressMessages, options)` — UI de progreso durante la ejecución
- `renderToolUseRejectedMessage(input, options)` — UI cuando se rechaza

**Dominio de metadatos**
- `isConcurrencySafe(input)` — declara si es seguro ejecutar en paralelo
- `isReadOnly(input)` — declara si es solo lectura (afecta el juicio de permisos)
- `isDestructive(input)` — declara si es irreversible (eliminar, sobrescribir, enviar)
- `shouldDefer` — si cargar de forma diferida (cargado bajo demanda por ToolSearch)
- `alwaysLoad` — siempre cargar en el prompt (sin diferir)
- `maxResultSizeChars` — umbral de activación para persistir resultados de herramientas en disco

La estructura de `ToolResult<T>` (`Tool.ts:289-298`) también merece atención:

```typescript
// Tool.ts:289-298
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

`contextModifier` permite modificar el contexto después de la ejecución de la herramienta (pero el comentario indica claramente: **solo las herramientas que no son seguras para concurrencia ejecutarán contextModifier** — esta es una restricción de seguridad de concurrencia importante).

### Mecanismo de Registro y Descubrimiento de Herramientas

`getAllBaseTools()` en `tools.ts` es la única fuente de verdad para el registro de herramientas (`tools.ts:108-186`). Esta función devuelve todas las herramientas integradas disponibles en el entorno actual y controla la disponibilidad de herramientas mediante múltiples capas de condiciones:

**Condiciones de entorno** (process.env):
- `USER_TYPE === 'ant'` — herramientas internas de Anthropic (ConfigTool, TungstenTool, REPLTool)
- `NODE_ENV === 'test'` — herramientas de prueba (TestingPermissionTool)
- `ENABLE_LSP_TOOL` — herramienta de integración LSP
- `CLAUDE_CODE_VERIFY_PLAN` — herramienta de verificación de plan

**Condiciones de Feature Flag** (`feature()` de `bun:bundle`):
- `PROACTIVE` / `KAIROS` — SleepTool (comportamiento proactivo)
- `AGENT_TRIGGERS` — ScheduleCronTool y otras herramientas de tareas programadas
- `COORDINATOR_MODE` — herramientas relacionadas con el modo coordinador
- `WEB_BROWSER_TOOL` — herramienta de navegador
- `WORKFLOW_SCRIPTS` — herramientas de flujo de trabajo

**Condiciones en tiempo de ejecución**:
- `isToolSearchEnabledOptimistic()` — si agregar ToolSearchTool
- `isTodoV2Enabled()` — si agregar el conjunto de herramientas de gestión de tareas
- `isAgentSwarmsEnabled()` — si agregar herramientas de colaboración en equipo
- `hasEmbeddedSearchTools()` — si los bfs/ugrep están integrados, no agregar GlobTool/GrepTool

La **deduplicación y ordenación** de herramientas (`assembleToolPool()`, `tools.ts:218-248`) adopta una estrategia cuidadosamente diseñada: las herramientas integradas y las herramientas MCP se ordenan por separado y luego se concatenan, con las herramientas integradas como prefijo y las herramientas MCP agregadas al final. Esto es para mantener la estabilidad del system prompt (prompt cache stability): el servidor de Anthropic establece puntos de ruptura de caché en posiciones fijas; si las herramientas integradas y las MCP se ordenaran mezcladas, cualquier nueva herramienta MCP rompería el caché.

---

## 3.4 Catálogo Completo de Herramientas

Basándose en la estructura del directorio `tools/` y la lógica de registro de `tools.ts`, se puede compilar un catálogo completo de herramientas:

### Operaciones de Archivos (File Operations)

| Herramienta | Directorio | Función | Seguro para concurrencia |
|------|------|------|---------|
| FileReadTool | `FileReadTool/` | Leer archivos, soporte PDF/imágenes/Notebook, lectura paginada | Sí |
| FileEditTool | `FileEditTool/` | Reemplazo preciso de cadenas, soporte replace_all | No |
| FileWriteTool | `FileWriteTool/` | Escribir/crear archivos | No |
| GlobTool | `GlobTool/` | Buscar archivos por patrón glob | Sí |
| GrepTool | `GrepTool/` | Búsqueda de contenido regex con ripgrep | Sí |
| NotebookEditTool | `NotebookEditTool/` | Edición de celdas de Jupyter Notebook | No |

### Ejecución de Código (Execution)

| Herramienta | Directorio | Función | Notas |
|------|------|------|------|
| BashTool | `BashTool/` | Ejecución de comandos Shell, soporte tareas en segundo plano, sandbox | Herramienta central |
| PowerShellTool | `PowerShellTool/` | Ejecución Windows PowerShell | Habilitación condicional |
| REPLTool | `REPLTool/` | Ejecución REPL en entorno VM aislado | Interno Ant |

### Coordinación de Agentes (Agent Orchestration)

| Herramienta | Directorio | Función |
|------|------|------|
| AgentTool | `AgentTool/` | Lanzar sub-agentes (subagents), soporte ejecución paralela |
| SendMessageTool | `SendMessageTool/` | Enviar mensajes a otros agentes |
| TeamCreateTool | `TeamCreateTool/` | Crear equipos de agentes |
| TeamDeleteTool | `TeamDeleteTool/` | Eliminar equipos de agentes |
| TaskCreateTool | `TaskCreateTool/` | Crear tareas en segundo plano |
| TaskGetTool | `TaskGetTool/` | Obtener estado de tareas |
| TaskUpdateTool | `TaskUpdateTool/` | Actualizar estado de tareas |
| TaskListTool | `TaskListTool/` | Listar todas las tareas |
| TaskStopTool | `TaskStopTool/` | Detener tareas |
| TaskOutputTool | `TaskOutputTool/` | Obtener salida de tareas |

### Contexto y Descubrimiento (Context & Discovery)

| Herramienta | Directorio | Función |
|------|------|------|
| SkillTool | `SkillTool/` | Cargar y ejecutar Skills (~/.claude/skills/) |
| ToolSearchTool | `ToolSearchTool/` | Buscar herramientas cargadas de forma diferida |
| MCPTool (dinámico) | `MCPTool/` | Herramientas de servidor MCP (registro dinámico en tiempo de ejecución) |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | Listar recursos MCP |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | Leer recursos MCP |
| LSPTool | `LSPTool/` | Integración con servidor de lenguaje LSP |

### Planificación y Estado (Planning & State)

| Herramienta | Directorio | Función |
|------|------|------|
| EnterPlanModeTool | `EnterPlanModeTool/` | Entrar en modo de planificación (solo lectura, sin ejecución) |
| ExitPlanModeTool | `ExitPlanModeTool/` | Salir del modo de planificación |
| EnterWorktreeTool | `EnterWorktreeTool/` | Entrar al entorno aislado de git worktree |
| ExitWorktreeTool | `ExitWorktreeTool/` | Salir del entorno worktree |
| TodoWriteTool | `TodoWriteTool/` | Escribir lista Todo (mostrada en barra lateral) |
| BriefTool | `BriefTool/` | Generar resumen de sesión |

### Acceso a Red (Network)

| Herramienta | Directorio | Función |
|------|------|------|
| WebFetchTool | `WebFetchTool/` | Recuperación HTTP, conversión HTML→Markdown, verificación de seguridad de dominio |
| WebSearchTool | `WebSearchTool/` | Búsqueda en internet |

### Sistema y Programación (System & Scheduling)

| Herramienta | Directorio | Función | Condición |
|------|------|------|------|
| ConfigTool | `ConfigTool/` | Leer/escribir configuración | Interno Ant |
| SleepTool | `SleepTool/` | Esperar (modo proactivo) | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | Salida sintética (uso especial) | — |
| ScheduleCronTool | `ScheduleCronTool/` | Crear/eliminar/listar tareas programadas | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | Disparadores remotos | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | Preguntar al usuario (interactivo) | — |

---

## 3.5 Flujo de Ejecución de Herramientas

### Flujo Completo desde tool_use del LLM hasta la Ejecución de Herramientas

El punto de entrada de la ejecución de herramientas es la función `runToolUse()` en `services/tools/toolExecution.ts` (`toolExecution.ts:298-428`), que es un async generator:

```
El LLM produce un bloque tool_use
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — buscar herramienta, soporte de alias (compatibilidad con herramientas renombradas)
    ↓
abortController.signal.aborted? → devuelve CANCEL_MESSAGE
    ↓
streamedCheckPermissionsAndCallTool() [devuelve AsyncIterable]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — validación de tipo Zod
  2. tool.validateInput(input, context)  — validación personalizada de la herramienta
  3. runPreToolUseHooks()                — ejecutar hooks PreToolUse
  4. canUseTool()                        — verificación de permisos (puede mostrar UI de confirmación)
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — persistir resultados grandes
  7. runPostToolUseHooks()               — ejecutar hooks PostToolUse
    ↓
yield MessageUpdateLazy (contiene tool_result)
    ↓
siguiente ronda de razonamiento del LLM
```

Un diseño de compatibilidad hacia atrás importante (`toolExecution.ts:350-360`): cuando una herramienta se renombra, el nombre antiguo se conserva como `aliases`. Cuando la herramienta no se puede encontrar en `options.tools`, el sistema también busca en `getAllBaseTools()` por coincidencia de alias — garantizando que los nombres de herramientas antiguas en transcripciones históricas puedan aún ejecutarse.

### Ejecución Streaming de Herramientas (Streaming Tool Execution)

La ejecución de herramientas se implementa en streaming mediante `Stream<MessageUpdateLazy>` (`toolExecution.ts:500-535`):

```typescript
// toolExecution.ts:500-535 (simplificado)
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // mensajes de progreso
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // resultados finales
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

El significado del diseño streaming: la UI puede mostrar el progreso en tiempo real mientras la herramienta aún se está ejecutando (por ejemplo, la salida en tiempo real de BashTool, el progreso de sub-agentes de AgentTool). Los mensajes de progreso y los resultados finales se pasan por el mismo canal `Stream`, simplificando el código del consumidor.

### Ejecución Paralela de Herramientas

Claude Code soporta que el LLM produzca múltiples bloques `tool_use` en una única respuesta, ejecutándolos en paralelo. El requisito previo para la concurrencia es: **todas las herramientas deben declarar `isConcurrencySafe: () => true`**.

Al ejecutarse en paralelo, `contextModifier` no se ejecuta (como dice el comentario de `ToolResult`: "contextModifier is only honored for tools that aren't concurrency safe"). Esta es una restricción de seguridad importante: las operaciones que modifican el contexto global no pueden realizarse en un entorno concurrente.

Herramientas típicamente seguras para concurrencia: GrepTool, GlobTool, FileReadTool (todas declaran `isConcurrencySafe: () => true`).

---

## 3.6 ToolSearch — Mecanismo de Carga Diferida

### Por Qué se Necesita ToolSearch (Problema de Inflación del Prompt)

La definición de cada herramienta (nombre + JSON Schema + descripción) consume tokens cuando se envía al LLM. Cuando el número de herramientas supera cierto umbral (experimentos muestran aproximadamente 40-60 herramientas), los problemas que surgen son:

1. **Aumento del costo de tokens**: cada llamada al API lleva grandes definiciones de herramientas
2. **Dilución de atención**: el LLM frente a decenas de herramientas puede reducir su atención a cada una
3. **Riesgo de invalidación de prompt cache**: los cambios en la lista de herramientas (como la adición dinámica de herramientas MCP) invalidarán el caché

La solución de ToolSearch es **carga bajo demanda**: la mayoría de las herramientas se marcan con `shouldDefer: true`, sin enviar el schema completo en el prompt inicial, cargándose solo después de ser descubiertas por búsqueda.

### Registro y Descubrimiento de Herramientas Diferidas

Las herramientas declaran si deben cargarse de forma diferida mediante el campo `shouldDefer` (`Tool.ts:456-462`):

```typescript
// Tool.ts:456-462
readonly shouldDefer?: boolean

/**
 * When true, this tool is never deferred — its full schema appears in the
 * initial prompt even when ToolSearch is enabled. For MCP tools, set via
 * `_meta['anthropic/alwaysLoad']`. Use for tools the model must see on
 * turn 1 without a ToolSearch round-trip.
 */
readonly alwaysLoad?: boolean
```

La función `isDeferredTool()` (definida en `tools/ToolSearchTool/prompt.ts`) determina si una herramienta debe ser diferida: las herramientas con `shouldDefer: true` y sin `alwaysLoad: true` se marcan como diferidas.

La propia ToolSearchTool **nunca se difiere**: debe estar disponible desde el primer turno, o no se podrían descubrir otras herramientas.

### Implementación de la Carga Bajo Demanda

El método `call()` de ToolSearchTool (`ToolSearchTool.ts:221-302`) soporta dos modos de consulta:

**Modo de selección directa** (prefijo `select:`):
```
query: "select:NotebookEdit"     → devuelve directamente NotebookEditTool
query: "select:Read,Edit,Grep"   → selección múltiple de herramientas
```

**Modo de búsqueda por palabras clave**:
```
query: "jupyter notebook"        → coincidencia de palabras clave, devuelve NotebookEditTool, etc.
query: "mcp__github"             → coincidencia de prefijo de servidor MCP
```

Algoritmo de puntuación de búsqueda (`ToolSearchTool.ts:155-198`):

```
Coincidencia exacta de nombre de herramienta (MCP): +12 puntos
Coincidencia exacta de nombre de herramienta (normal): +10 puntos
Nombre de herramienta contiene parcialmente la palabra clave (MCP): +6 puntos
Nombre de herramienta contiene parcialmente la palabra clave (normal): +5 puntos
Coincidencia completa de nombre de herramienta como comodín: +3 puntos
Coincidencia de límite de palabra en searchHint: +4 puntos (resumen de capacidad cuidadosamente seleccionado, señal fuerte)
Coincidencia de límite de palabra en texto de descripción: +2 puntos
```

El peso del campo `searchHint` (+4 puntos) es mayor que el del texto de descripción (+2 puntos), lo que alienta a los desarrolladores de herramientas a proporcionar resúmenes de capacidad precisos. Por ejemplo, `searchHint` de GrepTool: `'search file contents with regex (ripgrep)'`, y de FileEditTool: `'modify file contents in place'`.

Los resultados de búsqueda se devuelven al LLM como bloques `tool_reference` (`ToolSearchTool.ts:330-352`), que es una extensión especial del API de Anthropic que informa al servidor "por favor, inyecta el schema completo de estas herramientas en la lista de herramientas de la conversación actual".

---

## 3.7 Almacenamiento de Resultados de Herramientas

### Estrategia de Almacenamiento en Disco

Los resultados de la ejecución de herramientas pueden ser muy grandes (por ejemplo, leer un archivo de log de 10MB, ejecutar un comando que produce mucha salida). Colocar grandes resultados directamente en el historial de mensajes desperdicia tokens e infla el contexto de las solicitudes posteriores.

`utils/toolResultStorage.ts` implementa una estrategia de **persistencia bajo demanda**:

1. Calcular el tamaño del resultado (`contentSize()`)
2. Comparar con el umbral `maxResultSizeChars` de la herramienta (analizado mediante `getPersistenceThreshold()`)
3. Los resultados que superan el umbral se escriben en `~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`
4. Se reemplaza por un mensaje con ruta de archivo + vista previa

```typescript
// toolResultStorage.ts:168-177
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  let message = `${PERSISTED_OUTPUT_TAG}\n`
  message += `Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}\n\n`
  message += `Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):\n`
  message += result.preview
  message += result.hasMore ? '\n...\n' : '\n'
  message += PERSISTED_OUTPUT_CLOSING_TAG
  return message
}
```

`PREVIEW_SIZE_BYTES = 2000` (aproximadamente 2KB), la vista previa se trunca en el último salto de línea, evitando cortar líneas a mitad.

Un diseño de **idempotencia** clave (`toolResultStorage.ts:145-158`): al escribir se usa `{ flag: 'wx' }` (creación exclusiva); si el archivo ya existe, el error de escritura se ignora y se usa el archivo existente para generar la vista previa. Esto garantiza que la reproducción de mensajes históricos durante microcompact no escriba dos veces, y no produzca errores por EEXIST.

FileReadTool tiene un manejo especial: `maxResultSizeChars: Infinity` — los resultados de la herramienta de lectura nunca se persisten en disco. La razón se explica en el comentario: "persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits" (persistir crea un ciclo: Read lee el archivo, el resultado es demasiado grande y se persiste como archivo, el modelo usa Read para leer ese archivo...).

### Gestión del Presupuesto de Tokens

`toolResultStorage.ts` también implementa un **presupuesto de resultados de herramienta a nivel de mensaje**. Esto se impulsa mediante el mecanismo `ContentReplacementState` (`toolResultStorage.ts:395-440`):

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // tool_use_ids que ya pasaron la verificación de presupuesto (resultado congelado)
  replacements: Map<string, string>  // ID reemplazado → cadena de contenido reemplazada
}
```

Restricción central: **una vez que un resultado es juzgado (reemplazado o no), nunca cambia** (garantizado por el conjunto `seenIds`). Esto es por la estabilidad del prompt cache: el procesamiento del mismo tool_use_id debe ser consistente durante toda la sesión, o el caché se invalidará por cambios de contenido.

El límite del presupuesto es controlado dinámicamente por el feature flag de GrowthBook `tengu_hawthorn_window`; cuando el total de resultados de herramientas en un mensaje supera el límite, el sistema reemplaza los resultados de herramientas más grandes por versiones persistidas en disco hasta que el total caiga dentro del presupuesto.

---

## 3.8 Análisis de Decisiones de Diseño

### Tradeoff entre Auto-Descripción vs. Registro Externo

Claude Code eligió el modelo de **auto-descripción** (cada herramienta lleva su propio schema, descripción, prompt y lógica de renderizado), en lugar de centralizar esta información en un registro.

Ventajas:
- **Herramientas completamente auto-contenidas**: agregar una herramienta nueva solo requiere un directorio, sin modificar la lógica del registro central
- **Las descripciones pueden generarse dinámicamente**: `description()` y `prompt()` son funciones asíncronas que pueden ajustar su contenido según el entorno, permisos y estado de instalación
- **Lógica de renderizado coexiste con la herramienta**: los componentes React de renderizado están directamente en los archivos de herramienta; cambiar el comportamiento de la herramienta y cambiar la UI es el mismo PR

Desventajas:
- **Inflación de la interfaz de herramientas**: el tipo `Tool` tiene 40+ métodos/campos; los autores de nuevas herramientas necesitan familiarizarse con muchos detalles de interfaz
- **Código duplicado**: cada herramienta tiene métodos de renderizado como `renderToolUseMessage`, `renderToolResultMessage`, los patrones son muy similares
- **`buildTool()` no puede eliminar todo**: proporciona valores predeterminados, pero muchos métodos aún necesitan ser implementados por cada herramienta

En la práctica, Claude Code mitiga la duplicación de código mediante **componentes UI compartidos** (como `tools/shared/`) y **extracción de patrones** (como `lazySchema()`), pero la complejidad fundamental de la interfaz persiste.

### Por Qué Algunas Herramientas se Cargan de Forma Diferida

Las decisiones de carga diferida de ToolSearch siguen un principio: **todas las herramientas que posiblemente no se necesiten en la primera ronda de conversación deben diferirse**.

Las herramientas con `alwaysLoad` (nunca diferidas) deben satisfacer: el modelo necesita conocer su existencia desde la primera ronda. Ejemplos típicos son AgentTool, BashTool, FileReadTool — estas son herramientas fundamentales para cualquier tarea de programación.

Las herramientas con `shouldDefer` (carga diferida) suelen ser: herramientas necesarias solo en escenarios específicos (NotebookEditTool solo se necesita para tareas de Jupyter), grandes cantidades de herramientas MCP (el usuario instaló decenas de servidores MCP, pero en cada conversación solo usa unos pocos).

Las herramientas MCP activan el mecanismo ToolSearch por defecto según el número de herramientas, pero se puede forzar que no se difieran configurando `_meta['anthropic/alwaysLoad']` en los metadatos de la herramienta.

### Diseño por Capas de los Permisos de Herramientas

Los permisos de herramientas adoptan un diseño de **tres capas de defensa**:

1. **Validación de tipo Zod** (primer paso de `checkPermissionsAndCallTool`): el inputSchema de la herramienta valida estrictamente los tipos de parámetros; los parámetros del tipo incorrecto generados por el LLM se rechazan y se devuelve un mensaje de error
2. **Validación personalizada de la herramienta** (`validateInput()`): la herramienta implementa su propia validación de lógica de negocio; por ejemplo, FileEditTool verifica que old_string y new_string sean diferentes, verifica que el tamaño del archivo no supere 1GiB
3. **Sistema de permisos general** (`canUseTool()` + `checkPermissions()`): toma la decisión final basándose en las reglas de allow/deny configuradas por el usuario, si la herramienta es de solo lectura, si la operación es destructiva, etc.; puede mostrar confirmación interactiva

Estas tres capas se ejecutan en secuencia; el fallo de cualquier capa provoca un cortocircuito y no se entra en la siguiente.

---

## 3.9 Patrones Transferibles

### Diseño General del Patrón de Herramienta Auto-Descriptiva

El patrón más valioso para transferir extraído del sistema de herramientas de Claude Code es: **la herramienta como plugin auto-contenido**.

Para cualquier sistema que necesite exponer capacidades al LLM:

1. **Cada herramienta es un plugin auto-contenido**: nombre, descripción, schema de entrada, lógica de ejecución, lógica de renderizado, todo encapsulado en un directorio
2. **Las descripciones se generan dinámicamente**: no documentación estática, sino generación consciente del contexto en tiempo de ejecución
3. **Fail-closed como predeterminado**: `isConcurrencySafe: false`, `isReadOnly: false` — la postura predeterminada es conservadora, requiriendo declaración explícita antes de habilitar comportamientos de mayor riesgo
4. **Separación de ejecución y renderizado**: `call()` devuelve datos, los métodos de renderizado transforman los datos en UI; la misma ejecución de herramienta puede tener diferentes representaciones en CLI y API

### Optimización de Tokens de ToolSearch: Carga Diferida Bajo Demanda

La idea central de carga diferida se puede generalizar: **identificar subconjuntos centrales vs. periféricos, asegurar que el subconjunto central siempre esté disponible, y diferir el subconjunto periférico para carga bajo demanda**.

Para el sistema de herramientas de Doramagic:
- Siempre cargar: herramientas de uso frecuente en el dominio (ej. herramientas de extracción de conocimiento básicas)
- Diferir: herramientas especializadas de dominio (ej. herramientas de análisis de código específicas del lenguaje), cargadas de forma diferida cuando se detecta el tipo de archivo correspondiente mediante ToolSearch
