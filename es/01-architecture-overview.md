# Capítulo 1: Visión General de la Arquitectura y Flujo de Inicio

> Fuente de datos: Instantánea del código fuente TypeScript de Claude Code (2026-03-31)
> Ruta del código fuente (mini): `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 Descripción General y Posicionamiento

**¿Qué es Claude Code?** Claude Code es un asistente de programación AI que se ejecuta en la terminal, renderiza una TUI (Terminal User Interface) interactiva mediante React/Ink, y usa un bucle REPL para impulsar el API de Claude y completar tareas de desarrollo como edición de código, ejecución de comandos y operaciones de archivos.

### Resumen del Stack Tecnológico

| Capa | Tecnología | Uso |
|------|------|------|
| Runtime | Bun (principal) / Node.js 18+ (compatible) | Entorno de ejecución JavaScript |
| Lenguaje | TypeScript | Tipos estrictos en todo el proyecto |
| Framework UI | React + Ink | Renderizado TUI de terminal |
| Framework CLI | Commander.js (`@commander-js/extra-typings`) | Análisis de argumentos de línea de comandos |
| Cliente API | `@anthropic-ai/sdk` | Llamadas al API de Claude |
| Integración MCP | `@modelcontextprotocol/sdk` | Protocolo de servidor MCP |
| Feature flags | GrowthBook + `bun:bundle` feature flags | Pruebas A/B y DCE |
| Telemetría | OpenTelemetry (carga perezosa ~400KB) | Métricas/registros/trazas |
| Validación | Zod v4 | Validación de schema en tiempo de ejecución |

### Estadísticas de Tamaño del Código

- **Líneas totales**: 512,664 (archivos `.ts` + `.tsx`)
- **Número de archivos**: 1,884 archivos TypeScript
- **Directorios de nivel superior**: 35

Distribución de LOC por directorio principal:

```
utils/       180,472 líneas  (35.2%)  — funciones utilitarias, permisos, auth, configuración, etc.
components/   81,546 líneas  (15.9%)  — componentes UI de React
services/     53,680 líneas  (10.5%)  — servicios API, MCP, análisis, memoria, etc.
tools/        50,828 líneas  (9.9%)   — 30 implementaciones de herramientas (Bash/File/Agent, etc.)
commands/     26,428 líneas  (5.2%)   — implementaciones de comandos slash
screens/       5,977 líneas  (1.2%)   — pantallas de nivel superior como REPL
bootstrap/     ~5,000 líneas  (1.0%)  — estado global (state.ts 1,758 líneas)
entrypoints/   ~3,000 líneas  (0.6%)  — puntos de entrada CLI/SDK/MCP
main.tsx       4,683 líneas  (0.9%)   — coordinador del punto de entrada principal
setup.ts         477 líneas  (0.1%)   — configuración de inicialización
```

---

## 1.2 Fundamentos Teóricos

### Patrones Arquitectónicos de Aplicaciones de Línea de Comandos

Claude Code fusiona dos patrones clásicos de arquitectura CLI:

**Bucle REPL (Read-Eval-Print Loop)**
El REPL tradicional lee entrada, evalúa y muestra salida en un bucle síncrono. Claude Code lo actualiza a un REPL asíncrono impulsado por eventos: la entrada es capturada por componentes React, la "evaluación" es un round-trip al API de Claude (con múltiples rondas de llamadas a herramientas), y la salida se renderiza al terminal mediante el reconciliador de React/Ink.

**Event-Driven Architecture**
Al inicio, no bloquea esperando que toda la inicialización se complete: la lectura MDM, la precarga de Keychain, las conexiones MCP y la carga de hooks de plugins se activan en paralelo con fire-and-forget (ver sección 1.4). Esto minimiza el TTFR (Time To First Render), coherente con la filosofía de optimización de la Ruta de Renderizado Crítica de las aplicaciones web.

### Filosofía de Diseño del Framework de UI de Terminal: React in Terminal

Ink porta el modelo de componentes de React, el estado declarativo y el mecanismo de reconciliación al terminal. El concepto central:

- **Virtual DOM → Búfer de terminal virtual**: cada cambio de estado activa un diff, solo redibuja las líneas de caracteres modificadas, evitando parpadeos
- **Flexbox → Maquetación de terminal**: usa el motor CSS Yoga para calcular anchos de columna y saltos de línea, permitiendo describir la UI del terminal de forma declarativa con JSX
- **Reutilización de componentes**: el spinner de carga, los diálogos de confirmación, la visualización de diffs y otras lógicas de UI se encapsulan como componentes React testables

Esto permite que el código de UI de Claude Code comparta el marco cognitivo con el código de front-end web; las 81,546 líneas de código en el directorio `components/` se pueden entender con patrones React familiares.

### Fundamentos Teóricos de la Arquitectura de Plugins

El sistema de plugins de Claude Code se basa en el modelo de "Registro de Capacidades" (Capability Registration Pattern):

- Herramientas (Tools), Comandos (Commands) y Hooks se registran en un registro global durante el inicio
- Los plugins extienden la lista de herramientas/comandos mediante convenciones del sistema de archivos (`~/.claude/plugins/`)
- La función `feature()` de `bun:bundle` realiza Dead Code Elimination (DCE) en tiempo de compilación; las funciones experimentales no aparecerán en los artefactos de compilación externos

---

## 1.3 Diagrama de Arquitectura General

### Arquitectura en Capas (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                 Capa de Entrada (Entry Layer)            │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts│
│  (CLI int.)   (enrutamiento Commander.js) (modo servidor MCP) │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│               Capa de Bootstrap (Bootstrap Layer)        │
│    setup.ts      │    entrypoints/init.ts                │
│  (inicializ. sesión)│  bootstrap/state.ts                │
│  (worktree/tmux)  │    (singleton de estado global)      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│               Capa UI (Ink/React TUI)                    │
│  screens/REPL.tsx  │  components/App.tsx                 │
│  (interfaz principal)│  components/ (81K LOC)            │
│  replLauncher.tsx  │  (entrada/salida/diálogos/animaciones)│
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│               Capa de Motor (Engine Layer)               │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts  │
│  (gest. ciclo vida ses.)│(llamadas API)│(árbol de estado React)│
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│             Capa de Herramientas (Tool Layer)            │
│  tools/ (30 herramientas, 50K LOC)                       │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool       │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│             Capa de Servicios (Service Layer)            │
│  services/ (53K LOC)                                     │
│  api/         │ mcp/          │ analytics/                │
│  (Claude API)   (cliente MCP)  (GrowthBook/OTel)         │
│  lsp/         │ SessionMemory │ remoteManagedSettings     │
│  (servidor LSP) (memoria sesión) (config. empresa)       │
└─────────────────────────────────────────────────────────┘
```

