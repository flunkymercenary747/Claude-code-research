# Capítulo 7: Sistema de Comandos

## 7.1 Descripción General y Posicionamiento

El sistema de comandos de Claude Code es el punto de entrada central para la interacción del usuario con el REPL. Cada vez que el usuario escribe `/` en el cuadro de entrada, se activa este sistema. Tiene tres roles:

1. **Capa de control de UI**: opera directamente el estado de la interfaz de terminal sin pasar por el LLM (como `/clear`, `/theme`, `/vim`)
2. **Capa de gestión de sesiones**: gestiona el historial de conversación, la compresión del contexto y la restauración (como `/compact`, `/resume`, `/branch`)
3. **Capa de extensión de capacidades**: delega tareas complejas al modelo para su ejecución, a través del mecanismo de expansión de prompts (como `/review`, `/skills`)

El diseño del límite del sistema de comandos refleja una clara separación de responsabilidades: los comandos son responsables de "disparar", las herramientas (Tools) de "ejecutar", el LLM de "decidir". Un comando `/review` no llamará directamente a git, sino que inyectará el prompt de revisión en el flujo de conversación, dejando que el modelo impulse la cadena de llamadas a herramientas posterior.

---

## 7.2 Fundamentos Teóricos

### Patrón Command (Patrón de Comando)

El diseño del sistema se alinea estrechamente con el patrón Command clásico de GoF:

- **Interfaz Command**: tipo de unión `Command` (`PromptCommand | LocalCommand | LocalJSXCommand`), encapsulación unificada de solicitudes
- **ConcreteCommand**: cada archivo `commands/<name>/index.ts` es una implementación de comando concreta
- **Invoker**: `processSlashCommand` del REPL es responsable de la distribución para ejecución
- **Receiver**: `ToolUseContext` (estado de conversación), `AppState` (estado de aplicación) son los objetos operados

Pero Claude Code hace dos extensiones clave al patrón clásico:

**Carga perezosa**: los comandos se implementan mediante carga diferida a través de `load(): Promise<Module>`, en lugar de instanciarse inmediatamente al registrarse. Esto distribuye el costo de inicio a la primera llamada; para comandos con dependencias pesadas (como el módulo de renderizado HTML de 113KB de `/insights`), esto es significativo.

**Valores de retorno tipados**: los comandos no son acciones sin valor de retorno (void), sino que devuelven resultados estructurados (`LocalCommandResult`), dejando que el REPL de nivel superior decida cómo renderizarlos, logrando el desacoplamiento entre ejecución y presentación.

### Patrones de Diseño para el Procesamiento de Comandos REPL

El procesamiento de comandos REPL en Claude Code sigue dos principios centrales:

**Immediate vs. Queued**: el campo `immediate?: boolean` en el objeto de comando determina si el comando omite la cola de mensajes para ejecutarse inmediatamente. Las operaciones de interfaz como `/clear`, `/exit` necesitan respuesta inmediata, mientras que operaciones como `/compact` que involucran llamadas al API entran en la cola para procesarse ordenadamente.

**Disponibilidad controlada por auth**: a diferencia de los feature flags en tiempo de ejecución (`isEnabled()`), el campo `availability` tiene efecto en la fase de filtrado de la lista de comandos, asegurando que los usuarios no autorizados ni siquiera vean la existencia de ciertos comandos (como comandos solo para suscriptores de claude.ai).

---

## 7.3 Mecanismo de Registro de Comandos

### Flujo de Registro de commands.ts

La lógica central del registro de comandos está concentrada en `commands.ts` (754 líneas), dividida en cuatro niveles:

**Primer nivel: Comandos integrados estáticos**

```typescript
// commands.ts:240-310 (fragmento central)
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... aproximadamente 60 comandos integrados
])
```

La función `COMMANDS` está envuelta en `memoize` en lugar de ser una constante a nivel de módulo, porque algunos comandos necesitan leer archivos de configuración al registrarse, y la configuración aún no está disponible durante la inicialización del módulo.

