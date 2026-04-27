# Capítulo 9: Sistema de Memoria

## 9.1 Descripción General y Posicionamiento

El sistema de memoria de Claude Code es uno de los subsistemas más sofisticados y con mayor inversión de ingeniería de toda la cadena de herramientas. Resuelve la limitación más fundamental de los LLM: la ventana de contexto se reinicia a cero al terminar una sesión. Cada vez que el usuario abre una nueva sesión, Claude se enfrenta a una pizarra en blanco: no sabe quién es el usuario, cuáles son sus preferencias, qué errores cometió la última vez ni qué convenciones sigue el equipo.

El objetivo de diseño del sistema de memoria es: **permitir que Claude mantenga continuidad entre sesiones, actuando como un verdadero colaborador a largo plazo.**

En términos de volumen de código fuente, se trata de un sistema de considerable envergadura:
- Directorio `memdir/`: 7 archivos, 1.736 líneas
- `services/SessionMemory/`: 3 archivos, 1.026 líneas
- `services/extractMemories/`: 2 archivos, 769 líneas
- `services/teamMemorySync/`: 5 archivos, 2.167 líneas

Total: aproximadamente 5.700 líneas, alrededor del 1,1% del código base, aunque su densidad de complejidad y reflexión de diseño supera con creces esa proporción.

---

## 9.2 Fundamentos Teóricos

### El modelo de memoria humana como referencia

La arquitectura del sistema refleja explícitamente tres tipos de memoria de la ciencia cognitiva:

| Memoria humana | Equivalente en Claude Code | Implementación técnica |
|---|---|---|
| Memoria de trabajo (Working Memory) | Ventana de contexto actual | Lista de mensajes de la sesión, se borra al terminar |
| Memoria episódica (Episodic Memory) | Session Memory | `~/.claude/projects/<slug>/session-memory.md`, actualizada continuamente durante la sesión |
| Memoria semántica (Semantic Memory) | Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`, conservada a largo plazo entre sesiones |

Session Memory corresponde a "lo que se recuerda ahora mismo": registra qué se está haciendo en esta sesión y hasta dónde se ha llegado. Persistent Memory corresponde al "conocimiento acumulado": preferencias del usuario, lecciones aprendidas, contexto del proyecto.

### Grafo de conocimiento vs. memoria documental: la elección

El sistema optó por **documentos Markdown en el sistema de archivos** en lugar de bases de datos o índices vectoriales. Esta elección está explicitada en los comentarios de `memoryTypes.ts`:

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

Esto revela un principio de primer orden: **la información que se puede consultar en tiempo real no merece ser memorizada.** La memoria solo debe almacenar contexto "no derivable": las preferencias del usuario, las lecciones aprendidas en el equipo, las motivaciones detrás del proyecto. Esto contrasta radicalmente con el diseño de un grafo de conocimiento, que tiende a estructurar toda la información posible.

### Consistencia eventual aplicada a la memoria

El diseño de sincronización de Team Memory adopta explícitamente una semántica de consistencia eventual:

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

El diseño de que las eliminaciones no se propagan es intencional: la memoria del equipo es un activo de tipo "acumulación", y una eliminación accidental no debe convertirse en una pérdida permanente. Es una implementación conservadora del principio de consistencia eventual en sistemas distribuidos.

---

## 9.3 Arquitectura de Tres Capas

El sistema se compone de tres capas, ordenadas de menor a mayor ciclo de vida:

### Primera capa: Session Memory (nivel de sesión)

**Ruta del archivo**: `~/.claude/projects/<sanitized-cwd>/session-memory.md` (obtenida mediante `getSessionMemoryPath()`)

Session Memory es un archivo Markdown que se **mantiene activo durante la sesión actual**, con una estructura de contenido fija:

```markdown
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

(`services/SessionMemory/prompts.ts:14-36`, `DEFAULT_SESSION_MEMORY_TEMPLATE`)