### Resumen de Dependencias entre Módulos

```
main.tsx
  ├── entrypoints/init.ts       (memoized, solo se inicializa una vez)
  ├── entrypoints/cli.tsx       (enrutamiento de sub-comandos Commander)
  ├── bootstrap/state.ts        (estado global, prohibido ciclos de dependencia)
  ├── setup.ts                  (llamado en cada sesión)
  ├── QueryEngine.ts            (ruta headless/SDK)
  ├── replLauncher.tsx          (ruta interactive)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (carga de herramientas/recursos MCP)
```

**La posición especial de `bootstrap/state.ts`**: el código tiene un comentario explícito `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`, y existe una regla ESLint `custom-rules/bootstrap-isolation` que impide que este archivo sea importado por módulos no-hoja, previniendo dependencias circulares.

### Comparación de los Tres Puntos de Entrada

| Entrada | Archivo | Forma de activación | Características |
|------|------|----------|------|
| CLI interactiva | `entrypoints/cli.tsx` | Comando `claude` | REPL completo + React TUI |
| SDK sin cabeza | `QueryEngine.ts` | Flag `-p` / SDK API | Sin UI, salida única o streaming |
| Servidor MCP | `entrypoints/mcp.ts` | `claude --mcp` | Expone conjunto de herramientas como servidor MCP |