**Segundo nivel: Comandos condicionales por feature flag**

```typescript
// commands.ts:68-112 (fragmento de imports condicionales)
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Estos comandos se someten a dead code elimination mediante la función `feature()` de `bun:bundle`, recortando directamente los comandos no habilitados en tiempo de compilación en lugar de juzgar en tiempo de ejecución.

**Tercer nivel: Comandos solo para uso interno**

```typescript
// commands.ts:197-222 (INTERNAL_ONLY_COMMANDS)
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, ultraplan, subscribePr, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
].filter(Boolean)
```

Estos comandos se registran solo cuando `USER_TYPE === 'ant'` (usuarios internos de Anthropic) y no están en modo demo; son el mecanismo de aislamiento para herramientas internas y comandos de depuración.

**Cuarto nivel: Comandos cargados dinámicamente**

```typescript
// commands.ts:360-395 (loadAllCommands)
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

Los tres tipos de Skills, comandos de plugins y comandos de flujo de trabajo se cargan asincrónicamente en paralelo, ordenados por prioridad: los bundled skills tienen la mayor prioridad, y los comandos integrados la menor. Esto asegura que los comandos personalizados del usuario puedan sobrescribir (shadow) los comandos integrados con el mismo nombre.

### Definición de Tipos de Comandos

`types/command.ts` define tres tipos de comandos mutuamente excluyentes que forman el tipo de unión `Command`:

```typescript
// types/command.ts (tipo de unión central)
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| Tipo | Descripción | Comandos típicos |
|------|------|----------|
| `PromptCommand` | Se expande en prompt inyectado en el flujo de conversación, ejecutado por el modelo | `/review`, `/skills`, todos los Skills |
| `LocalCommand` | Ejecución local síncrona pura, devuelve resultado de texto | `/compact`, `/context` |
| `LocalJSXCommand` | Renderiza componente UI React de Ink | `/model`, `/resume`, `/config` |

`CommandBase` es el conjunto de campos base compartidos por los tres:

```typescript
// types/command.ts (campos centrales de CommandBase)
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // verificación de feature flag en tiempo de ejecución
  isHidden?: boolean                      // oculto en typeahead
  argumentHint?: string                   // texto de sugerencia de parámetros
  whenToUse?: string                      // descripción del escenario de llamada del modelo
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // omitir cola y ejecutar inmediatamente
  isSensitive?: boolean                   // parámetros anonimizados del historial
}
```

### Clasificación de Comandos (Integrados vs. Plugin vs. Definidos por Usuario)

```
Jerarquía de fuentes de comandos (prioridad de mayor a menor)
├── bundledSkills        # Skills integrados empaquetados con Claude Code
├── builtinPluginSkills  # Skills proporcionados por plugins integrados habilitados
├── skillDirCommands     # Skills en el directorio .claude/skills/ del usuario
├── workflowCommands     # Comandos de scripts de flujo de trabajo (feature: WORKFLOW_SCRIPTS)
├── pluginCommands       # Comandos registrados por plugins de terceros
├── pluginSkills         # Skills proporcionados por plugins de terceros
└── COMMANDS()           # Comandos integrados codificados (menor prioridad)
```

---

## 7.4 Catálogo Completo de Clasificación de Comandos

Lo siguiente está organizado a partir de la salida `ls` de `commands.ts` y la lista de registro.

### Gestión de Sesiones

| Comando | Descripción |
|------|------|
| `/compact [instrucciones]` | Comprime el historial de conversación, libera la ventana de contexto |
| `/resume` | Selecciona y restaura una conversación de la lista de sesiones históricas |
| `/branch [título]` | Bifurca una nueva sesión desde la conversación actual |
| `/rewind` | Retrocede a un nodo histórico en la conversación |
| `/clear` | Limpia el registro de conversación actual |
| `/session` | Muestra información de la sesión actual |
| `/rename` | Renombra la sesión actual |
| `/summary` | Genera un resumen de la conversación actual (comando interno) |
| `/export` | Exporta contenido de la conversación |
| `/copy` | Copia el último mensaje al portapapeles |

### Herramientas de Desarrollo

| Comando | Descripción |
|------|------|
| `/review [PR#]` | Revisión de código local (llama a `gh pr diff`) |
| `/ultrareview [PR#]` | Revisión de código profunda en la nube (10-20 minutos, impulsada por bughunter) |
| `/commit` | Confirmar cambios de código (comando interno) |
| `/commit-push-pr` | Confirmar + Push + Crear PR (comando interno) |
| `/diff` | Ver el git diff actual |
| `/init` | Inicializar proyecto (generar CLAUDE.md) |
| `/add-dir` | Añadir directorio de trabajo adicional |
| `/hooks` | Gestionar configuración de event hooks |
| `/files` | Listar archivos rastreados en la sesión |
| `/pr_comments` | Ver comentarios del PR |
| `/issue` | Crear/ver GitHub Issue (comando interno) |
| `/autofix-pr` | Corregir automáticamente problemas en el PR (comando interno) |

### Configuración

| Comando | Descripción |
|------|------|
| `/model [nombre]` | Cambiar modelo de conversación (con selector interactivo) |
| `/config` | Ver/modificar elementos de configuración |
| `/theme` | Cambiar tema de terminal |
| `/vim` | Cambiar al modo de entrada vim |
| `/keybindings` | Gestionar atajos de teclado |
| `/permissions` | Ver/modificar permisos de herramientas |
| `/privacy-settings` | Gestionar configuración de privacidad |
| `/output-style` | Establecer preferencia de formato de salida |
| `/effort` | Establecer nivel de esfuerzo de respuesta |
| `/fast` | Cambiar al modo rápido |
| `/plan` | Cambiar al modo Plan (solo planificar, sin ejecutar) |
| `/sandbox-toggle` | Cambiar modo sandbox |

### Depuración y Diagnóstico

| Comando | Descripción |
|------|------|
| `/doctor` | Diagnosticar problemas de configuración y entorno |
| `/cost` | Mostrar el consumo de tokens y costo de la sesión actual |
| `/context` | Mostrar detalles de uso de la ventana de contexto (por categoría en tablas) |
| `/stats` | Mostrar estadísticas de uso |
| `/usage` | Mostrar información de uso del API |
| `/insights` | Generar informe de análisis de uso de sesiones históricas (carga perezosa de módulo de 113KB) |
| `/heapdump` | Generar instantánea del heap de memoria (para depuración) |
| `/debug-tool-call` | Depurar llamadas a herramientas (comando interno) |
| `/perf-issue` | Registrar problema de rendimiento (comando interno) |
| `/ant-trace` | Rastreo interno de Anthropic (comando interno) |

### Identidad y Servicios

| Comando | Descripción |
|------|------|
| `/login` | Iniciar sesión en cuenta Claude.ai |
| `/logout` | Cerrar sesión |
| `/upgrade` | Actualizar a un plan superior |
| `/install-github-app` | Instalar GitHub App |
| `/install-slack-app` | Instalar Slack App |
| `/ide` | Gestión de integración IDE |
| `/terminalSetup` | Configuración de integración de terminal |
| `/mobile` | Mostrar código QR de conexión móvil |
| `/chrome` | Gestión de extensión Chrome |
| `/desktop` | Gestión de aplicación de escritorio |

### Funcionalidades Avanzadas

| Comando | Descripción |
|------|------|
| `/mcp` | Gestión de servidores MCP (listar/iniciar/reiniciar) |
| `/skills` | Gestión de Skills (listar/instalar/actualizar) |
| `/tasks` | Gestión de tareas en segundo plano |
| `/agents` | Gestión de sub-agentes |
| `/memory` | Gestión de archivos de memoria del proyecto (CLAUDE.md) |
| `/plan` | Entrar en modo de planificación |
| `/thinkback` | Remontar el proceso de pensamiento del modelo |
| `/thinkback-play` | Reproducir animación de retrospectiva del pensamiento |
| `/advisor` | Modo de asesor IA |
| `/plugin` | Gestión de plugins |
| `/reload-plugins` | Recargar plugins |
| `/passes` | Gestión de passes de revisión multi-ronda |
| `/feedback` | Enviar comentarios a Anthropic |
| `/btw` | Añadir mensaje de anotación |
| `/tag` | Etiquetar la conversación |
| `/stickers` | Mostrar pegatinas (función de easter egg) |

Comandos condicionales por feature flag (invisibles por defecto):

| Comando | Feature Flag | Descripción |
|------|-------------|------|
| `/ultraplan` | `ULTRAPLAN` | Planificación ultra en la nube (largo plazo asíncrono) |
| `/voice` | `VOICE_MODE` | Modo de entrada de voz |
| `/bridge` | `BRIDGE_MODE` | Modo de puente de control remoto |
| `/workflows` | `WORKFLOW_SCRIPTS` | Comandos de flujo de trabajo de scripts |
| `/peers` | `UDS_INBOX` | Comunicación de sesiones par |
| `/fork` | `FORK_SUBAGENT` | Crear explícitamente un sub-agente |
| `/buddy` | `BUDDY` | Modo de colaboración Buddy |

---

## 7.5 Flujo de Ejecución de Comandos

### Ruta Completa desde la Entrada "/" del Usuario hasta la Ejecución del Comando

```
El usuario escribe "/compact some instructions"
        │
        ▼
    Procesador de entrada REPL
    Detecta prefijo "/"
        │
        ▼
    getCommands(cwd)                    ← agregar lista de comandos de todas las fuentes
    findCommand("compact", commands)     ← buscar por name / aliases
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← verificar control de tipo auth
    isCommandEnabled(cmd)               ← verificar feature flag / isEnabled()
        │
        ├── verificar cmd.immediate      ← true: omitir cola y ejecutar inmediatamente
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        devuelve LocalCommandResult
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        renderiza componente React al terminal
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   devuelve ContentBlockParam[]
                                   inyectado en flujo de conversación → activa razonamiento del modelo
```

### Análisis de Argumentos del Comando

El sistema de comandos no tiene un framework de análisis de argumentos unificado integrado — esta es una elección de diseño intencional. Cada comando maneja su propio parámetro `args: string`, manteniendo gran flexibilidad:

- `/compact` usa directamente `args.trim()` como instrucción de compresión personalizada
- `/review` usa `/^\d+$/.test(prNumber)` para determinar si es un número de PR
- `/model` cuando tiene args va por `SetModelAndClose` para configurar directamente; sin args renderiza el `ModelPickerWrapper` interactivo
- `/resume` soporta ID de sesión (UUID), título personalizado, o cuando no hay parámetro abre el selector de lista

Este diseño evita la complejidad de una capa de análisis unificada, a costa de que cada comando necesite manejar sus propios casos extremos.

### Renderizado de Salida del Comando

Los tres tipos de `LocalCommandResult` corresponden a diferentes rutas de renderizado:

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // renderizado como mensaje de texto
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // activa lógica de reemplazo de contexto
  | { type: 'skip' }                      // no renderiza nada
```

`LocalJSXCommand` pasa resultados al REPL mediante el callback `onDone()`:

```typescript
// types/command.ts (LocalJSXCommandOnDone)
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // modo de visualización del mensaje
    shouldQuery?: boolean                   // si disparar inmediatamente una consulta al modelo
    metaMessages?: string[]                 // mensajes visibles al modelo pero no al usuario
    nextInput?: string                      // rellenar automáticamente la próxima entrada
    submitNextInput?: boolean               // si enviar automáticamente
  },
) => void
```

`display: 'system'` significa mostrar en estilo de mensaje de sistema (gris cursiva), `display: 'user'` muestra como mensaje de usuario normal, `display: 'skip'` no muestra nada.

---

## 7.6 Análisis Profundo de Comandos Representativos

### Detalles de Implementación del Comando /compact

`/compact` es uno de los comandos con lógica más compleja en el sistema de comandos, con la responsabilidad central de comprimir el historial de conversación.

**Árbol de decisión de ejecución** (`commands/compact/compact.ts`):

```
/compact [instrucciones]
    │
    ├── ¿Hay instrucciones personalizadas?
    │   └── Sin instrucciones → trySessionMemoryCompaction()   ← intentar primero la compresión de Session Memory
    │                            Si tiene éxito devuelve directamente, ruta más rápida
    │
    ├── isReactiveOnlyMode() ?
    │   └── Sí → compactViaReactive()               ← ruta de compresión reactiva (nueva arquitectura)
    │               Paralelo: executePreCompactHooks + getCacheSharingParams
    │               Llamada: reactiveCompactOnPromptTooLong()
    │
    └── No → ruta de compresión tradicional
              microcompactMessages()                  ← primero micro-comprimir para reducir tokens
              compactConversation()                   ← compresión principal (generación de resumen)
              setLastSummarizedMessageId(undefined)   ← resetear puntero de seguimiento
