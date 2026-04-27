# Capítulo 4: Orquestación de Agentes y Arquitectura Multi-Agente

## 4.1 Descripción General y Posicionamiento

El sistema multi-agente de Claude Code es el subsistema más complejo de toda la arquitectura del producto, abarcando aproximadamente 8,700 líneas de código central en 12 módulos clave. Este sistema resuelve un problema de ingeniería fundamental: **cómo orquestar de forma segura y eficiente la ejecución concurrente de múltiples agentes LLM en una aplicación REPL de hilo único**.

El sistema proporciona tres modos de colaboración progresivos:

| Modo | Forma de activación | Concurrencia | Mecanismo de comunicación | Nivel de aislamiento |
|------|---------|--------|---------|---------|
| **Subagent (predeterminado)** | Llamada a AgentTool | Síncrono/asíncrono | Valor de retorno de función | AsyncGenerator en proceso |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | Totalmente asíncrono | XML `<task-notification>` | AbortController independiente |
| **Team Mode** | `spawnTeammate()` + TeamFile | Paralelo persistente | Buzón de archivos + sondeo | Tmux Pane / InProcess / Remoto |

Estos tres modos no son implementaciones independientes, sino que comparten el mismo motor central `runAgent()` (`runAgent.ts`), logrando diferentes características de comportamiento mediante combinación de parámetros — esta es una de las decisiones de diseño más elegantes de todo el sistema.

**Estadísticas del código fuente:**

| Archivo | Líneas | Responsabilidad |
|------|------|------|
| `AgentTool.tsx` | 1,397 | Punto de entrada unificado, decisiones de enrutamiento, gestión del ciclo de vida |
| `runAgent.ts` | 973 | Motor de ejecución del agente, bucle query() |
| `loadAgentsDir.ts` | 755 | Análisis de definiciones de agentes (Markdown/JSON/Plugin) |
| `agentToolUtils.ts` | 686 | Filtrado de herramientas, permisos, serialización de resultados |
| `UI.tsx` | 871 | Renderizado de progreso y resultados del agente |
| `coordinatorMode.ts` | 369 | Prompt de sistema y contexto del Coordinator |
| `SendMessageTool.ts` | 917 | Enrutamiento de mensajes de 5 vías |
| `spawnMultiAgent.ts` | 1,093 | Generación de compañeros de equipo (Tmux/InProcess) |
| `inProcessRunner.ts` | 1,552 | Implementación completa del backend InProcess |
| `teammateMailbox.ts` | 1,183 | Protocolo de buzón de archivos |
| `worktree.ts` | 1,519 | Aislamiento Git Worktree |

## 4.2 Fundamentos Teóricos

### 4.2.1 Relación entre el Modelo Actor y la Orquestación de Agentes

La arquitectura multi-agente de Claude Code es una variante pragmática del Modelo Actor en el dominio de la orquestación LLM. Las tres primitivas centrales del Modelo Actor clásico (Hewitt, 1973) — **recibir mensajes, crear nuevos actores, enviar mensajes** — tienen correspondencias claras en el código:

| Primitiva Actor | Implementación en Claude Code | Ubicación en el código |
|-----------|-----------------|---------|
| Recibir mensaje | Bucle de sondeo `waitForNextPromptOrShutdown()` | `inProcessRunner.ts:689-868` |
| Crear Actor | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| Enviar mensaje | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

Pero hay dos desviaciones clave del Modelo Actor puro:

1. **Jerarquía asimétrica**: el Leader tiene una vista global (AppState), los Workers solo tienen su propio ToolUseContext. Esto no es un Actor par a par, sino un árbol de supervisión con una jerarquía clara de Leader-Worker.
2. **Canal de estado compartido**: los Teammates del backend InProcess comparten el AppState raíz a través de `setAppStateForTasks` (`runAgent.ts:336-337`), en lugar de puro paso de mensajes. Esta es una concesión pragmática al Modelo Actor — dentro de un único proceso, el estado compartido es más eficiente que los mensajes serializados.

### 4.2.2 Paso de Mensajes vs. Modelo de Concurrencia de Memoria Compartida

El sistema usa simultáneamente dos modelos de concurrencia, seleccionando según el nivel de aislamiento:

**Modelo de paso de mensajes** (Team Mode - backend Tmux Pane):
```
Leader → writeToMailbox("worker-1", {...}) → sistema de archivos → readMailbox() → Worker
```
La comunicación se implementa mediante archivos JSON + bloqueo de archivos. `LOCK_OPTIONS` en `teammateMailbox.ts` configura reintentos con retroceso exponencial (10 reintentos, 5-100ms) para serializar escrituras concurrentes:

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**Modelo de memoria compartida** (backend InProcess):
```
Leader → setAppState(prev => {...}) → mismo AppState store ← getAppState() ← Worker
```
Los Teammates InProcess leen y escriben directamente en el store raíz mediante `toolUseContext.setAppStateForTasks`. Las condiciones de carrera se evitan mediante la semántica de actualización funcional estilo React `setAppState(prev => {...})` (aunque el subyacente no es React, adopta el mismo patrón CAS).

### 4.2.3 Patrón Coordinador en Sistemas Distribuidos

El diseño de Coordinator Mode mapea el patrón Coordinator clásico (también llamado Master-Worker) en sistemas distribuidos, pero añade una restricción única: **el propio Coordinator es un agente LLM; su "lógica de coordinación" no está codificada, sino programada mediante system prompt**.

La función `getCoordinatorSystemPrompt()` definida en `coordinatorMode.ts:126-369` devuelve un prompt estructurado de aproximadamente 5,000 caracteres, que contiene la estrategia completa de programación de Workers:

```typescript
// coordinatorMode.ts:161-167 — reglas clave de programación
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

Este patrón de "programar la lógica de coordinación mediante prompt" significa que el comportamiento del Coordinator puede ajustarse modificando el prompt — el flujo de trabajo de cuatro fases investigación→síntesis→implementación→verificación no está forzado por código, sino implementado mediante las capacidades de seguimiento de instrucciones del LLM. Esto contrasta marcadamente con la lógica de programación codificada del Coordinator tradicional distribuido.

## 4.3 Arquitectura y Estructuras de Datos

### 4.3.1 Diagrama de Arquitectura General (Leader-Worker)

```
                    ┌─────────────────────────────────────────┐
                    │        Usuario Humano (Terminal)          │
                    └──────────────┬──────────────────────────┘
                                   │ entrada del usuario
                    ┌──────────────▼──────────────────────────┐
                    │        REPL Principal (bucle query())     │
                    │    ┌──────────────────────────────┐      │
                    │    │  AgentTool.call() — decisión  │      │
                    │    └──┬─────────┬─────────┬───────┘      │
                    │       │         │         │               │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐      │
                    │  │ Sync   │ │ Async  │ │Teammate │      │
                    │  │Agent   │ │Agent   │ │Spawn    │      │
                    │  │(block) │ │(fire&  │ │         │      │
                    │  │        │ │forget) │ │         │      │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘      │
                    │      │          │         │               │
                    │      └────┬─────┘    ┌────▼──────────┐   │
                    │           │          │  spawnMulti-   │   │
                    │      ┌────▼────┐     │  Agent.ts      │   │
                    │      │runAgent │     └────┬───────────┘   │
                    │      │  .ts    │          │               │
                    │      │         │     ┌────▼──────────┐   │
                    │      │ query() │     │  3 Backends:   │   │
                    │      │  loop   │     │ • Tmux Pane    │   │
                    │      │         │     │ • InProcess    │   │
                    │      └─────────┘     │ • Remoto (ant) │   │
                    │                      └───────────────┘   │
                    └─────────────────────────────────────────┘

    Capa de comunicación:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Sync Agent:    yield message → padre recopila       │
    │  Async Agent:   XML <task-notification> → user msg   │
    │  Teammate:      buzón de archivos (.claude/teams/*)  │
    │  InProcess:     AppState compartido + buzón fallback │
    │  Remoto (ant):  teleportToRemote() → sesión CCR      │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 Sistema de Tipos de AgentDefinition

La definición de agente adopta un diseño de tipo de unión de tres capas:

```typescript
// loadAgentsDir.ts — jerarquía del tipo central

// Tipo base: campos compartidos por todos los agentes
type BaseAgentDefinition = {
  agentType: string              // clave de enrutamiento (ej. "Explore", "worker")
  whenToUse: string              // base para que el LLM seleccione el agente
  tools?: string[]               // lista blanca (undefined = todos)
  disallowedTools?: string[]     // lista negra
  model?: string                 // 'inherit' | nombre de modelo concreto
  effort?: EffortValue           // nivel de esfuerzo de razonamiento
  permissionMode?: PermissionMode // estrategia de herencia de permisos
  maxTurns?: number              // máximo de turnos de conversación
  background?: boolean           // siempre ejecutar en segundo plano
  isolation?: 'worktree' | 'remote' // modo de aislamiento
  memory?: AgentMemoryScope      // memoria persistente
  omitClaudeMd?: boolean         // omitir CLAUDE.md (ahorra ~5-15 Gtok/semana)
  // ...
}

// Built-in Agent: prompt dinámico, sin systemPrompt estático
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Custom Agent: cargado desde Markdown/JSON
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Plugin Agent: del sistema de plugins
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// Tipo de unión final
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

La elegancia de este diseño está en el método `getSystemPrompt`: los Built-in Agents reciben el parámetro `toolUseContext` (pueden ajustar dinámicamente el prompt según el conjunto de herramientas actual), mientras que los Custom/Plugin Agents usan cierres para capturar el contenido Markdown analizado en el momento del parseo. Esto significa:

- **El prompt del Built-in Agent es dinámico**: puede ser diferente en cada llamada
- **El prompt del Custom Agent es estático**: definido por el archivo Markdown, pero si `memory` está habilitado, se añade contenido de memoria en tiempo de ejecución (`loadAgentsDir.ts:335-340`)

La prioridad de carga de definiciones de agentes sigue la cadena de cobertura: `builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`, implementando "el último cubre al anterior" mediante Map en `getActiveAgentsFromList()` (`loadAgentsDir.ts:169-186`).

### 4.3.3 Abstracción Unificada de los Tres Backends de Ejecución

Los tres backends comparten la misma interfaz AsyncGenerator de `runAgent()`, pero difieren radicalmente en el modelo de proceso y el mecanismo de comunicación:

| Dimensión | Tmux Pane | InProcess | Remoto (solo ant) |
|------|-----------|-----------|-------------------|
| **Modelo de proceso** | Proceso Claude CLI independiente | Aislamiento AsyncLocalStorage en el mismo proceso | Sesión remota CCR |
| **Inicio** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **Comunicación** | Sondeo de buzón de archivos (500ms) | AppState compartido + buzón como fallback | HTTP API |
| **Permisos** | Contexto de permisos independiente | Puente de cola UI del Leader | Remoto independiente |
| **Costo de recursos** | Alto (proceso completo) | Bajo (heap V8 compartido) | Muy alto (instancia remota) |
| **Tiempo de vida** | Independiente del Leader | Vinculado al proceso Leader | Independiente |
| **Lógica de detección** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

La detección de backend y la degradación en `spawnMultiAgent.ts:339-375` implementa una cadena de degradación elegante:

```
iTerm2 (backend it2) → Tmux → fallback InProcess
```

Si se detecta iTerm2 pero la CLI it2 no está instalada, el sistema muestra un prompt de configuración interactivo (`It2SetupPrompt`), permitiendo al usuario elegir instalar it2 o degradar a Tmux.

### 4.3.4 Estructuras de Datos del Protocolo de Comunicación

**Formato de mensaje del buzón de archivos** (`teammateMailbox.ts:42-49`):

```typescript
type TeammateMessage = {
  from: string       // nombre del remitente
  text: string       // contenido del mensaje (puede ser texto plano o JSON estructurado)
  timestamp: string  // marca de tiempo ISO
  read: boolean      // indicador de leído
  color?: string     // identificador de color del remitente
  summary?: string   // resumen de vista previa UI (5-10 palabras)
}
```

La ruta del buzón sigue un formato fijo: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**Tipos de mensajes estructurados** (pasados como JSON codificado en el campo `text`):

| Tipo de mensaje | Dirección | Uso |
|---------|------|------|
| `shutdown_request` | Leader → Worker | Solicitar cierre |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | Respuesta al cierre |
| `idle_notification` | Worker → Leader | Notificación de inactividad |
| `permission_request` | Worker → Leader | Solicitud de permiso |
| `permission_response` | Leader → Worker | Respuesta de permiso |
| `plan_approval_request` | Worker → Leader | Solicitud de aprobación en Plan Mode |
| `plan_approval_response` | Leader → Worker | Respuesta de aprobación |
| `sandbox_permission_request` / `_response` | Bidireccional | Permisos de sandbox de red |
| `task_assignment` | Leader → Worker | Asignación de tarea |
| `team_permission_update` | Leader → Workers | Difusión de permisos |

## 4.4 Algoritmos Centrales y Flujos

### 4.4.1 Árbol de Decisión de Enrutamiento de AgentTool (Completo)

`AgentTool.call()` es el punto de entrada unificado del sistema; su lógica de enrutamiento se implementa en `AgentTool.tsx:238-764`. El árbol de decisión completo:

```
AgentTool.call(input) entrada
│
├─ [1] ¿Los parámetros team name + name existen?
│   ├─ SÍ: ¿Es un intento de Teammate de generar anidadamente?
│   │   ├─ SÍ: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NO: → spawnTeammate() (devolver teammate_spawned)
│   └─ NO: continuar
│
├─ [2] Resolver effectiveType (subagent_type)
│   ├─ Especificado explícitamente → usar valor especificado
│   ├─ No especificado + Fork Gate ON → undefined (ruta Fork)
│   └─ No especificado + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (ruta Fork)
│   ├─ SÍ: verificación recursiva de Fork
│   │   ├─ Ya en sub-proceso Fork → throw
│   │   └─ Pasado → selectedAgent = FORK_AGENT
│   └─ NO: buscar en activeAgents
│       ├─ Encontrado → selectedAgent = found
│       ├─ Denegado por permiso → throw (con info de deny rule)
│       └─ No existe → throw (listar agents disponibles)
│
├─ [4] Resolver effectiveIsolation
│   ├─ 'remote' (solo ant) → teleportToRemote() → devolver remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → pasos posteriores usan worktreePath
│
├─ [5] Construir system prompt y mensajes de prompt
│   ├─ Ruta Fork: heredar prompt padre + buildForkedMessages()
│   └─ Normal: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] Determinación de shouldRunAsync
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — PERO NO isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → devolver { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → entrar en bucle while(true)
│       ├─ Carrera: nextMessage vs backgroundSignal
│       │   ├─ background gana → cambiar a ejecución asíncrona (wasBackgrounded=true)
│       │   └─ message gana → yield message, rastrear progreso
│       └─ Bucle termina → finalizeAgentTool() → devolver AgentToolResult
```

### 4.4.2 Flujo de Ejecución de runAgent() AsyncGenerator

`runAgent()` es el motor central de todo el sistema multi-agente (`runAgent.ts:247-860`); es un `AsyncGenerator<Message, void>`: cada vez que se hace yield de un Message, el llamador puede procesarlo (registrar, mostrar o empujar a cola en segundo plano).

**Fases clave del flujo de ejecución:**

1. **Resolución de herramientas**: `resolveAgentTools()` resuelve la lista blanca `tools` de la definición del agente en objetos Tool reales, aplicando simultáneamente la lista negra `disallowedTools` (`runAgent.ts:500-502`)

2. **Construcción del System Prompt**: basado en `override?.systemPrompt` o `getAgentSystemPrompt()`; los Explore/Plan Agents omiten `claudeMd` y `gitStatus`, ahorrando ~5-15 Gtok/semana a nivel de flota (`runAgent.ts:389-409`)

3. **Estrategia de AbortController** (`runAgent.ts:524-528`):
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // control externo (ruta async)
     : isAsync
       ? new AbortController()      // asíncrono: controller independiente
       : toolUseContext.abortController  // síncrono: compartir controller del padre
   ```

4. **Cobertura de permisos** (`runAgent.ts:414-497`): el `permissionMode` del agente cubrirá el modo del padre, pero los modos de padre `bypassPermissions`, `acceptEdits` y `auto` siempre tienen prioridad — esto asegura que las políticas de seguridad establecidas por el administrador no puedan ser degradadas por sub-agentes.

5. **Bucle central** — llamada directa a `query()` con yield (`runAgent.ts:748-806`):
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... manejar stream_event, attachment, mensajes grabables
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **Bloque finally de limpieza** (`runAgent.ts:816-858`): limpieza MCP, limpieza de session hooks, seguimiento de prompt cache, liberación de caché de estado de archivos, desregistro de Perfetto, limpieza de todos de AppState, kill de tareas bash en segundo plano — 9 operaciones de limpieza en total, garantizando que no haya fugas de recursos.

### 4.4.3 Ciclo de Vida del Agente Asíncrono (fire-and-forget)

El ciclo de vida completo del agente asíncrono es impulsado por `runAsyncAgentLifecycle()` (`agentToolUtils.ts:322-497`):

```
registerAsyncAgent() → registrar tarea en AppState
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — recopilar todos los mensajes
   │   ├─ agentMessages.push(message)
   │   ├─ si task.retain → añadir a AppState.tasks[taskId].messages
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — evento de progreso SDK
   │
   ├─ finalizeAgentTool() — extraer resultado final
   │
   ├─ completeAsyncAgent() — marcar como completado (PRIMERO, antes de cualquier operación lenta)
   │   │                      ↑ decisión de diseño clave: corrección gh-20236
   │   │                        classifyHandoff y worktree cleanup pueden bloquearse
   │   │                        no pueden bloquear la transición de estado
   │
   ├─ classifyHandoffIfNeeded() — verificación del clasificador de seguridad (opcional)
   │
   ├─ getWorktreeResult() — limpieza de worktree
   │
   └─ enqueueAgentNotification() — notificar al padre con XML <task-notification>
```

**La corrección gh-20236** es una decisión de diseño que vale la pena registrar: `completeAsyncAgent()` se llama antes de `classifyHandoffIfNeeded()` y `getWorktreeResult()`. El comentario explica claramente la razón:

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 Filtrado de Herramientas y Herencia de Permisos

El filtrado de herramientas es una cadena de filtrado de tres capas (`agentToolUtils.ts:66-115`):

```
Capa 1: ALL_AGENT_DISALLOWED_TOOLS — herramientas prohibidas para todos los agentes
Capa 2: CUSTOM_AGENT_DISALLOWED_TOOLS — herramientas adicionales prohibidas solo para Custom Agents
Capa 3: ASYNC_AGENT_ALLOWED_TOOLS — lista blanca para agentes asíncronos (lógica invertida)
```

Excepciones especiales:
- Las herramientas MCP (prefijo `mcp__`) siempre están permitidas
- `ExitPlanMode` siempre está permitido en Plan Mode
- Los Teammates InProcess en modo Agent Swarms pueden usar `AgentTool` (generar sub-agentes síncronos) y herramientas Task (coordinar usando lista de tareas compartida)

La resolución de herramientas también admite wildcards (`'*'` o `undefined` = todas las herramientas) y restricciones de alcance de agente (sintaxis `AgentTool(worker, researcher)`, `agentToolUtils.ts:165-172`).

### 4.4.5 Flujo de Trabajo de Cuatro Fases del Coordinator Mode

La lógica central del Coordinator Mode se define mediante prompt en `getCoordinatorSystemPrompt()` de `coordinatorMode.ts:126-369`. Divide todas las tareas en cuatro fases:

**Fase 1: Investigación** (Workers ejecutan en paralelo)
- Múltiples Workers exploran simultáneamente el código base
- Instrucción clave en el prompt: *"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Fase 2: Síntesis** (el Coordinator lo hace él mismo)
- Esta es la fase más crítica — el Coordinator debe leer personalmente los resultados de la investigación y comprenderlos
- Anti-patrón explícitamente prohibido: *"Never write 'based on your findings'"*
- Requiere producir una especificación sintetizada con rutas de archivos concretas, números de línea y contenido de modificaciones

**Fase 3: Implementación** (Workers ejecutan)
- El Coordinator decide si `continue` (`SendMessageTool`) o `spawn fresh` (`AgentTool`)
- La decisión se basa en el grado de superposición de contexto (el prompt tiene una tabla de decisiones completa)

**Fase 4: Verificación** (Worker independiente)
- Requiere verificación independiente explícita: *"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- Criterio de verificación: *"proving the code works, not confirming it exists"*

### 4.4.6 Colaboración Persistente en Team Mode

Team Mode implementa estado persistente de equipo basado en TeamFile (`.claude/teams/{team_name}/team.json`). A diferencia de los Workers fire-and-forget del Coordinator Mode, los Teammates son **procesos de larga duración**:

1. **Generación**: `spawnTeammate()` crea un Tmux pane o tarea InProcess
2. **Ejecución**: Teammate ejecuta prompt → completa → envía `idle_notification` → espera el próximo prompt
3. **Comunicación**: todos los mensajes pasan por el buzón de archivos (todos los backends pueden usar comunicación mediante sistema de archivos)
4. **Cierre**: el Leader envía `shutdown_request` → el LLM del Teammate decide aprobar o rechazar

El bucle principal del InProcess Runner (`inProcessRunner.ts:883-1464`) implementa semántica de persistencia completa:

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. Ejecutar prompt actual (llamar a runAgent())
  // 2. Marcar como inactivo
  // 3. Enviar idle_notification al Leader
  // 4. waitForNextPromptOrShutdown() — sondear buzón
  //    ├─ shutdown_request → pasar al LLM para decidir
  //    ├─ new_message → establecer como prompt de la próxima ronda
  //    └─ aborted → shouldExit = true
}
```

Vale la pena notar la estrategia de prioridad de mensajes (`inProcessRunner.ts:760-804`):
1. Prioridad más alta: `shutdown_request` (la instrucción de cierre del Leader no se perderá)
2. Siguiente: mensajes de `team-lead` (el Leader representa la intención del usuario)
3. Finalmente: mensajes peer en cola FIFO

### 4.4.7 Protocolo de Comunicación del Buzón de Archivos

El buzón de archivos es la base de comunicación de todos los backends. Su diseño elige **simplicidad** sobre rendimiento:

**Protocolo de escritura** (`teammateMailbox.ts:133-191`):
```
1. ensureInboxDir() — asegurar que el directorio existe
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — creación atómica (si no existe)
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — adquirir bloqueo de archivo
4. readMailbox() — releer dentro del bloqueo (evitar lecturas sucias)
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — escribir de vuelta
7. release() — liberar bloqueo
```

**Protocolo de lectura** (`teammateMailbox.ts:83-107`):
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. devolver TeammateMessage[]
```

Nótese que la lectura es **sin bloqueo**: este es un diseño intencional. El extremo lector solo necesita consistencia eventual, mientras que el extremo escritor garantiza atomicidad mediante `lockfile`.

### 4.4.8 Enrutamiento de 5 Vías de SendMessage

`SendMessageTool.call()` implementa 5 rutas de enrutamiento de mensajes independientes (`SendMessageTool.ts`):

```
Valor de input.to
│
├─ [Ruta 1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — Control Remoto entre máquinas
│   (requiere verificación de seguridad: mensajes entre máquinas necesitan consentimiento explícito del usuario)
│
├─ [Ruta 2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — Unix Domain Socket local
│
├─ [Ruta 3] coincidencia con agentNameRegistry o toAgentId
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task detenida/desalojada → resumeAgentBackground()
│       (recuperación automática del Agent detenido desde transcript en disco)
│
├─ [Ruta 4] to === '*'
│   → handleBroadcast() — iterar sobre TeamFile.members y escribir en cada buzón
│
└─ [Ruta 5] otros
    ├─ texto plano → handleMessage() — escribir en buzón
    └─ mensaje estructurado → distribuir al handler correspondiente:
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

El **mecanismo de recuperación automática** en la Ruta 3 es especialmente elegante: cuando se envía un mensaje a un agente detenido, el sistema lo recupera automáticamente desde el transcript en disco y lo ejecuta en segundo plano. Esto significa que el Coordinator puede continuar sin problemas un Worker completado anteriormente mediante `SendMessage`, sin importarle si todavía está en ejecución.

### 4.4.9 Flujo Completo de Delegación de Permisos

El manejo de permisos del Teammate InProcess es una de las partes más complejas de todo el sistema (`inProcessRunner.ts:127-449`). El desafío central es: **¿cómo solicita autorización humana un agente en segundo plano?**

La solución es un fallback de dos capas:

**Ruta principal: Puente de cola de UI del Leader**
```
Worker activa herramienta que requiere permiso
  → createInProcessCanUseTool() es llamado
  → hasPermissionsToUseTool() devuelve { behavior: 'ask' }
  → Verificar auto-aprobación del clasificador Bash (si disponible)
  → getLeaderToolUseConfirmQueue() — obtener cola de confirmación de UI del Leader
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ identificador del Worker
     └→ terminal del Leader muestra diálogo de permisos con badge del Worker
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Ruta de fallback: Solicitud de permiso por buzón**
```
Worker activa herramienta que requiere permiso
  → Cola de UI del Leader no disponible
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → sondear propio buzón (intervalo de 500ms)
  → esperar que el Leader escriba permission_response de vuelta
  → processMailboxPermissionResponse()
```

La propagación de actualizaciones de permisos también es importante: cuando el Leader aprueba un permiso y selecciona "Always allow", `persistPermissionUpdates()` escribe en disco, mientras que `getLeaderSetToolPermissionContext()` escribe la actualización de vuelta al contexto compartido del Leader — pero con `preserveMode: true`, evitando que el modo `acceptEdits` del Worker se filtre de vuelta al Coordinator (`inProcessRunner.ts:275-277`).

### 4.4.10 Ciclo de Vida Completo del Worker

```
Nacimiento
  │
  ├─ Ruta Sync Agent:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → devolver AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ Ruta Async Agent:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ Ruta InProcess Teammate:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — bucle persistente:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → nuevo mensaje/shutdown/abort → decidir si continuar
          }

En ejecución
  │
  ├─ bucle query() → llamada API → tool_use → verificación canUseTool
  │   ├─ allow → ejecutar herramienta
  │   ├─ deny → herramienta rechazada
  │   └─ ask → diálogo de permisos (sync) o permisos por buzón (async/teammate)
  │
  ├─ Seguimiento de progreso:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (evento SDK)
  │
  └─ Paso automático a segundo plano (solo Sync Agent):
      carrera backgroundPromise → si el usuario presiona Ctrl+Z
      → wasBackgrounded = true → continuar en segundo plano

Comunicación
  │
  ├─ Sync Agent: yield message → padre recopila directamente
  ├─ Async Agent: <task-notification> inyectado en mensajes de usuario del padre
  └─ Teammate: writeToMailbox() → Leader lee sondeando

Terminación
  │
  ├─ Completado normalmente: finalizeAgentTool() → extraer texto final → marcar completed
  ├─ Kill del usuario: AbortError → killAsyncAgent() → extraer partialResult → notificar
  ├─ Error: catch → failAsyncAgent() → notificar error
  └─ Limpieza: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Creación y Limpieza del Aislamiento de Worktree

Git Worktree proporciona aislamiento a nivel del sistema de archivos para los agentes (`worktree.ts`). Flujo central:

**Creación** (`worktree.ts:234-374`):
```
1. validateWorktreeSlug(slug) — prevenir ataques de path traversal
2. Verificación de recuperación rápida: readWorktreeHeadSha() — si el worktree ya existe, omitir fetch
3. Si no existe:
   a. Intentar leer ref de origin/<default> local (evitar los 6-8s de git fetch)
   b. Si no existe localmente → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. Opcional: sparse-checkout (solo checkout de rutas especificadas)
4. performPostCreationSetup():
   - Copiar settings.local.json
   - Configurar git hooks (manejar problema core.hooksPath de husky)
   - Enlace simbólico de node_modules y otros directorios grandes
   - Copiar archivos gitignored especificados en .worktreeinclude
```

**Decisión de limpieza** (`AgentTool.tsx:644-685`):
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // Hook-based siempre conserva
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // Sin cambios, eliminar worktree
    }
  }
  return { worktreePath, worktreeBranch }; // Con cambios, conservar
};
```

Medidas de seguridad clave:
- `validateWorktreeSlug()` verifica que cada segmento separado por `/` coincida con `[a-zA-Z0-9._-]+`, previniendo path traversal con `../../../`
- `flattenSlug()` aplana slugs anidados (`user/feature` → `user+feature`), evitando conflictos D/F de git ref y anidamiento de directorios
- `GIT_NO_PROMPT_ENV` deshabilita todos los prompts de credenciales de git, evitando que la CLI se bloquee

## 4.5 Análisis de Decisiones de Diseño

### 4.5.1 Por Qué Elegir el Buzón de Archivos en Lugar de IPC

El buzón de archivos parece una elección "primitiva" — ¿por qué no usar Unix Domain Sockets, Named Pipes o gRPC?

**Razón central: independencia del backend**. El sistema de archivos es el mínimo común denominador de los tres backends (Tmux, InProcess, Remoto):
- Tmux Pane es un proceso independiente, sin memoria compartida
- InProcess está en el mismo proceso pero usa AsyncLocalStorage para aislamiento
- Remoto es a través de red, pero puede compartir sistema de archivos en red

Ventajas adicionales del buzón de archivos:
1. **Observabilidad**: un simple `cat ~/.claude/teams/*/inboxes/*.json` es suficiente para depurar
2. **Persistencia**: los mensajes no se pierden si el proceso se bloquea
3. **Simplicidad**: sin gestión compleja de conexiones, heartbeats ni reconexiones
4. **Seguridad de concurrencia**: el bloqueo de archivos proporcionado por `proper-lockfile` es suficientemente confiable

El costo es la **latencia**: un intervalo de sondeo de 500ms significa que en el peor caso la entrega de mensajes tiene 500ms de retraso. Pero en escenarios de agentes LLM, donde cada llamada a herramienta tarda varios segundos, 500ms es despreciable.

### 4.5.2 Tradeoff entre Backend InProcess y Tmux Pane

| Dimensión | InProcess | Tmux Pane |
|------|-----------|-----------|
| **Memoria** | Heap V8 compartido (bajo) | Heap de proceso independiente (alto) |
| **Latencia de inicio** | ~0ms | ~2-3s (inicio de CLI) |
| **Aislamiento** | AsyncLocalStorage (débil) | Proceso SO (fuerte) |
| **Permisos** | Puente de UI del Leader (en tiempo real) | Sondeo de buzón (con retraso) |
| **Depuración** | Registros compartidos (complejo) | Terminal independiente (intuitivo) |
| **Tiempo de vida** | Vinculado al Leader | Independiente |

La mayor ventaja del backend InProcess es el **puente de permisos**: mediante `getLeaderToolUseConfirmQueue()`, el diálogo de permisos del Worker se muestra directamente en el terminal del Leader con un badge del Worker. Esto significa que el usuario no necesita cambiar al terminal del Worker para aprobar permisos.

Pero InProcess tiene una limitación fundamental: **los Workers no pueden generar agentes en segundo plano** (`AgentTool.tsx:277-278`), porque su tiempo de vida está vinculado al proceso del Leader y los agentes en segundo plano necesitan AbortController independiente.

### 4.5.3 Filosofía de Diseño: Los Humanos Siempre Controlan los Permisos

Todo el sistema de permisos del sistema multi-agente sigue un principio no negociable: **los humanos siempre son los otorgadores finales de permisos**.

Este principio en el código:
1. **Los sub-agentes no pueden escalar permisos**: `runAgent.ts:419` — los modos del padre `bypassPermissions`, `acceptEdits` y `auto` siempre tienen prioridad sobre el `permissionMode` del sub-agente
2. **Los permisos del Leader no se filtran al Worker**: `runAgent.ts:467-477` — cuando se especifica `allowedTools`, se vacían las reglas de allow de nivel de sesión, conservando solo las reglas de nivel de argumento CLI
3. **Los mensajes entre máquinas requieren consentimiento explícito**: `SendMessageTool.ts:checkPermissions` — el envío a dirección `bridge:` requiere `safetyCheck`, y `classifierApprovable: false` (el clasificador de seguridad no puede auto-aprobar)
4. **Aprobación de Plan Mode**: los Teammates pueden configurarse como `plan_mode_required`; en ese caso deben enviar primero el plan al Leader para aprobación antes de ejecutar

### 4.5.4 Diseño Recursivo de Reutilización del Bucle query()

El núcleo de `runAgent()` es llamar a la función `query()` — y `query()` es exactamente la misma función que usa el bucle REPL principal. Esto significa que **los sub-agentes y el agente principal usan exactamente el mismo pipeline de llamadas al API y ejecución de herramientas**.

```typescript
// runAgent.ts:748-757 — llamada query() del agente
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```