No se borra al finalizar la sesión, sino que el mecanismo Auto Compact lo lee al comprimir el contexto y lo inyecta como "resumen previo" en la nueva ventana de contexto.

**Restricciones de estructura de datos**:
- Límite por sección: 2.000 tokens (`MAX_SECTION_LENGTH = 2000`)
- Límite total: 12.000 tokens (`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`)
- Cuando se supera el límite, el sistema añade una advertencia al prompt solicitando al Agent que comprima

**Ciclo de vida**: vinculado a la sesión del proyecto actual; se lee cuando se activa Auto Compact

### Segunda capa: Persistent Memory (memoria persistente entre sesiones)

**Ruta del archivo**: `~/.claude/projects/<sanitized-git-root>/memory/`

Esta es la capa de memoria a largo plazo. Cada recuerdo se almacena como un archivo `.md` independiente con frontmatter YAML:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

(`memdir/memoryTypes.ts:230-240`, `MEMORY_FRONTMATTER_EXAMPLE`)

La lógica de resolución de rutas la gestiona `getAutoMemPath()` (`memdir/paths.ts:173-190`), con la siguiente prioridad:

1. Variable de entorno `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` (usada en escenarios multi-usuario de Cowork)
2. `autoMemoryDirectory` en `settings.json` (solo se confía en orígenes policy/local/user; **no** en projectSettings, para prevenir que repositorios maliciosos secuestren la ruta de escritura)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` (por defecto)

Los worktrees de Git se unifican a la raíz canónica de Git (`findCanonicalGitRoot`), asegurando que distintos worktrees del mismo repositorio compartan la misma memoria.

**Ciclo de vida**: permanente, hasta que el usuario la elimine explícitamente o el Agent la actualice/borre

### Tercera capa: Team Memory (memoria compartida del equipo)

**Ruta del archivo**: `~/.claude/projects/<sanitized-git-root>/memory/team/` (valor retornado por `getTeamMemPath()`)

Team Memory es un subdirectorio de Persistent Memory, sincronizado entre todos los miembros autenticados del mismo repositorio GitHub mediante una API REST. Es una extensión por encima de Auto Memory; `isTeamMemoryEnabled()` primero comprueba `isAutoMemoryEnabled()` para asegurarse de que el sistema padre está activo.

**Ciclo de vida**: mantenido por el servidor de Anthropic, persistente entre usuarios y máquinas

---

## 9.4 Mecanismo de Índice MEMORY.md

MEMORY.md es el **archivo de índice** de la capa Persistent Memory, no un archivo de contenido. El sistema distingue claramente entre ambos en múltiples puntos:

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### Formato

Cada línea de MEMORY.md es un enlace al archivo de memoria específico:

```
- [Preferencia del usuario: respuestas concisas](feedback_terse_responses.md) — al usuario no le gusta resumir al final de la respuesta
- [Contexto del proyecto: reescritura del middleware de Auth](project_auth_rewrite.md) — requisito de cumplimiento legal, no deuda técnica
```

MEMORY.md se carga en el system prompt al inicio de cada sesión, por lo que su tamaño afecta directamente el consumo de tokens de cada solicitud.

### Doble límite: 200 líneas / 25 KB

El sistema define límites dobles estrictos en `memdir/memdir.ts`:

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

(`memdir/memdir.ts:30-33`)

La lógica de truncamiento se implementa en `truncateEntrypointContent()` (`memdir/memdir.ts:55-102`): primero se trunca por número de líneas, luego por bytes (cortando en el último salto de línea para evitar cortar una línea a mitad). Tras el truncamiento se añade un mensaje de advertencia indicando que el índice es demasiado largo.

**Intención de diseño**: ~125 caracteres por línea × 200 líneas ≈ 25 KB, un límite razonable basado en datos reales (percentil 97). El límite de bytes cubre el caso extremo de "menos de 200 líneas pero líneas extremadamente largas" (el percentil 100 observado fue de 197 KB sin superar el límite de líneas).

### Relación con los archivos de memoria

Escribir en memoria es una **operación en dos pasos**:
1. Escribir el archivo de contenido (`user_role.md`, `feedback_testing.md`, etc.)
2. Añadir la entrada correspondiente en MEMORY.md

Durante la lectura, solo se leen los archivos seleccionados por `findRelevantMemories` (véase sección 9.7); MEMORY.md reside permanentemente en el system prompt.

---

## 9.5 Cuatro Tipos de Memoria

El sistema restringe toda la memoria a cuatro tipos, una de las decisiones de diseño más importantes. Los tipos se definen en `memdir/memoryTypes.ts` (constante `MEMORY_TYPES`):

### Tipo `user`

**Casos de uso**: rol del usuario, objetivos, responsabilidades, nivel de conocimiento

**Cuándo se activa**: en cualquier momento en que se conozca el rol, preferencias, responsabilidades o nivel de conocimiento del usuario

**Propósito**: adaptar las respuestas al nivel cognitivo y las necesidades del usuario concreto

**Alcance**: siempre private (privado del individuo), incluso en modo Team Memory

**Contraejemplo (qué no guardar)**:
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### Tipo `feedback`

**Casos de uso**: correcciones y confirmaciones del usuario sobre la forma de trabajar, tanto "no hagas esto" como "continúa haciendo esto"

**Requisitos de estructura**:
- La regla en sí
- Línea `**Why:**` (con la razón, para poder juzgar si aplica en casos límite)
- Línea `**How to apply:**` (cuándo y dónde tiene efecto)

**Diseño único**: exige registrar tanto **correcciones ante fallos como confirmaciones de éxitos**:

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**Cuándo se activa**: cuando el usuario dice "no hagas esto" (corrección explícita) o "exactamente así" / "perfecto" (confirmación implícita, más difícil de detectar)

**Alcance**: private por defecto; solo se guarda como `team` cuando la directriz es claramente una norma de nivel de proyecto (por ejemplo, estrategia de pruebas, restricciones de compilación)

### Tipo `project`

**Casos de uso**: información sobre el trabajo en curso, objetivos, planes, bugs o eventos que **no pueden derivarse del código o del historial de Git**

**Requisitos de estructura**:
- El hecho/decisión en sí
- Línea `**Why:**` (motivación: generalmente restricciones, fechas límite o necesidades de partes interesadas)
- Línea `**How to apply:**` (cómo influye en las recomendaciones)

**Regla importante**: al guardar, las fechas relativas deben convertirse a fechas absolutas ("el próximo jueves" → "2026-04-08"), para que la memoria siga siendo interpretable con el paso del tiempo.

**Alcance**: `team` por defecto (el contexto del proyecto es intrínsecamente compartido)

**Característica de caducidad**: el tipo `project` se vuelve obsoleto con mayor rapidez; el campo `Why` ayuda a juzgar si la memoria sigue siendo válida.

### Tipo `reference`

**Casos de uso**: punteros hacia la ubicación de información en sistemas externos (proyectos de Linear, canales de Slack, dashboards de Grafana, etc.)

**Cuándo se activa**: cuando se conoce la ubicación de un recurso externo y su propósito

**Alcance**: generalmente `team`

**Ejemplo típico**:

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### Qué no se debe guardar (exclusiones explícitas)

`WHAT_NOT_TO_SAVE_SECTION` enumera explícitamente seis categorías que no deben guardarse (`memdir/memoryTypes.ts:196-207`):

1. Patrones de código, convenciones, arquitectura, rutas de archivos: derivables del estado actual del proyecto
2. Historial de Git, cambios recientes: `git log`/`git blame` son la fuente autorizada
3. Soluciones de depuración o métodos de corrección: la corrección está en el código, el contexto en el mensaje de commit
4. Contenido ya documentado en CLAUDE.md
5. Detalles de tareas temporales: trabajo en curso, estado provisional, contexto de la sesión actual
6. **Incluso el contenido anterior si el usuario lo solicita explícitamente**: si el usuario pide guardar una lista de PRs, la respuesta correcta es "¿hay algo inesperado o no obvio ahí? Eso sí merece guardarse"

---

## 9.6 Extracción Automática de Memorias

### Mecanismo de extracción con Fork Agent

La extracción de memorias utiliza el patrón "Fork Agent": se crea un contexto de Agent idéntico al de la sesión principal y se ejecuta de forma asíncrona en segundo plano, sin bloquear la conversación principal.

El núcleo de este mecanismo es `runForkedAgent()`. El Agent de extracción comparte la caché de prompts del padre, maximizando la tasa de aciertos de caché:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // no escribe en el registro principal, evita condiciones de carrera
  maxTurns: 5,            // límite estricto para evitar bucles de verificación
})
```