```

Punto de diseño clave: antes de la compresión se debe llamar a `getMessagesAfterCompactBoundary(messages)` para filtrar los mensajes ya recortados que el REPL conserva para el scrollback de UI — estos mensajes no deben aparecer en el resumen.

La secuencia de limpieza fija después de una compresión exitosa es:
1. `setLastSummarizedMessageId(undefined)` — resetear puntero de mensajes
2. `suppressCompactWarning()` — suprimir advertencia de "contexto a punto de agotarse"
3. `getUserContext.cache.clear?.()` — limpiar caché del contexto del usuario
4. `runPostCompactCleanup()` — activar hooks de post-compresión

**La ruta Reactive Compact** aprovecha la optimización paralela:

```typescript
// compact.ts:compactViaReactive (segmento paralelo central)
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // ejecutar hooks pre-compresión (puede iniciar sub-procesos)
  getCacheSharingParams(context, messages),  // construir system prompt (recorrer todas las herramientas)
])
```

Los dos son independientes entre sí; la ejecución en paralelo reduce significativamente el tiempo de espera.

### Lógica de Cambio de Modelo del Comando /model

`/model` es de tipo `local-jsx`, renderizando un selector interactivo mediante componentes React.

**Dos rutas de ejecución**:

- **Con parámetros** (`/model claude-sonnet-4-6`): renderiza el componente `SetModelAndClose`, ejecuta la validación del modelo de forma asíncrona en `useEffect`, completa inmediatamente mediante `onDone()`
- **Sin parámetros** (`/model`): renderiza el componente `ModelPickerWrapper`, muestra la interfaz interactiva completa de `ModelPicker`

**Actualización de estado al cambiar modelo**:

```typescript
// model.tsx:handleSelect (actualización de estado central)
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // limpiar cobertura temporal a nivel de sesión
}))
```

**Jerarquía de validación del modelo** (de más rápida a más lenta):
1. Verificar `isModelAllowed(model)` — lista blanca de restricciones de organización
2. Verificar `isOpus1mUnavailable(model)` — verificación de privilegio de contexto 1M
3. Verificar `isKnownAlias(model)` — alias conocido pasa directamente (omite validación API)
4. `validateModel(model)` — llamar al API para validar nombre de modelo personalizado

Fast Mode y el cambio de modelo tienen una vinculación: si el nuevo modelo no soporta Fast Mode, se desactivará automáticamente; si soporta y ya está habilitado, mostrará "Fast mode ON" en el mensaje de confirmación.

### Flujo de Revisión de Código del Comando /review

`/review` demuestra el uso típico del tipo `PromptCommand` — usa una plantilla de prompt concisa para impulsar un proceso de revisión completo:

```typescript
// review.ts:LOCAL_REVIEW_PROMPT (plantilla de prompt completa)
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