---

## 1.4 Análisis Detallado del Flujo de Inicio

### Secuencia Completa de Inicio de main.tsx

Las 4,683 líneas de `main.tsx` no se ejecutan secuencialmente: los efectos secundarios de import en la parte superior del archivo son una secuencia cuidadosamente orquestada de precalentamiento en paralelo.

**Fase 0: Período de carga de módulos (efectos secundarios de import, ~135ms)**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. Punto de inicio de referencia de rendimiento

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. Paralelo: subproceso MDM (plutil/reg query)

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. Paralelo: prelectura de macOS Keychain (OAuth + clave API)

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // todos los imports completos
```

Los comentarios explican con precisión el beneficio de estas tres operaciones en paralelo: la lectura MDM ahorra ~135ms de tiempo de evaluación de módulos, la prelectura de Keychain ahorra ~65ms de spawn síncrono secuencial. Este es el truco central de optimización de inicio de Claude Code: **aprovechar las características de análisis estático de los módulos ES para ejecutar operaciones intensivas en I/O durante el período de evaluación del grafo de módulos**.

**Fase 1: Enrutamiento Commander (síncrono)**

En `entrypoints/cli.tsx`, Commander.js analiza argv y distribuye a diferentes rutas de ejecución según el sub-comando (`chat`, `api`, `mcp`, `resume`, etc.) o flag:

```typescript
// entrypoints/cli.tsx (estructura simplificada)
async function main(): Promise<void> {
  // Ruta rápida: --version cero imports
  // Ruta normal: await init() → setup() → ejecución ramificada
}
```

**Fase 2: Inicialización init() (memoized, ejecutada solo una vez)**

La función `init` en `entrypoints/init.ts` está envuelta en `memoize`, garantizando que múltiples llamadas solo inicialicen una vez:

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // activación del sistema de configuración
  applySafeConfigEnvironmentVariables()  // variables de entorno seguras antes de conversación de confianza
  applyExtraCACertsFromConfig()     // configurar CA certs antes de cualquier conexión TLS
  setupGracefulShutdown()           // registrar hooks de limpieza al salir
  // carga perezosa: OpenTelemetry (~400KB) + gRPC (~700KB)
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // caché asíncrona
  detectCurrentRepository()          // detección de repositorio GitHub
  preconnectAnthropicApi()           // preconexión TCP+TLS (~100-200ms overlap)
  configureGlobalMTLS()
  configureGlobalAgents()            // configuración de proxy
})
```

**Fase 3: Inicialización de sesión setup() (llamada en cada sesión)**