(`services/extractMemories/extractMemories.ts:258-267`)

El comentario de diseño sobre `maxTurns: 5` explica la intención:

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

La estrategia eficiente del Agent de extracción está diseñada explícitamente para completarse en 2 turnos:
- **Turno 1**: emite en paralelo todas las peticiones FileRead de los archivos que deben actualizarse
- **Turno 2**: emite en paralelo todas las peticiones FileWrite/FileEdit

### Cuándo se activa (Stop Hooks)

La extracción se activa **después de cada ciclo completo de consulta**: cuando el modelo produce una respuesta final sin `tool_use`, se llama a `executeExtractMemories()` desde `handleStopHooks`.

El estado se gestiona mediante closures; las variables clave son:

```typescript
let lastMemoryMessageUuid: string | undefined    // cursor: hasta dónde se extrajo la última vez
let inProgress = false                           // previene ejecuciones concurrentes
let pendingContext: {...} | undefined            // llamadas que llegan durante la ejecución
let turnsSinceLastExtraction = 0                // para control de throttle
```

(`services/extractMemories/extractMemories.ts:225-240`)

**Estrategia de control de concurrencia**: si llega una nueva llamada mientras hay una extracción en curso, se "guarda" en `pendingContext` en lugar de descartarse. Cuando termina la extracción actual, se ejecuta inmediatamente una "extracción posterior" con el contexto más reciente, asegurando que no se pierda el último lote de mensajes.