El propio comando tiene solo 4 líneas de código clave; el resto lo completa el modelo — esta es exactamente la filosofía de diseño de `PromptCommand`: **el comando define el QUÉ, el modelo decide el CÓMO**.

En contraste, `/ultrareview` (tipo `local-jsx`) ejecuta una ruta completamente diferente:

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← verificar cuota gratuita / saldo Extra Usage
    │   ├── Team/Enterprise → pasa directamente
    │   ├── Tiene usos gratuitos → pasa, con aviso
    │   └── Cuota agotada → mostrar diálogo de confirmación de exceso
    │
    └── launchRemoteReview()
        ├── Modo PR → teleportToRemote(branchName: "refs/pull/N/head")
        └── Modo rama → git merge-base → verificación git diff → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → devuelve URL de tarea, modelo notifica al usuario
```

`/ultrareview` "teletransporta" la tarea de revisión de código para ejecutarla en la nube; después de registrar `RemoteAgentTask` localmente regresa inmediatamente, recibiendo resultados mediante un mecanismo de sondeo — es un patrón de delegación de tareas asíncrono, completamente diferente del modelo de ejecución síncrona del comando local.

---

## 7.7 El Límite entre Comandos y Skills

### Similitudes y Diferencias entre Ambos

| Dimensión | Comando (Command) | Skill |
|------|---------------|-------|
| Definición | Código TypeScript, lógica codificada | Archivo Markdown, frontmatter + contenido de prompt |
| Momento de carga | Registro estático al inicio (integrado) o carga asíncrona (plugin) | Escaneo del sistema de archivos en tiempo de ejecución |
| Tipo de ejecución | `local` / `local-jsx` / `prompt` | Solo `prompt` (expandido a prompt) |
| Invocable por modelo | La mayoría de los comandos integrados prohíben la invocación del modelo (`source: 'builtin'`) | Diseñado para soporte del modelo mediante llamada a SkillTool |
| Visibilidad para el usuario | Todos los comandos aparecen en `/` typeahead | Depende de `userInvocable` y `hasUserSpecifiedDescription` |
| Conciencia del contexto | Accede al estado completo de la aplicación mediante `ToolUseContext` | Solo puede usar contenido de prompt, sin acceso directo al estado |
| Identificador de fuente | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### Consideraciones Detrás de las Decisiones de Diseño

**¿Por qué los comandos integrados no usan Markdown Skill?**

Los comandos integrados necesitan acceder al estado de la aplicación (`AppState`), llamar a APIs de Node.js (sistema de archivos, cifrado) y renderizar componentes React — estas capacidades van mucho más allá de lo que una plantilla de prompt puede expresar. `/compact` necesita llamar a 4 estrategias de compresión diferentes; `/model` necesita renderizar una UI interactiva; `/resume` necesita leer y escribir archivos de sesión. Todo esto debe ser código.

**La lógica de filtrado de SkillTool** revela la delimitación precisa del límite:

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← los comandos integrados se excluyen
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`** es la regla central: los comandos integrados se excluyen explícitamente de la lista invocable por el modelo. Esto evita que el modelo use SkillTool para omitir las verificaciones de permisos y operar directamente el estado de sesión.

