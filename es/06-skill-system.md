# Capítulo 6: Sistema de Skills

## 6.1 Descripción General y Posicionamiento

El sistema de Skills es una de las arquitecturas más innovadoras de Claude Code. Codifica flujos de trabajo reutilizables en archivos Markdown y los activa mediante comandos slash (`/skill-name`) o llamadas proactivas de la IA. En esencia, un Skill es un "SOP para la IA" — sedimenta los pasos de ejecución, los puntos de decisión y los criterios de éxito de los expertos humanos en tareas complejas en un formato Markdown estructurado, dotando a la IA de capacidad de ejecución profesional reproducible.

A diferencia de los prompts ordinarios, el sistema de Skills tiene las siguientes características centrales:

1. **Fusión declarativa + ejecutiva**: el Frontmatter declara metadatos (permisos, modelo, condiciones de activación), el cuerpo son instrucciones de ejecución
2. **Carga desde múltiples fuentes**: integrado (bundled), nivel-usuario, nivel-proyecto, nivel-Plugin, fuente MCP, fusionado por prioridad
3. **Dos modos de ejecución**: inline (inyectar en el contexto de la sesión actual) y fork (ejecución aislada en sub-agente independiente)
4. **Activación condicional**: activación automática de Skills según ruta de archivo mediante frontmatter `paths`
5. **Descubrimiento dinámico**: durante la sesión, a medida que el usuario opera archivos, se descubren y cargan automáticamente Skills de directorios más profundos

El sistema de Skills no es un simple alias de comandos, sino un framework completo de orquestación de flujos de trabajo.

---

## 6.2 Fundamentos Teóricos

### Patrón de Diseño de Flujos de Trabajo Reutilizables

El sistema de Skills resuelve un problema central en el uso de herramientas IA: **¿cómo se sedimenta el conocimiento especializado y se reproduce?** La reutilización de código tradicional se hace a través de funciones y clases, pero el "conocimiento" ejecutado por la IA es un flujo de trabajo descrito en lenguaje natural que no puede encapsularse directamente en funciones de código.

El diseño de Skills toma prestada la idea del SOP (Procedimiento Operativo Estándar) — registra estructuradamente el proceso de ejecución del experto, los puntos de decisión y los criterios de éxito, permitiendo que la IA siga la misma ruta de alta calidad en cada ejecución.

### Definición de Flujos de Trabajo Declarativa vs. Imperativa

El sistema de Skills soporta dos estilos simultáneamente:

- **Declarativo**: declarar atributos como `allowed-tools`, `model`, `context` mediante frontmatter, dejando al sistema manejar automáticamente el control de permisos y la configuración del contexto de ejecución
- **Imperativo**: el cuerpo del Skill puede incrustar comandos shell (`!``command``) para ejecución directa, implementando "operaciones mezcladas con instrucciones"

### La Filosofía de Markdown-as-Code

Elegir Markdown en lugar de JSON/YAML como formato del Skill es una decisión de diseño deliberada:

- **Legibilidad humana**: los desarrolladores pueden leer y editar Skills directamente, comprendiendo su intención
- **Naturalmente amigable para la IA**: los datos de entrenamiento de la IA contienen grandes cantidades de Markdown; la IA comprende Markdown más naturalmente que JSON
- **Estructuración progresiva**: puede comenzar con prosa pura, añadiendo gradualmente encabezados, pasos y reglas, sin forzar una estructura completa
- **Compatible con control de versiones**: los diffs de Markdown son amigables para humanos, de un vistazo en revisión de código se ven los cambios del flujo de trabajo

---

## 6.3 Formato del Skill y Estructuras de Datos

### Especificaciones de Formato del Archivo Markdown del Skill

Los archivos Skill siguen una estructura de directorio fija:

```
.claude/skills/<skill-name>/SKILL.md
```

El formato del archivo es frontmatter + cuerpo Markdown:

```markdown
---
name: my-skill
description: Descripción en una frase de lo que hace este Skill
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  Usar cuando el usuario quiera... Por ejemplo: "cherry-pick to release", "hotfix".
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## Pasos

### 1. Primer paso
Operaciones concretas...

**Criterio de éxito**: punto de control que demuestra la completitud de este paso
```

### Descripción Detallada de los Campos del Frontmatter

Los siguientes son todos los campos analizados por la función `parseSkillFrontmatterFields` (`loadSkillsDir.ts:184`):

| Campo | Tipo | Descripción |
|------|------|------|
| `name` | string | Nombre de visualización (puede diferir del nombre del directorio) |
| `description` | string | Descripción en una frase, mostrada en `/help` |
| `allowed-tools` | string[] | Lista blanca de herramientas, soporta patrón de prefijo `Bash(git:*)` |
| `argument-hint` | string | Sugerencia de parámetros al activar por el usuario, ej. `"<branch-name>"` |
| `arguments` | string[] | Lista de nombres de parámetros, para sustitución de variable `$arg_name` |
| `when_to_use` | string | Le dice a la IA cuándo llamar proactivamente a este Skill, con frases de activación |
| `version` | string | Número de versión del Skill |
| `model` | string | Cobertura de modelo, ej. `opus`, `sonnet`; `inherit` significa heredar |
| `disable-model-invocation` | boolean | Prohibir llamada proactiva de la IA, solo activación manual del usuario |
| `user-invocable` | boolean | Si es visible en `/help` (predeterminado `true`) |
| `context` | `"fork"` | Si se establece, se ejecuta en sub-agente independiente |
| `agent` | string | Especificar tipo de agente |
| `effort` | EffortValue | Afecta la profundidad de reflexión del modelo |
| `paths` | string[] | Patrones de ruta en sintaxis gitignore, para activación condicional |
| `hooks` | HooksSettings | Configuración de hooks durante la ejecución del Skill |
| `shell` | FrontmatterShell | Configuración de ejecución de comandos shell en línea |

### Tipo SkillDefinition

`bundledSkills.ts` define `BundledSkillDefinition` (líneas 12-41), y los Skills del sistema de archivos corresponden al tipo `Command` (`src/types/command.js`). Ambos convergen en el objeto `Command` unificado en `createSkillCommand` (`loadSkillsDir.ts:269`):

```typescript
// loadSkillsDir.ts:316-400
return {
  type: 'prompt',
  name: skillName,
  description,
  allowedTools,
  argumentHint,
  argNames: argumentNames.length > 0 ? argumentNames : undefined,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context: executionContext,
  agent,
  effort,
  paths,
  contentLength: markdownContent.length,
  isHidden: !userInvocable,
  progressMessage: 'running',
  loadedFrom,
  hooks,
  skillRoot: baseDir,
  async getPromptForCommand(args, toolUseContext) { ... }
} satisfies Command
```

---

## 6.4 Mecanismo de Carga de Skills

### Flujo de Carga Completo de loadSkillsDir

`getSkillDirCommands` (`loadSkillsDir.ts:638`) es el punto de entrada de todo el flujo de carga, usando `lodash-es/memoize` para cachear resultados y evitar I/O redundante:

```
Al inicio
  ├── policySettings: ~/.claude-managed/.claude/skills/ (gestión empresarial)
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/ (desde cwd hacia arriba hasta home)
  ├── additionalDirs: directorios adicionales especificados por --add-dir
  └── legacyCommands: .claude/commands/ (compatibilidad hacia atrás)

Durante la sesión (descubrimiento dinámico)
  └── cuando el usuario lee/escribe archivos → discoverSkillDirsForPaths() → addSkillDirectories()
```

Los resultados de carga se deduplicar mediante `realpath` (`loadSkillsDir.ts:728-763`), evitando cargas duplicadas causadas por symlinks.

### Prioridad de Carga desde Múltiples Fuentes

El comentario del código indica explícitamente la prioridad de carga (`loadSkillsDir.ts:677-714`):

```
managed (política empresarial) < user (nivel usuario) < project (nivel proyecto) < additional (--add-dir)
```

Este es el principio de "más específico tiene mayor prioridad": el nivel de proyecto cubre el nivel de usuario, porque el proyecto tiene necesidades específicas.

**Casos especiales:**
- Modo `--bare`: omite el descubrimiento automático, solo carga directorios especificados explícitamente por `--add-dir`
- `skillsLocked` (política solo para plugins): prohíbe cargar Skills de nivel usuario/proyecto, solo permite la fuente Plugin
- Variable de entorno `CLAUDE_CODE_DISABLE_POLICY_SKILLS`: omite Skills de nivel managed

### Lógica de Descubrimiento y Coincidencia de Skills

**Descubrimiento estático** (al inicio): `getSkillDirCommands` escanea los directorios `~/.claude/skills/` en cada nivel, solo soporta formato de directorio (`skill-name/SKILL.md`), no soporta archivos `.md` individuales.

**Descubrimiento dinámico** (durante la sesión): cuando el usuario lee/escribe archivos, `discoverSkillDirsForPaths` (`loadSkillsDir.ts:861`) recorre la ruta del archivo hacia arriba, verificando en cada directorio si existe `.claude/skills/`, cargando cuando se encuentra mediante `addSkillDirectories`. Los directorios marcados por `.gitignore` se omiten (para evitar la contaminación de Skills en `node_modules`).

**Activación condicional** (frontmatter paths): los Skills con campo `paths` inicialmente no son visibles para el modelo, almacenados en el Map `conditionalSkills`. Cuando el usuario opera archivos que coinciden con las rutas, `activateConditionalSkillsForPaths` (`loadSkillsDir.ts:997`) usa la biblioteca `ignore` (sintaxis gitignore) para hacer coincidir; cuando hay coincidencia, se mueve a `dynamicSkills` y se activa.

---

## 6.5 Flujo de Ejecución de SkillTool

### Ruta Completa desde /skill-name hasta la Ejecución

`SkillTool` (`tools/SkillTool/SkillTool.ts:330`) es una implementación estándar de `Tool`; la IA ejecuta Skills llamando a esta herramienta. La ruta de ejecución completa:

```
El usuario escribe /skill-name o la IA decide llamar a SkillTool
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── eliminar barra diagonal inicial (manejo de compatibilidad)
  │     ├── verificar prefijo _canonical_ (Skill remoto, experimental)
  │     ├── findCommand() buscar el Command registrado
  │     ├── verificar flag disableModelInvocation
  │     └── confirmar type === 'prompt'
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── verificar reglas deny
  │     ├── verificar Skill canonical remoto (auto-permitir)
  │     ├── verificar reglas allow
  │     ├── skillHasOnlySafeProperties() → auto-permitir Skill seguro
  │     └── predeterminado: mostrar diálogo preguntando al usuario (behavior: 'ask')
  │
  └── call (SkillTool.ts:580)
        ├── verificar context === 'fork' → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent() (sub-agente independiente)
        └── de lo contrario (inline) → processPromptSlashCommand()
              └── inyectar newMessages + contextModifier en la sesión actual
```

### Inyección de Contexto del Skill

En la ejecución inline, `call` devuelve `newMessages` y `contextModifier` (`SkillTool.ts:767-840`):

- **newMessages**: lista de mensajes después de expandir el Skill, inyectados en el contexto de conversación actual
- **contextModifier**: función que modifica `ToolUseContext`, usada para:
  - Superponer `allowedTools` (permisos de herramientas declarados por el Skill)
  - Anular `mainLoopModel` (si el Skill especifica modelo)
  - Anular `effortValue` (si el Skill especifica effort)

Vale la pena notar que `contextModifier` adopta patrón de cadena de llamadas (`SkillTool.ts:777`), manejando correctamente la superposición de múltiples contextModifiers, en lugar de simplemente sobrescribir.

### Sustitución de Variables del Skill

`getPromptForCommand` en `createSkillCommand` (`loadSkillsDir.ts:343-398`) realiza las siguientes sustituciones antes de devolver el contenido del Skill:

1. **Sustitución de argumentos**: `$arg_name` → `substituteArguments()` inyecta los argumentos pasados por el usuario
2. **Variable de directorio**: `${CLAUDE_SKILL_DIR}` → ruta absoluta del directorio donde se encuentra el archivo Skill
3. **ID de sesión**: `${CLAUDE_SESSION_ID}` → ID de la sesión actual
4. **Ejecución de comandos shell**: `!``command`` ` → resultado ejecutado en línea (solo para Skills no-MCP)

Los Skills MCP deshabilitan la ejecución de comandos shell (`loadSkillsDir.ts:372`), evitando que Skills remotos no confiables inyecten comandos shell arbitrarios.

### Interacción entre Skills y Herramientas

En el modo de ejecución Forked (`executeForkedSkill`, `SkillTool.ts:121`), el Skill se ejecuta en un sub-agente completamente aislado:

- Se inicia un agente independiente mediante `runAgent()`, con presupuesto de tokens independiente
- Los mensajes de uso de herramientas durante la ejecución se reportan mediante el callback `onProgress`, la UI puede mostrar el progreso
- El resultado de ejecución se extrae como texto final mediante `extractResultText`, devuelto al agente padre
- La memoria se libera mediante `clearInvokedSkillsForAgent` (`SkillTool.ts:286`)

---

## 6.6 Catálogo Completo y Análisis de Bundled Skills

Los Skills integrados se registran mediante `registerBundledSkill()` (`bundledSkills.ts:55`) e inicializan al iniciar la CLI. Los siguientes son los 17 Skills integrados:

### 1. `update-config` (`updateConfig.ts`, 475 líneas)

**Función**: Configurar `settings.json` de Claude Code, incluyendo todas las configuraciones de Permissions, Hooks, Model, MCP, etc.

**Características**: El cuerpo del Skill se genera dinámicamente — usa `toJSONSchema(SettingsSchema())` para generar automáticamente la documentación JSON Schema desde el schema Zod, garantizando que la documentación siempre esté sincronizada con los tipos reales. Incluye documentación completa de Hooks (todos los eventos Hook, tipos de Hook, formato de salida JSON).

**Escenario de activación**: cuando el usuario quiere configurar automatizaciones de comportamiento, reglas de permisos, variables de entorno, configuraciones de modelo.

### 2. `schedule` (`scheduleRemoteAgents.ts`, 447 líneas)

**Función**: Gestionar agentes remotos programados (disparadores cron), crear, actualizar, listar y ejecutar tareas programadas.

**Características**: Verifica múltiples condiciones previas antes de llamar (tokens OAuth, información del repositorio, conectores MCP, entornos cloud), inyectando esta información dinámica en el prompt del Skill. Interactúa con el usuario a través de la herramienta `AskUserQuestion`.

**Escenario de activación**: cuando el usuario quiere crear agentes de Claude Code que se ejecuten de forma programada (ej. revisión diaria de código, informes automáticos).

### 3. `keybindings-help` (`keybindings.ts`, 339 líneas)

**Función**: Ayudar al usuario a personalizar atajos de teclado, modificar `~/.claude/keybindings.json`.

**Características**: Genera dinámicamente documentación desde constantes del código mediante `generateContextsTable()`, `generateActionsTable()`, y lista los atajos no reasignables mediante `generateReservedShortcuts()`, evitando errores del usuario.

**Escenario de activación**: el usuario quiere reasignar atajos, añadir atajos combinados, cambiar la tecla de envío.

### 4. `lorem-ipsum` (`loremIpsum.ts`, 282 líneas)

**Función**: Generar texto de marcador de posición de una cantidad fija de palabras de un token, para conteo de tokens y pruebas de rendimiento.

**Características**: Usa una lista de palabras de un token validadas por el API, garantizando que el parámetro `lorem` pueda controlar con precisión el número de tokens. Comúnmente usado para benchmarking y análisis de facturación de tokens.

**Escenario de activación**: texto de prueba que necesita un número preciso de tokens.

### 5. `skillify` (`skillify.ts`, 197 líneas)

**Función**: Convierte automáticamente el proceso de operaciones de la sesión actual en un archivo SKILL.md reutilizable.

**Características**: Este es el mecanismo de "auto-reproducción" del sistema de Skills. Leyendo la memoria de sesión y el historial de mensajes del usuario, guía al usuario con 4 rondas de diálogo `AskUserQuestion` para confirmar el nombre del flujo de trabajo, pasos, parámetros y condiciones de activación, generando finalmente un SKILL.md en formato estándar y escribiéndolo en disco.

**Limitación**: Solo disponible para `USER_TYPE === 'ant'` (empleados internos de Anthropic).

**Escenario de activación**: al final de la sesión, el usuario quiere solidificar el flujo de operaciones recién completado como Skill reutilizable.

### 6. `claude-api` (`claudeApi.ts`, 196 líneas + `claudeApiContent.ts`, 220 líneas)

**Función**: Ayudar a los desarrolladores a construir aplicaciones usando el Claude API o el Anthropic SDK.

**Características**:
- Detecta automáticamente el lenguaje del proyecto actual (escaneando sufijos de archivos, soporta Python/TypeScript/Java/Go/Ruby/C#/PHP/curl)
- Carga diferida (el contenido `.md` de 247KB solo se carga al ser llamado), evitando afectar el tiempo de inicio
- Incluye documentación API específica del lenguaje, patrones del Agent SDK, streaming, etc.
- Escribe documentación en un directorio temporal mediante el mecanismo `files`, el modelo puede leerla bajo demanda con herramientas Read/Grep

**Escenario de activación**: código que importa `anthropic` o el usuario pregunta cómo usar el Claude API.

### 7. `batch` (`batch.ts`, 124 líneas)

**Función**: Descompone grandes cambios de código (como migraciones, refactorizaciones, renombramiento masivo) en 5-30 agentes de worktree en paralelo para ejecutar.

**Características**: Modelo de ejecución de tres fases — Plan (entrar en Plan Mode para investigación profunda y descomposición) → Spawn Workers (iniciar en paralelo agentes en background con `isolation: "worktree"`) → Track Progress (renderizar tabla de estado en tiempo real). Cada worker trabaja en un git worktree independiente sin interferirse mutuamente, abriendo PRs al completar.

**Escenario de activación**: migración masiva de código, refactorización completa de la base de código, modificaciones en lote.

### 8. `loop` (`loop.ts`, 92 líneas)

**Función**: Ejecuta repetidamente un prompt o comando slash a intervalos fijos.

**Características**: Analiza inteligentemente los intervalos de tiempo (soporta formato de prefijo `5m`, `2h` y formato de sufijo `every 20m`), convirtiéndolos en expresiones cron y registrando tareas programadas llamando a `ScheduleCronTool`. Ejecuta inmediatamente una vez después de la configuración, sin esperar el primer disparador programado.

**Escenario de activación**: el usuario quiere verificar periódicamente el estado del despliegue, ejecutar periódicamente algún Skill.

### 9. `remember` (`remember.ts`, 82 líneas)

**Función**: Revisa entradas de auto-memory, propone promoverlas a `CLAUDE.md`, `CLAUDE.local.md` o memoria de equipo.

**Características**: Adopta el principio de "proponer primero, confirmar después"; no modifica archivos directamente, sino que muestra un informe clasificado (a promover/a limpiar/dudoso/sin acción necesaria), esperando la aprobación del usuario antes de ejecutar. Distingue convenciones de nivel de proyecto (CLAUDE.md), preferencias personales (CLAUDE.local.md) y conocimiento de nivel organizativo (memoria de equipo).

**Limitación**: Solo disponible para `USER_TYPE === 'ant'` con la función de auto-memory habilitada.

**Escenario de activación**: el usuario quiere organizar la memoria, evitando la acumulación infinita de auto-memory.

### 10. `simplify` (`simplify.ts`, 69 líneas)

**Función**: Realiza revisión de código en tres dimensiones del git diff actual (reutilización de código, calidad de código, eficiencia) y corrige directamente los problemas encontrados.

**Características**: Lanza simultáneamente tres sub-agentes en paralelo, responsables respectivamente de:
- **Agente de reutilización de código**: detecta reinvención de la rueda, señala funciones utilitarias existentes
- **Agente de calidad de código**: detecta estado redundante, inflación de parámetros, copiar-pegar, abstracciones que presentan fugas, etc.
- **Agente de eficiencia**: detecta cálculos innecesarios, falta de concurrencia, patrones N+1, fugas de memoria, etc.

Los tres agentes se fusionan después de completar y corrigen directamente, no solo reportan.

**Escenario de activación**: revisión de calidad después de completar un segmento de código; también llamado automáticamente por el flujo de trabajo del worker del Skill `batch`.

### 11. `debug` (`debug.ts`)

**Función**: Diagnostica los registros de depuración de la sesión actual de Claude Code para ayudar a solucionar problemas.

**Características**: Lee mediante tail (máximo 64KB) las últimas líneas del registro de depuración, evitando picos de memoria causados por archivos de registro demasiado grandes en sesiones largas. Para usuarios no empleados de Anthropic, primero habilita el registro de depuración antes de leer. Marcado con `disableModelInvocation: true`, evitando que la IA lo llame automáticamente (solo puede ser activado manualmente por el usuario).

### 12. `stuck` (`stuck.ts`)

**Función**: Diagnostica otros procesos de Claude Code congelados o bloqueados en la máquina, y envía el informe al canal de Slack.

**Características**: Herramienta de diagnóstico interna de Anthropic. Detecta anomalías como alta CPU (≥90% sostenida), estado D (I/O bloqueado), estado T (detenido por Ctrl+Z), estado Z (proceso zombie), alta memoria (≥4GB). Usa estructura de dos mensajes para enviar informes Slack (resumen de nivel superior + detalles en hilo).

### 13. `verify` (`verify.ts`)

**Función**: Verifica si los cambios de código coinciden con las expectativas ejecutando la aplicación.

**Características**: Lee el cuerpo del Skill desde `verifyContent.ts` (análisis SKILL.md), escribiendo archivos auxiliares en un directorio temporal mediante el mecanismo `files`. Solo disponible para `USER_TYPE === 'ant'`.

### 14. `claudeInChrome` (`claudeInChrome.ts`)

**Función**: Inicia una sesión headless conectada a un navegador Chrome real, con extensión Side Panel, Claude puede controlar el navegador en tiempo real.

### 15. `claudeCodeGuide` (integrado en el sistema `AgentTool`)

Usado para el flujo de orientación interna de Claude Code.

---

## 6.7 Relación entre Skills y Comandos

### El Límite entre Ambos

En el diseño de Claude Code, Skill y Command eran conceptos diferentes, pero ahora se han unificado:

- **Históricamente**: el directorio `/commands/` almacenaba comandos de prompt simples (archivos `.md`), y el directorio `/skills/` almacenaba flujos de trabajo más complejos con estructura de directorio (`skill-name/SKILL.md`)
- **Actualmente**: ambos son cargados por `loadSkillsDir.ts`, unificados en el tipo `Command`; `/commands/` se marca como `loadedFrom: 'commands_DEPRECATED'` (`loadSkillsDir.ts:608`)

La diferencia real actualmente está solo en la ruta de carga:
- `/skills/skill-name/SKILL.md`: formato nuevo, recomendado, soporta `baseDir` (el Skill puede llevar archivos auxiliares)
- `/commands/skill-name.md` o `/commands/skill-name/SKILL.md`: formato antiguo, compatibilidad hacia atrás

### Cuándo Usar Skill vs. Cuándo Usar Command

| Escenario | Método recomendado |
|------|---------|
| Flujo de trabajo multi-archivo (Skill con archivos de recursos auxiliares) | Formato de directorio `/skills/` |
| Reutilización de prompt simple (un solo archivo md es suficiente) | Aún se puede usar `/commands/` (compatible) |
| Necesita variable `${CLAUDE_SKILL_DIR}` | Debe usar formato de directorio `/skills/` |
| Necesita recursos incrustados `files:` (skill integrado) | `BundledSkillDefinition.files` |
| Integrado en el binario CLI | `registerBundledSkill()` |

---

## 6.8 Análisis de Decisiones de Diseño

### Por Qué Elegir Markdown en Lugar de JSON/YAML

Las instrucciones de ejecución del Skill (cuerpo) se escriben en lenguaje natural para que la IA pueda entenderlas y seguirlas. JSON/YAML solo puede codificar datos estructurados; no puede escribir directamente instrucciones complejas como "primero buscar archivos relevantes, luego analizar las dependencias, tener cuidado de no modificar los archivos de prueba".

Markdown combina ambos: el frontmatter (YAML) es responsable de los metadatos estructurados, el cuerpo (Markdown) es responsable de las instrucciones de ejecución legibles por humanos. Esta es una elección de formato pragmático.

### Control de Permisos del Skill

El control de permisos adopta un mecanismo de "lista blanca + preguntar" (`SkillTool.ts:871-900`):

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // Propiedades de CommandBase...
  'name', 'description', 'hasUserSpecifiedDescription', ...
  // NO incluido: 'allowedTools', 'hooks', 'paths', etc.
])
```

`skillHasOnlySafeProperties()` verifica si un Skill solo usa "propiedades seguras" — si el Skill no declara propiedades sensibles como `allowedTools`, `hooks`, `paths`, se permite automáticamente la ejecución sin confirmación del usuario. Este es un buen diseño de seguridad: las nuevas propiedades son inseguras por defecto y necesitan revisión explícita antes de poder agregarse a la lista blanca.

### Mecanismo de Escritura Segura de Archivos

Los Skills integrados incrustan archivos auxiliares mediante el campo `files`, usando medidas de seguridad estrictas al escribir en disco (`bundledSkills.ts:171-194`):

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

Usa `O_NOFOLLOW | O_EXCL` para prevenir ataques de symlink, los permisos del archivo son `0o600` (solo lectura/escritura del propietario). El directorio de escritura contiene un nonce aleatorio de cada inicio del proceso, previniendo ataques de predicción de rutas.

### Estrategia de Integración de MCP Skill

Los MCP Skills implementan una elegante inversión de dependencias mediante `mcpSkillBuilders.ts` (`mcpSkillBuilders.ts:1-43`):

La lógica de descubrimiento MCP (`mcpSkills.ts`) necesita usar `createSkillCommand` y `parseSkillFrontmatterFields`, pero importarlos directamente causaría dependencias circulares. La solución es:

1. `loadSkillsDir.ts` llama a `registerMCPSkillBuilders()` al inicializar el módulo para registrar estas dos funciones
2. `mcpSkills.ts` las obtiene cuando las necesita mediante `getMCPSkillBuilders()`

Este diseño también resuelve las limitaciones técnicas del empaquetado de Bun: en el bundle de Bun, las importaciones dinámicas con variables (no literales) no se pueden resolver, por lo que no se puede usar `await import(variable)`, solo este patrón de registro.

---

## 6.9 Patrones Transferibles

### Comparación del Sistema de Skills de Doramagic

| Dimensión | Claude Code Skill | Doramagic Skill |
|------|------------------|-----------------|
| Formato de archivo | `SKILL.md` (Markdown + YAML frontmatter) | `SKILL.md` (mismo formato) |
| Estructura de directorio | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| Motor de ejecución | SkillTool (llamada a herramienta IA) | Llamada a herramienta OpenClaw |
| Prioridad de fuente | policy < user < project < plugin | Reglas de plataforma OpenClaw |
| Skills integrados | 15+, compilados en el binario | En construcción |
| Sustitución de argumentos | `$arg_name`, frontmatter `arguments` | Mismo mecanismo |
| Contexto de ejecución | inline / fork (sub-agente) | inline (fase actual) |
| Activación condicional | frontmatter `paths` | No implementado aún |
| Descubrimiento dinámico | El trigger de operación de archivos activa el descubrimiento automático | No implementado aún |

### Patrones Centrales que se Pueden Aprender

**1. Patrón `skillify`: Auto-reproducción de flujos de trabajo**

El Skill `skillify` de Claude Code es un diseño extremadamente elegante — hace que la IA analice las operaciones que acaba de ejecutar y, a través del diálogo con el usuario, solidifique el proceso en un Skill reutilizable. Doramagic también puede implementar un `/dora-skillify`, solidificando un proceso exitoso de extracción de conocimiento como Skill específico del proyecto.

**2. Mecanismo de llamada proactiva de la IA mediante `when_to_use`**

El campo `when_to_use` del frontmatter hace que la IA sepa cuándo llamar proactivamente al Skill, sin necesidad de que el usuario ingrese explícitamente el comando slash. Los Skills de Doramagic también deberían dar importancia a este campo, permitiendo que la extracción de conocimiento se active automáticamente en el momento adecuado.

**3. Descubrimiento dinámico de habilidades y activación condicional**

El mecanismo de activar Skills según la ruta del archivo es muy adecuado para escenarios de conocimiento específico del proyecto de Doramagic: cuando el usuario opera archivos de un determinado dominio, activar automáticamente el Skill de extracción del dominio correspondiente (ej. activar el Skill de análisis de arquitectura front-end cuando se operan archivos TypeScript).

**4. El mecanismo `files` para gestionar recursos auxiliares**

Los Skills integrados incrustan documentación de referencia y código de ejemplo en el paquete del Skill mediante el campo `files`, el modelo los lee bajo demanda sin inyectarlos todos en el contexto de una vez. Los Skills grandes de Doramagic (como Soul Extractor) pueden adoptar este patrón para gestionar plantillas de extracción y materiales de referencia.

**5. Modelo de seguridad: Lista blanca allowedTools + auto-permitir Skills seguros**

Los Skills solo pueden usar las herramientas declaradas en el frontmatter. Claude Code distingue además entre "Skills seguros" (sin permisos especiales) y "Skills que requieren confirmación" (con allowedTools/hooks), permitiendo automáticamente los primeros para reducir la fricción. Este modelo de permisos merece ser aprendido por la plataforma OpenClaw.

---

## 6.10 Índice del Código Fuente

| Archivo | Líneas | Función |
|------|------|------|
| `skills/loadSkillsDir.ts` | 1,087 | Núcleo de carga de Skills: descubrimiento, análisis, deduplicación, activación condicional, descubrimiento dinámico |
| `skills/bundledSkills.ts` | 220 | Registro de Skills integrados, extracción de archivos, escritura segura |
| `tools/SkillTool/SkillTool.ts` | 1,108 | Herramienta de ejecución de Skills: validación, permisos, ejecución inline/fork |
| `skills/mcpSkillBuilders.ts` | 44 | Registro de constructores de MCP Skill (romper dependencias circulares) |