**Regla de exclusión mutua**: si el Agent principal ya ha escrito archivos de memoria (detectado mediante `hasMemoryWritesSince`), el Fork Agent omite la extracción de esa ronda y solo avanza el cursor:

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // el Agent principal escribió, omitir fork agent, avanzar cursor
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

(`services/extractMemories/extractMemories.ts:198-209`)

### Análisis del prompt de extracción

La filosofía central del prompt de extracción es la **eficiencia de información**:

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // pre-inyecta la lista de memorias existentes para evitar que el Agent gaste un turno en ls
  ].join('\n')
}
```

(`services/extractMemories/prompts.ts:20-47`)

La pre-inyección del manifiesto de memorias existentes (`existingMemories`) es una optimización clave: evita que el Agent desperdicie un turno listando el directorio, proporcionando directamente en el prompt un listado estructurado (nombre de archivo, tipo, timestamp, descripción).

### Mecanismo de activación de Session Memory

Session Memory usa un mecanismo de activación diferente: `postSamplingHooks` en lugar de Stop Hooks, evaluando si es necesario actualizar después de cada muestreo del modelo:

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

(`services/SessionMemory/sessionMemory.ts:130-150`)

Umbrales de activación por defecto (`DEFAULT_SESSION_MEMORY_CONFIG`, `services/SessionMemory/sessionMemoryUtils.ts:29-33`):

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| `minimumMessageTokensToInit` | 10.000 | Tokens mínimos para inicializar la memoria de sesión |
| `minimumTokensBetweenUpdate` | 5.000 | Tokens mínimos de crecimiento entre dos actualizaciones |
| `toolCallsBetweenUpdates` | 3 | Llamadas a herramientas mínimas entre actualizaciones |

Estos valores pueden ajustarse dinámicamente mediante configuración remota de GrowthBook (`tengu_sm_config`).

---

## 9.7 Recuperación Inteligente de Memorias

### Sonnet selecciona hasta 5 memorias relevantes

La recuperación de memorias no es una lectura exhaustiva, sino que **primero escanea los frontmatters, luego usa Sonnet para seleccionar las 5 más relevantes**.

El flujo central en `findRelevantMemories()` (`memdir/findRelevantMemories.ts:32-66`):

1. `scanMemoryFiles()` escanea el directorio de memorias, lee las primeras 30 líneas de cada archivo (el frontmatter) y devuelve `MemoryHeader[]`
2. Filtra las memorias ya mostradas en turnos anteriores (`alreadySurfaced`), reservando los 5 slots para contenido nuevo
3. Usa Sonnet con `selectRelevantMemories()`, seleccionando los nombres de archivo más relevantes en función de la consulta y la descripción del archivo
4. Devuelve las rutas y `mtime` de las memorias seleccionadas

### Lógica de relevancia

El system prompt de Sonnet está cuidadosamente diseñado:

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

(`memdir/findRelevantMemories.ts:13-23`)

**Diseño clave**: la documentación de referencia para herramientas usadas recientemente no debe seleccionarse (no se necesita mientras se usa), pero las memorias de **trampas/problemas conocidos** de esa misma herramienta sí deben seleccionarse (son más necesarias precisamente cuando se está usando).

La llamada a la API usa salida estructurada (JSON Schema) para garantizar un formato parseable:

```typescript
output_format: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      selected_memories: { type: 'array', items: { type: 'string' } },
    },
    required: ['selected_memories'],
    additionalProperties: false,
  },
},
```

(`memdir/findRelevantMemories.ts:97-108`)

### Cómo se inyectan las memorias en el contexto

Las memorias seleccionadas se inyectan antes del mensaje del usuario, envueltas en etiquetas `<system-reminder>` (`wrapMessagesInSystemReminder`). Las memorias con más de 1 día de antigüedad incluyen una advertencia de frescura:

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

(`memdir/memoryAge.ts:38-47`)

Este diseño resuelve un problema real: usuarios reportaron que el modelo hacía afirmaciones confiadas basadas en memorias obsoletas, citando rutas de archivos o nombres de funciones que habían sido modificados, lo que hacía las afirmaciones parecer más creíbles de lo que deberían.

**Mecanismo anti-deriva**: `MEMORY_DRIFT_CAVEAT` se inyecta en el system prompt, indicando al Agent que verifique el estado actual antes de responder basándose en memorias:

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Sincronización de Team Memory

### Mecanismo de sincronización mediante API REST

Team Memory implementa sincronización con el servidor a través de `services/teamMemorySync/`, con una API descrita completamente al inicio de `index.ts`:

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → solo metadatos+hashes
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert entries
404  = sin datos aún
```