**El conjunto de comandos remotamente seguros (REMOTE_SAFE_COMMANDS)** refina aún más este límite:

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

Solo 20 comandos están disponibles en modo `--remote` — estos comandos no dependen del sistema de archivos local, git ni IDE; son operaciones de estado TUI puras que pueden ejecutarse de forma segura en sesiones de bridge remotas.

---

## 7.8 Análisis de Decisiones de Diseño

**Decisión 1: Tres tipos de comandos en lugar de interfaz unificada**

La tricotomía `local` / `local-jsx` / `prompt` puede parecer que aumenta la complejidad, pero cada tipo resuelve un problema central diferente:
- `local` maneja operaciones con efectos secundarios pero sin UI (necesita devolver datos estructurados)
- `local-jsx` maneja operaciones que requieren interfaz interactiva (dependiendo del árbol de renderizado de Ink)
- `prompt` maneja operaciones que pueden delegarse al modelo (menor acoplamiento)

Si se forzara una interfaz única, todos los comandos tendrían que manejar el renderizado de React (dependencia innecesaria) o se perdería la seguridad de tipos.

**Decisión 2: memoize por cwd en lugar de singleton global**

`loadAllCommands = memoize(async (cwd: string) => ...)` usa el directorio de trabajo como clave de caché, lo que significa que las instancias de Claude Code en diferentes directorios tienen cachés de comandos independientes. Esto soporta la necesidad de que cada directorio tenga su propio conjunto de Skills en escenarios de monorepo y proyectos múltiples.