```typescript
// setup.ts — secuencia de pasos clave
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. Servidor de mensajería UDS (modo swarm/ant)
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Verificación de respaldo de terminal (iTerm2/Terminal.app)
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — debe llamarse antes que cualquier código que dependa de cwd
  setCwd(cwd)
  // 4. Instantánea de configuración de Hooks (debe ser después de setCwd())
  captureHooksConfigSnapshot()
  // 5. Creación de Worktree (si --worktree)
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. Registro de tareas en segundo plano (SessionMemory, context collapse)
  if (!isBareMode()) initSessionMemory()
  // 7. Precarga de plugins (paralela, no bloqueante)
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. Activación de sink de análisis + primer evento de telemetría
  initSinks()
  logEvent('tengu_started', {})
  // 9. Verificación de notas de versión (modo interactivo)
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**Fase 4: Renderizado REPL**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // carga perezosa UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

Finalmente, Ink toma el control del terminal, el árbol de componentes React comienza a renderizarse y el REPL está listo.

### Estrategia de Precarga en Paralelo

La optimización de inicio de Claude Code sigue el principio "**cuanto antes se active, más tarde se espera**":

| Operación | Momento de activación | Momento de espera |
|------|----------|----------|
| Subproceso MDM (`plutil/reg query`) | Efecto secundario de import línea 1 de `main.tsx` | Antes de `applySafeConfigEnvironmentVariables()` |
| Prelectura Keychain (OAuth + clave API) | Efecto secundario de import línea 3 de `main.tsx` | `ensureKeychainPrefetchCompleted()` |
| Preconexión TCP al Claude API | `preconnectAnthropicApi()` dentro de `init()` | Reutilización automática de conexión en primera solicitud API |
| Carga de plugin hooks | Fire-and-forget dentro de `setup()` | `processSessionStartHooks()` antes del renderizado |
| Lectura de configs MCP | Inicio de `getClaudeCodeMcpConfigs()` | `getMcpToolsCommandsAndResources()` en modo interactivo |

### Mecanismo de Carga Perezosa

Claude Code aplica carga perezosa explícita a los módulos grandes en la ruta crítica de inicio:

```typescript
// entrypoints/init.ts — comentario de carga perezosa de OpenTelemetry
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

Además, `replLauncher.tsx` solo importa los componentes App y REPL en el último momento, evitando que el árbol React se evalúe antes de que el enrutamiento de Commander esté completo.

La función `feature()` de `bun:bundle` implementa DCE en tiempo de compilación:

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

En las compilaciones externas, este código se elimina completamente, reduciendo el tamaño del paquete.

### Análisis Detallado de los Pasos de Inicialización de setup.ts

Las 477 líneas de `setup.ts` giran en torno a las siguientes restricciones clave:

1. **`setCwd()` debe llamarse primero**: todas las operaciones posteriores (hooks, settings, carga de plugins) dependen del cwd correcto
2. **La instantánea de Hooks debe ser después de `setCwd()`**: garantiza leer `.claude/settings.json` desde el directorio correcto
3. **La creación de Worktree debe ser antes de `getCommands()`**: de lo contrario el comando `/eject` no estará disponible
4. **`initSinks()` después de registrar todas las tareas en segundo plano**: garantiza que la cola de eventos de análisis esté lista

El modo `--bare` (llamadas sin cabeza mediante scripts/SDK) omite muchas funciones interactivas: verificación de respaldo de terminal, precarga de hooks de plugins, atribución de commits, watcher de memoria de equipo, etc., minimizando el costo de inicio de las llamadas mediante scripts.

### Construcción del Estado de bootstrap/state.ts

`state.ts` (1,758 líneas) mantiene el estado singleton global de toda la sesión. El tipo `State` central cubre:

```typescript
// bootstrap/state.ts (definición de tipo State, parcial)
type State = {
  originalCwd: string
  projectRoot: string          // directorio raíz estable del proyecto, worktree no lo cambia
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // contadores de telemetría
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // proveedores de registros/trazas
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... aproximadamente 60 campos en total
}
```

**Restricción de diseño**: el comentario `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` es un guardián arquitectónico. La regla ESLint `custom-rules/bootstrap-isolation` previene que state.ts sea importado por módulos que causarían dependencias circulares. Todo el estado se accede mediante funciones setter/getter, sin exponer objetos mutables directamente.

---

## 1.5 Análisis de Puntos de Entrada

### Entrada CLI (modo interactivo)

`entrypoints/cli.tsx` es la entrada más compleja, responsable del enrutamiento de todas las funciones orientadas al usuario:

**Ruta de inicio**:
1. Commander.js analiza argv → identifica sub-comando o flag
2. `await init()` inicialización (memoized)
3. Procesa configs MCP, políticas empresariales, integración Chrome
4. `await setup(cwd, permissionMode, ...)` inicialización de sesión
5. Ramificación según modo:
   - **Modo interactivo**: `showSetupScreens()` → `launchRepl()` → React TUI
   - **Modo Print (`-p`)**: `runHeadless()` → `QueryEngine` → stdout
   - **Modo Resume**: `loadConversationForResume()` → restaura sesión histórica
   - **Modo Teleport**: toma control de sesión remota