(`services/teamMemorySync/index.ts:10-13`)

La sincronización requiere **autenticación OAuth** (necesita `CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`), y utiliza el repositorio GitHub (`owner/repo`) como scope.

**Mecanismo watcher**: `watcher.ts` usa `fs.watch({recursive: true})` para escuchar cambios en el directorio `team`, activando un push tras un debounce de 2 segundos (`DEBOUNCE_MS = 2000`). Se eligió deliberadamente `fs.watch` nativo en lugar de chokidar:

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS usa FSEvents (O(1) en descriptores de archivo), Linux usa inotify (O(número de subdirectorios)), ambos superiores al esquema kqueue de chokidar.

### Bloqueo optimista (If-Match)

Las subidas usan control de concurrencia optimista, enviando el ETag (checksum) mediante la cabecera HTTP `If-Match`:

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

(`services/teamMemorySync/index.ts:uploadTeamMemory`)

Cuando el servidor devuelve 412 Precondition Failed, indica un conflicto (otro usuario modificó la memoria compartida en ese intervalo). El sistema usa el endpoint `GET ?view=hashes` (ligero, solo devuelve los hashes SHA-256 de cada clave, sin cuerpo de contenido) para refrescar `serverChecksums` localmente, recalcula el delta y reintenta:

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### Estrategia de resolución de conflictos