**Decisión 3: No hacer análisis unificado de argumentos**

Este es un "diseño relajado" intencional. Un framework de análisis unificado (como commander.js) obligaría a cada comando a declarar un schema completo de parámetros, lo que no tiene sentido para comandos de "instrucciones en texto libre" como `/compact`. Mantener la cadena de texto sin procesar para que cada comando decida cómo analizarla intercambia flexibilidad por consistencia.

**Decisión 4: Control de dos capas de Availability vs. isEnabled**

El control de dos capas resuelve problemas de visibilidad en diferentes ciclos de vida:
- `availability` filtra en el momento de construcción de la lista de comandos, el resultado se cachea; apropiado para verificación de tipo auth estática
- `isEnabled()` se reevalúa en cada llamada a `getCommands()` (sin caché); apropiado para verificación de feature flag dinámica

El comentario explica especialmente que `isEnabled()` no se memoize porque: después de ejecutar `/login` el estado de auth cambia y debe reflejarse inmediatamente en la lista de comandos.

**Decisión 5: Los comandos internos no se gestionan en paquetes separados**

`INTERNAL_ONLY_COMMANDS` controla la visibilidad directamente mediante la variable de entorno `USER_TYPE === 'ant'`, en lugar de mediante paquetes npm separados. Esto simplifica la complejidad de la compilación, a costa de que las compilaciones externas necesiten recortar esta parte del código mediante dead code elimination (el `filter(Boolean)` también funciona de forma efectiva para los comandos condicionales `null`).

---

## 7.9 Patrones Transferibles

### Patrón 1: Tricotomía de Tipos de Comandos

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**Escenario de aplicación**: cualquier sistema de comandos que necesite soportar simultáneamente "lógica pura", "interacción UI" y "delegación al LLM". Los límites de los tres son muy claros y pueden portarse directamente a otros frameworks REPL/CLI.

**Valor central**: el sistema de tipos fuerza la separación de responsabilidades, sin necesidad de comprobaciones isinstance en tiempo de ejecución.

### Patrón 2: Carga Perezosa + Memoize por Cwd

```typescript
const loadAllCommands = memoize(async (cwd: string) => ...)
```

**Escenario de aplicación**: cualquier sistema de plugins en el que el contenido del plugin varíe según el directorio de trabajo. El cwd como clave de caché hace que el sistema soporte naturalmente múltiples proyectos con diferentes conjuntos de plugins.

### Patrón 3: Resultado de Comando Tipado

```typescript
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult }
  | { type: 'skip' }
```

**Escenario de aplicación**: cualquier comando que necesite devolver resultados al REPL para decidir cómo renderizar. Los resultados tipados son superiores a los void callbacks: el código del llamador puede verificar el tipo de resultado y manejar de acuerdo a ello, en lugar de depender de efectos secundarios.

### Patrón 4: Control de dos capas Static + Dynamic

La distinción entre `availability` (estático, cacheado) e `isEnabled()` (dinámico, sin caché) es el patrón correcto para gestionar la visibilidad de comandos en múltiples dimensiones.

**Escenario de aplicación**: cualquier sistema de registro de características (feature registry) en el que algunas características se determinan por el estado del usuario (estático a largo plazo) y otras por feature flags (dinámicos, pueden cambiar).