**Opciones CLI clave** (parcial):

| Flag | Función |
|------|------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | configuración dinámica de servidor MCP |
| `--worktree` | crea aislamiento git worktree |
| `--tmux` | ejecuta en sesión tmux |
| `--model` | anula modelo del bucle principal |
| `--resume` | restaura sesión histórica |

### Entrada SDK (API programático)

Al llamar mediante el flag `-p` o el API programático SDK, se omite la React TUI y se entra directamente en `QueryEngine.ts`:

- `isNonInteractiveSession = true`
- Omite todo el renderizado UI (Ink)
- Salida streaming hacia stdout mediante el tipo `SDKMessage`
- Soporta salida estructurada como `SDKStatus`, `SDKPermissionDenial`, `SDKCompactBoundaryMessage`

El modo SDK también tiene features beta exclusivos: `entrypoints/sdk/coreSchemas.ts` define schemas de entrada/salida JSON estructurados, `entrypoints/agentSdkTypes.ts` define tipos exclusivos del SDK como `HookEvent`, `ModelUsage`, etc.

### Entrada MCP (modo servidor MCP)

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools: expone todas las herramientas de Claude Code como herramientas MCP
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool: delega la ejecución a la implementación de herramienta correspondiente
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

El modo MCP expone el conjunto completo de herramientas de Claude Code (BashTool, FileReadTool, GrepTool, etc.) a clientes MCP externos, implementando "Claude Code como servidor MCP".

### Lógica Compartida de los Tres Puntos de Entrada

Independientemente del punto de entrada, todos comparten:
- Estado global de `bootstrap/state.ts`
- Inicialización de `entrypoints/init.ts` (memoized garantiza que solo se ejecute una vez)
- Registro de herramientas `Tool.ts`
- Todos los servicios en `services/` (cliente API, sistema de permisos, etc.)
- Sistema de ciclo de vida de Hooks

La diferencia está en si se renderiza la React TUI y en el formato de salida (texto interactivo vs. JSON estructurado).

---

## 1.6 Análisis de Decisiones de Diseño

### Por Qué Bun en Lugar de Node.js

A partir del código se pueden observar las características de uso de Bun:

1. **Función `feature()` de `bun:bundle`**: este es el mecanismo de feature flag en tiempo de compilación exclusivo de Bun, que soporta Dead Code Elimination. Se usa ampliamente en `main.tsx` (COORDINATOR_MODE, KAIROS, CHICAGO_MCP, UDS_INBOX, etc.); las compilaciones externas eliminan completamente este código experimental.

2. **WebView API de Bun** (referencia condicional): `typeof Bun !== 'undefined' && 'WebView' in Bun`, lo que indica que algunas funciones dependen de APIs exclusivas de Bun.

3. **Ejecutable de archivo único de Bun**: los comentarios mencionan que `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`, lo que indica que el artefacto publicado es un ejecutable de archivo único compilado con Bun.

4. **Rendimiento**: la velocidad de inicio y la velocidad de carga de módulos de Bun son significativamente superiores a las de Node.js, lo cual es crucial para el TTFR de las herramientas CLI.

Al mismo tiempo, se mantiene la compatibilidad con Node.js 18+ (hay verificación de versión de Node en `setup.ts`), para soportar entornos sin Bun (CI, máquinas controladas por empresas).

### Por Qué React/Ink para la UI de Terminal

Las 81,546 líneas de código en el directorio `components/` indican una complejidad UI extremadamente alta. Si se escribiera manualmente con códigos de control ANSI brutos, el costo de mantenimiento sería incontrolable. La elección de React/Ink aporta:

1. **UI declarativa**: la salida en streaming, el estado de ejecución de herramientas, los diálogos de confirmación de permisos, etc., todos pueden ser impulsados por el estado de React, en lugar de control imperativo del cursor
2. **Aislamiento de componentes**: `screens/REPL.tsx` solo necesita preocuparse del diseño general; cada sub-funcionalidad (cuadro de entrada, lista de mensajes, progreso de herramientas) se encapsula individualmente
3. **Amigable para recarga en caliente**: durante el desarrollo se puede depurar con el enfoque estándar de React DevTools
4. **Testabilidad**: los componentes React se pueden probar con pruebas unitarias usando `@testing-library/react`, sin depender de un terminal real

### Filosofía de Optimización de Rendimiento de la Precarga en Paralelo

La optimización de inicio de Claude Code tiene un modelo de prioridad claro: **TTFR (Time To First Render) tiene la máxima prioridad, no "completar toda la inicialización"**.

Manifestación concreta:
- La lectura de Keychain (~65ms) se activa en el primer efecto secundario de import, no cuando se necesita la clave API
- Las conexiones de servidores MCP se realizan en segundo plano en paralelo, el renderizado REPL no espera (el usuario ve la interfaz y luego se completan las conexiones MCP)
- Las notas de versión, la configuración de GrowthBook, los hooks de plugins son todos fire-and-forget

El costo es que se necesita gestionar cuidadosamente las condiciones de carrera de "consumo antes de que la precarga se complete", controladas con precisión mediante puntos de await como `ensureKeychainPrefetchCompleted()`.

### Tradeoff entre Carga Perezosa y Precarga

| Estrategia | Objeto | Razón |
|------|------|------|
| Precarga (efecto secundario de import) | Subproceso MDM, Keychain | Intensivo en I/O, cuanto antes mejor |
| Carga perezosa (`await import()`) | OpenTelemetry (~400KB), gRPC (~700KB), componentes React TUI | Evaluación de módulo costosa, no en ruta crítica |
| Carga condicional (DCE `feature()`) | COORDINATOR_MODE, KAIROS, CHICAGO_MCP | Funciones experimentales, usuarios externos no las necesitan |
| Retardo `setImmediate()` | Hook de atribución de commit | Evitar bloquear el event loop en la ventana de microtareas de setup() |

Esta estrategia por capas hace que Claude Code solo realice "el trabajo mínimo necesario para mostrar la interfaz" durante el inicio.

---

## 1.7 Patrones Transferibles

### Patrón General de Optimización de Inicio

La secuencia de inicio de Claude Code demuestra un framework de optimización de tres capas reutilizable de "**precalentamiento paralelo + carga perezosa + DCE**":

**Patrón 1: Usar efectos secundarios de módulos ES para precalentar I/O**
```typescript
// Insertar fire-and-forget I/O entre declaraciones import
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // activar inmediatamente, sin await
import { SomethingElse } from './other.js'  // carga en paralelo
```
Aplicable a: cualquier dato de inicialización "necesario leer pero lento" (archivos de configuración, credenciales, preconexiones de red).

**Patrón 2: Inicialización única con memoize**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
Aplicable a: lógica de inicialización compartida entre múltiples puntos de entrada, previniendo ejecuciones redundantes.

**Patrón 3: Estratificación del modo `--bare`**
Las llamadas mediante scripts/API no necesitan UI, verificaciones de terminal ni análisis; usar `isBareMode()` para omitirlos rápidamente, manteniendo bajo el costo de las llamadas headless.

**Patrón 4: Separación de estado**
`bootstrap/state.ts` como módulo hoja estricto (sin dependencias circulares), accedido mediante setter/getter, con reglas ESLint para forzar su cumplimiento. Esto permite importar el módulo de estado de forma segura desde cualquier lugar.

### Lecciones para el CLI de Doramagic

Basándose en el análisis anterior, el CLI de Doramagic puede adoptar los siguientes patrones en su diseño arquitectónico:

1. **Separar la ruta crítica de inicio**: distinguir estrictamente entre lo que "debe completarse antes del renderizado" y lo que "puede completarse después del renderizado", documentar las razones con comentarios (estilo del comentario `// ~65ms on every macOS startup` de Claude Code)

2. **Singleton de estado global + patrón de accesores**: basándose en `bootstrap/state.ts`, mantener el estado de sesión en un módulo hoja estricto, evitando que el estado se disperse

3. **Función de inicialización `memoize`**: independientemente del punto de entrada desde el que se llame, garantizar que la inicialización solo se ejecute una vez

4. **Separación de tres modos**: interactive (React TUI) / headless (flag -p) / server (MCP), compartiendo la capa de herramientas y servicios subyacente

5. **Feature flag + DCE**: envolver funciones experimentales con feature flags, eliminadas automáticamente al publicar

---

## 1.8 Índice del Código Fuente

| Archivo | Líneas | Contenido clave |
|------|------|----------|
| `main.tsx` | 4,683 | Punto de entrada principal, enrutamiento Commander, inicialización de estado, ramas interactivo/headless |
| `setup.ts` | 477 | Inicialización de sesión: cwd, hooks, worktree, precarga de plugins |
| `bootstrap/state.ts` | 1,758 | Singleton de estado global, definición del tipo `State`, todos los getters/setters |
| `entrypoints/init.ts` | ~400 | Inicialización global memoized: config, mTLS, proxy, carga perezosa OTel |
| `entrypoints/cli.tsx` | ~2,000 | Enrutamiento Commander.js, ramas interactivo/print/resume/teleport |
| `entrypoints/mcp.ts` | ~200 | Modo servidor MCP, exposición del conjunto de herramientas |
| `entrypoints/sdk/coreSchemas.ts` | - | Schema de entrada/salida estructurado para modo SDK |
| `entrypoints/agentSdkTypes.ts` | - | Tipos exclusivos del SDK (HookEvent, ModelUsage, etc.) |
| `replLauncher.tsx` | ~30 | Carga perezosa App + REPL, inicio de React TUI |
| `QueryEngine.ts` | ~1,500 | Gestión del ciclo de vida de sesión, núcleo de la ruta headless |
| `Tool.ts` | - | Definición de interfaz de herramientas (inputSchema, call, prompt, etc.) |
| `tools/` | 50,828 | 30 implementaciones de herramientas (BashTool/FileEditTool/AgentTool, etc.) |
| `services/api/` | - | Llamadas al Claude API, reintentos, estadísticas de uso |
| `services/mcp/client.ts` | - | Gestión de conexiones de clientes MCP |
| `utils/startupProfiler.ts` | - | Instrumentación de rendimiento `profileCheckpoint()` |
| `utils/secureStorage/keychainPrefetch.ts` | - | Prelectura paralela de macOS Keychain |
| `utils/settings/mdm/rawRead.ts` | - | Lectura paralela de configuración MDM |

### Localización del Código Clave

- **Punto de inicio del precalentamiento paralelo**: `main.tsx:12-20` (3 efectos secundarios de import)
- **Inicialización memoized**: `entrypoints/init.ts:57` (`export const init = memoize(...)`)
- **Tipo de estado global**: `bootstrap/state.ts:30-200` (`type State = {...}`)
- **Definición de servidor MCP**: `entrypoints/mcp.ts:42` (`startMCPServer`)
- **Punto de entrada de renderizado REPL**: `replLauncher.tsx:14` (`launchRepl`)
- **Interfaz de herramientas**: `Tool.ts:1-30` (`ToolInputJSONSchema`, `ToolUseContext`)
- **Orden crítico de setup**: `setup.ts:77-230` (setCwd → captureHooksConfigSnapshot → worktree → tareas en segundo plano)

---

*Palabras del capítulo: aproximadamente 9,800 caracteres | Fecha de la instantánea del código fuente: 2026-03-31*