La estrategia de resolución de conflictos es **el servidor gana (server wins per-key)**: al hacer Pull, el contenido del servidor sobreescribe el local. El push de delta solo sube las claves cuyo contenido difiere del hash del servidor; el servidor usa semántica upsert (las claves no incluidas en el PUT se conservan).

El límite de subida por lote (`MAX_PUT_BODY_BYTES = 200_000`) previene que el cuerpo de la solicitud sea demasiado grande para el API Gateway (se observó que el gateway devuelve un 413 en formato HTML para cuerpos de ~256-512 KB, diferente del 413 estructurado de la capa de aplicación). Cuando se supera el límite, se divide automáticamente en varios PUTs secuenciales; la semántica upsert garantiza la seguridad:

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // empaquetamiento greedy: lotes por bytes, sin superar MAX_PUT_BODY_BYTES
  ...
}
```

(`services/teamMemorySync/index.ts:batchDeltaByBytes`)

**Supresión de fallos permanentes**: algunos errores (no_oauth, no_repo, 4xx distintos de 409/429) no pueden resolverse con reintentos. Al detectar estos errores, el sistema establece `pushSuppressedReason` para evitar que el watcher caiga en un bucle de reintentos infinito (se observó un dispositivo sin OAuth que emitió 167K eventos push en 2,5 días).

---

## 9.9 Análisis de Decisiones de Diseño

### Por qué usar el sistema de archivos en lugar de una base de datos

El diseño basado en sistema de archivos + Markdown ofrece varias ventajas clave:

1. **El Agent puede operarlo directamente**: FileRead/FileWrite/FileEdit son las herramientas nativas de Claude. Escribir memorias y escribir código usan el mismo conjunto de herramientas, reduciendo la carga cognitiva.

2. **El usuario puede inspeccionarlo**: `~/.claude/projects/.../memory/` es una carpeta ordinaria; el usuario puede hacer directamente `ls`, `cat`, `vim`. Completamente transparente.

3. **Amigable con Git**: los archivos Markdown soportan de forma natural diff, grep y git history, lo que facilita el cálculo de deltas para Team Memory.

4. **Evita abstracciones innecesarias**: una base de datos requiere migraciones de esquema, estrategias de respaldo y capa de consultas. Para "unos cientos de KB de archivos Markdown", eso sería sobreingeniería.

### Por qué limitar el tamaño de MEMORY.md

El límite de 200 líneas / 25 KB se basa en datos empíricos (percentiles p97/p100 observados). La razón central:

- MEMORY.md se carga en el system prompt en **cada solicitud**, por lo que su tamaño afecta directamente el consumo de tokens
- Un índice demasiado grande expulsa contexto verdaderamente útil
- El límite forzado obliga a usuarios y Agents a mantener el índice conciso, con solo "ganchos" por línea, no contenido

Es el diseño típico de "usar restricciones para promover la calidad": no porque técnicamente sea imposible albergar más, sino para guiar el uso correcto mediante restricciones.

### Consideraciones de seguridad en el diseño de la memoria

El sistema cuenta con múltiples capas de diseño de seguridad:

**Protección contra path traversal**: `teamMemPaths.ts` implementa tres capas de verificación: primero comprobación de cadena de texto para `..`, codificación URL y ataques de normalización Unicode, luego resolución de la ruta real mediante `realpath` para verificar la ruta real del sistema de archivos:

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

(`memdir/teamMemPaths.ts:130-133`)

**Escaneo de secretos**: al escribir en Team Memory, `scanForSecrets()` escanea 30 patrones de credenciales de alta confianza (subconjunto de reglas de gitleaks), incluyendo formatos de tokens de AWS, GCP, GitHub, Anthropic, OpenAI y otras plataformas principales. El escaneo se ejecuta **dos veces**: antes de subir y antes de escribir:

- `checkTeamMemSecrets()` de `teamMemSecretGuard.ts` intercepta escrituras en la fase `validateInput` de FileWriteTool/FileEditTool
- `readLocalTeamMemory()` vuelve a escanear antes del push, omitiendo archivos con información sensible

**Control de herramientas con mínimo privilegio**: la función `canUseTool` del Agent de extracción solo permite:
- FileRead/Grep/Glob (solo lectura)
- Comandos Bash de solo lectura (ls/find/cat/stat/wc/head/tail)
- FileEdit/FileWrite con rutas dentro del directorio de memoria

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

(`services/extractMemories/extractMemories.ts:171-176`)

**Exención de seguridad para ProjectSettings**: la configuración de `autoMemoryDirectory` solo confía en orígenes policy/local/user, excluyendo explícitamente projectSettings (`.claude/settings.json`):

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 Patrones Transferibles

Los siguientes patrones del diseño del sistema de memoria de Doramagic son directamente aplicables:

### Patrón 1: El principio de lo no derivable

**Qué debe memorizarse**: cualquier información que pueda obtenerse consultando el estado actual (código, archivos, git) no merece ser memorizada. La memoria solo debe almacenar "contexto histórico": por qué se tomó esta decisión, qué trampas se pisaron, las preferencias implícitas del usuario.

**Aplicación en Doramagic**: las capas "UNSAID" y "WHY" que extrae Soul Extractor cumplen naturalmente este principio. Los documentos de reglas de OpenClaw son consultables, no necesitan memorizarse; pero "esta regla de OpenClaw causó un fallo de publicación" sí es el tipo de lección que merece ser memorizada.

### Patrón 2: Escritura en dos pasos + índice ligero

El patrón de escritura en dos pasos (archivo + índice) garantiza que el índice sea siempre conciso (restringiendo cada línea a 150 caracteres), mientras los archivos de contenido pueden ser detallados. El consumo de tokens del índice es fijo; la lectura del contenido es bajo demanda.

**Aplicación en Doramagic**: MEMORY.md del sistema de memoria es similar al "catálogo de bloques" de Doramagic: un índice ligero y cargable que apunta a archivos detallados expandibles bajo demanda.

### Patrón 3: Extracción en segundo plano con Fork Agent

No bloquear la conversación principal, compartir la caché de prompts y maximizar los aciertos de caché es el patrón estándar para tareas de postprocesamiento en segundo plano. Detalles de implementación clave:
- `skipTranscript: true` evita escribir en el registro principal de la sesión
- `maxTurns: N` previene que el Agent entre en bucles de verificación
- El mecanismo de cursor (`lastMemoryMessageUuid`) asegura que cada ejecución solo procese el incremento
- La estrategia Stash + trailing run garantiza que no se pierdan los últimos mensajes mientras el Agent está ocupado

### Patrón 4: Conciencia de frescura

Las memorias no son hechos eternamente válidos, sino observaciones con caducidad. El sistema garantiza esto mediante:
1. Añadir una indicación de antigüedad ("hace N días") en la recuperación
2. Inyectando instrucciones anti-deriva en el system prompt (verificar antes de citar)
3. Indicando al Agent que actualice activamente las memorias obsoletas en lugar de conservarlas

Esto es especialmente relevante para el escenario de "extracción de conocimiento" de Doramagic: los WHY/UNSAID extraídos pueden volverse obsoletos con la evolución del proyecto, y se necesita un mecanismo similar para mantener su frescura.

### Patrón 5: Escaneo de secretos como requisito previo

Antes de cualquier escritura "a través de una frontera" (escritura en espacio compartido, subida a la red), se deben escanear las credenciales. La biblioteca de reglas de gitleaks ofrece un conjunto de patrones de alta confianza reutilizable directamente. Diseño clave: el escaneo se ejecuta en la fase `validateInput` de la herramienta de escritura (no a posteriori), asegurando que los secretos no toquen ninguna ruta de persistencia.

---

## 9.11 Índice de Código Fuente

| Archivo | Líneas | Responsabilidad principal |
|---|---|---|
| `services/SessionMemory/sessionMemory.ts` | 495 | Lógica principal de Session Memory: condiciones de activación, llamada al Fork Agent, API de activación manual |
| `services/SessionMemory/prompts.ts` | 324 | Plantilla de Session Memory, construcción del prompt de actualización, análisis de tamaño por sección |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Gestión de estado de Session Memory: configuración, evaluación de umbrales, utilidades de espera/sincronización |
| `services/extractMemories/extractMemories.ts` | 615 | Extracción de Persistent Memory: llamada al Fork Agent, estado de closure, control de concurrencia |
| `services/extractMemories/prompts.ts` | 154 | Construcción del prompt de extracción: variantes auto-only y combinada (con Team Memory) |
| `memdir/memdir.ts` | 507 | Lógica de truncamiento de MEMORY.md, construcción del prompt de memorias, garantía de creación de directorio |
| `memdir/paths.ts` | 278 | Resolución de rutas de Auto Memory, evaluación de habilitación/deshabilitación, validación de seguridad de rutas |
| `memdir/memoryTypes.ts` | 271 | Definición de cuatro tipos de memoria, formato de frontmatter, principios de recuperación/anti-deriva/no derivable |
| `memdir/findRelevantMemories.ts` | 141 | Selección de recuperación con Sonnet: escanear frontmatter → 5 memorias relevantes |
| `memdir/memoryScan.ts` | 94 | Primitivas de escaneo de directorio: leer frontmatter, formatear listado |
| `memdir/memoryAge.ts` | 53 | Cálculo de frescura: días, texto legible, advertencia de obsolescencia |
| `memdir/teamMemPaths.ts` | 292 | Rutas de Team Memory, protección contra path traversal (tres capas), resolución de enlaces simbólicos |
| `memdir/teamMemPrompts.ts` | 100 | Construcción del prompt combinado de Team Memory + Auto Memory |
| `services/teamMemorySync/index.ts` | 1.256 | Núcleo de sincronización: lógica fetch/push, bloqueo optimista, batching, reintentos de conflicto |
| `services/teamMemorySync/watcher.ts` | 387 | Escucha de archivos: push con debounce, supresión de fallos permanentes, ciclo de vida inicio/parada |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 reglas de escaneo de secretos (subconjunto gitleaks), funciones de redacción |
| `services/teamMemorySync/types.ts` | 156 | Zod Schema: TeamMemoryData, tipos de resultado de sincronización, SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | Interceptación de secretos pre-escritura: integración en validateInput de FileWriteTool/FileEditTool |

**Referencia rápida de constantes clave**:

| Constante | Valor | Ubicación |
|---|---|---|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25.000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH` (por sección de Session Memory) | 2.000 tokens | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12.000 tokens | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10.000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5.000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| Límite de recuperación | 5 memorias | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| Límite de archivos de memoria | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Líneas de lectura de frontmatter | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Timeout de Team Memory | 30.000 ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Debounce de push | 2.000 ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| Tamaño máximo por archivo | 250.000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| Límite del cuerpo PUT | 200.000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
| Reglas de escaneo de secretos | 30 | `secretScanner.ts:SECRET_RULES` |
