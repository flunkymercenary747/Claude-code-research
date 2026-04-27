# Capítulo 11: Integración MCP

## 11.1 Descripción General y Posicionamiento

### Qué es MCP

MCP (Model Context Protocol) es un protocolo abierto diseñado bajo el liderazgo de Anthropic, que define el formato de comunicación estandarizado entre aplicaciones de IA y servicios de herramientas externas. En esencia es un protocolo JSON-RPC 2.0, que funciona sobre múltiples capas de transporte (stdio, SSE, HTTP Streamable, WebSocket), especificando formatos de mensajes estándar para descubrimiento de herramientas (`tools/list`), llamadas a herramientas (`tools/call`), gestión de recursos (`resources/list`/`resources/read`), plantillas de Prompt (`prompts/list`/`prompts/get`), entre otros.

### El Rol de MCP en Claude Code

El conjunto de herramientas integradas de Claude Code (Bash, Read, Edit, etc.) cubre el sistema de archivos y escenarios de desarrollo local. El posicionamiento de diseño de MCP es una **interfaz abierta de extensión de herramientas**: cualquier servicio de terceros (Slack, GitHub, Jira, bases de datos, automatización de navegador, etc.) puede implementar un servidor MCP, y Claude Code, al conectarse mediante el protocolo estándar, puede invocar estas capacidades externas sin modificar el código central.

Arquitectónicamente, Claude Code es un **cliente MCP** puro, no implementa ninguna capacidad de servidor MCP (excepto responder a solicitudes `roots/list` para notificar al servidor del directorio de trabajo). Las herramientas de cada servidor MCP conectado se registran dinámicamente como objetos Tool con el formato `mcp__<serverName>__<toolName>`, compartiendo el mismo framework de ejecución que las herramientas integradas.

### Volumen de Código

La integración MCP involucra aproximadamente 12.310 líneas de código TypeScript, distribuidas en los siguientes archivos:

| Archivo | Líneas | Responsabilidad |
|---|---|---|
| `services/mcp/client.ts` | 3.348 | Gestión de conexiones, descubrimiento de herramientas, núcleo de ejecución |
| `services/mcp/config.ts` | 1.578 | Gestión de configuración (fusión de múltiples fuentes, filtrado por política) |
| `services/mcp/auth.ts` | 2.465 | Autenticación OAuth 2.0 (incluyendo acceso entre aplicaciones XAA) |
| `services/mcp/utils.ts` | 575 | Filtrado de herramientas, hashing de nombres, detección de obsolescencia |
| `services/mcp/types.ts` | 258 | Definiciones de tipos (Transport, ServerConfig, estado de conexión) |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | Clasificación para plegado en UI (identificación de herramientas Search/Read) |
| `tools/MCPTool/UI.tsx` | 402 | Renderizado de resultados de ejecución de herramientas |
| `services/mcp/channelPermissions.ts` | 240 | Intermediación de permisos de Channel |
| `services/mcp/channelNotification.ts` | 316 | Mecanismo de inyección de mensajes en Channel |
| `services/mcp/elicitationHandler.ts` | 313 | Manejo de Elicitation (interacción mediante formularios/URL) |
| `skills/mcpSkillBuilders.ts` | 44 | Registro de constructores de Skills (desacoplamiento del grafo de dependencias) |

---

## 11.2 Fundamentos Teóricos

### El Patrón de Extensión de Herramientas Impulsado por Protocolo

Los sistemas de plugins tradicionales generalmente dependen del SDK proporcionado por la aplicación anfitriona; los desarrolladores de plugins deben conocer las interfaces internas del anfitrión. MCP adopta el modelo **impulsado por protocolo** (protocol-driven): todas las interacciones entre el anfitrión (Claude Code) y el plugin (servidor MCP) se realizan mediante mensajes JSON-RPC estándar, y ambas partes pueden evolucionar independientemente.

Esto es muy coherente con la filosofía de diseño de LSP (Language Server Protocol):

| Dimensión | LSP | MCP |
|---|---|---|
| Patrón central | Editor ↔ Servidor de lenguaje | AI Agent ↔ Servidor de herramientas |
| Mecanismo de descubrimiento | Intercambio de capabilities en `initialize` | `tools/list`, `resources/list`, `prompts/list` |
| Capa de transporte | stdio, LSP over TCP | stdio, SSE, HTTP Streamable, WebSocket |
| Comunicación bidireccional | Soportada | Soportada (notifications, elicitation) |
| Negociación de versión | Soportada | Soportada (`protocolVersion`) |

LSP resolvió el problema de explosión M×N de "cada editor necesita integrarse con cada lenguaje"; MCP resuelve el mismo tipo de problema de "cada herramienta de IA necesita integrarse con cada servicio externo".

### Principios de Diseño del Protocolo Cliente-Servidor

Dos elecciones de diseño clave de MCP tienen profunda influencia en la implementación de Claude Code:

**Negociación de capacidades (Capability Negotiation)**: al conectarse, el servidor declara mediante `ServerCapabilities` el subconjunto de funcionalidades que soporta (`tools`, `prompts`, `resources`, `elicitation`, `experimental`); el cliente solo invoca las funcionalidades que el servidor ha declarado. Esto significa que Claude Code no necesita escribir ramas especiales para cada tipo de servidor; uniformemente decide el comportamiento mediante comprobación de `capabilities`.

**Anotaciones de herramientas (Tool Annotations)**: la versión MCP 2025-03 introdujo el campo `tool.annotations`, donde el servidor puede declarar marcadores semánticos como `readOnlyHint`, `destructiveHint`, `openWorldHint`. Claude Code mapea directamente estos marcadores a los métodos `isReadOnly()`, `isDestructive()`, `isOpenWorld()` de la herramienta, pudiendo tomar decisiones de seguridad sin mantener una lista blanca estática de nombres de herramientas.

---

## 11.3 Arquitectura del Cliente MCP

### Interfaz Principal de la Clase MCPClient

Claude Code no implementa directamente un cliente MCP, sino que encapsula la clase `Client` proporcionada por el SDK `@modelcontextprotocol/sdk`. `connectToServer` es la función de entrada principal (`client.ts`), usando `lodash/memoize` para caché a nivel de conexión, con clave de caché `${name}-${jsonStringify(serverRef)}`:

```typescript
// client.ts (aproximadamente línea 540)
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... inicializa transport según serverRef.type
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... conexión, timeout, negociación de capacidades
  },
  getServerCacheKey,
)
```

### Gestión de Conexiones (Establecimiento, Mantenimiento, Desconexión)

**Establecimiento de conexión**: `connectToServer` crea el transport correspondiente según `serverRef.type`, luego inicia `client.connect(transport)` y establece un timeout de 30 segundos (`getConnectionTimeoutMs()`, que puede sobreescribirse con la variable de entorno `MCP_TIMEOUT`):

```typescript
// client.ts (aproximadamente línea 1000)
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    transport.close().catch(() => {})
    reject(new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
      `MCP server "${name}" connection timed out after ${getConnectionTimeoutMs()}ms`,
      'MCP connection timeout',
    ))
  }, getConnectionTimeoutMs())
  connectPromise.then(() => clearTimeout(timeoutId), () => clearTimeout(timeoutId))
})
await Promise.race([connectPromise, timeoutPromise])
```

**Mantenimiento de conexión**: se implementa la detección de errores y reconexión automática sobreescribiendo `client.onerror` y `client.onclose`. Para transportes remotos (SSE/HTTP), se mantiene un contador `consecutiveConnectionErrors`; después de 3 errores terminales consecutivos (`ECONNRESET`/`ETIMEDOUT`/`EPIPE`, etc.) se activa `closeTransportAndRejectPending`, que llama a `client.close()` para rechazar todos los `callTool()` pendientes, limpia la caché de memoize y reconecta automáticamente en la siguiente solicitud:

```typescript
// client.ts (aproximadamente línea 1250)
client.onclose = () => {
  // limpia todas las cachés relacionadas; la próxima llamada activa reconexión
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Manejo de expiración de sesión**: los servidores MCP con transporte HTTP pueden devolver HTTP 404 + código de error JSON-RPC `-32001` (Session Not Found). Claude Code detecta este patrón de error específico, activa la reconexión y reintenta de forma transparente en `fetchToolsForClient.call()` (máximo 1 vez):

```typescript
// client.ts (aproximadamente línea 150)
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**Desconexión**: el transporte stdio usa escalada de señales en tres fases: primero `SIGINT` (espera 100ms), luego `SIGTERM` (espera 400ms), finalmente `SIGKILL`; tiempo límite total de desconexión 600ms, para evitar bloquear la salida del CLI.

### Descubrimiento y Registro Dinámico de Herramientas

`fetchToolsForClient` (con caché LRU, capacidad 20) envía `tools/list` al servidor y envuelve cada herramienta como un objeto `Tool` que cumple la interfaz interna:

- **Regla de nomenclatura**: `mcp__${normalizeNameForMCP(serverName)}__${toolName}` (formato con guiones bajos)
- **Truncamiento de descripción**: las descripciones que superan `MAX_MCP_DESCRIPTION_LENGTH = 2048` caracteres se truncan y se añade `… [truncated]`, previniendo que la documentación excesivamente larga de servidores generadores de OpenAPI contamine el contexto
- **Mapeo de permisos**: `tool.annotations.readOnlyHint` → `isReadOnly()`, `tool.annotations.destructiveHint` → `isDestructive()`
- **Clasificación para plegado**: se llama a `classifyMcpToolForCollapse(serverName, toolName)` para determinar si es una herramienta de tipo Search/Read

Igualmente, `fetchCommandsForClient` envía `prompts/list` y convierte los MCP Prompts en objetos de `/comando`; `fetchResourcesForClient` envía `resources/list` e inyecta `ListMcpResourcesTool` y `ReadMcpResourceTool` para servidores que soportan recursos.

### Capa de Transporte de Mensajes

Claude Code soporta 6 tipos de transporte:

| Tipo | Caso de uso | Clase Transport |
|---|---|---|
| `stdio` | Subproceso local (mayoría de servidores de la comunidad) | `StdioClientTransport` |
| `sse` | Servidor SSE remoto (con OAuth) | `SSEClientTransport` |
| `sse-ide` | SSE interno de extensión IDE (sin OAuth) | `SSEClientTransport` (configuración simplificada) |
| `http` | MCP Streamable HTTP (especificación más reciente) | `StreamableHTTPClientTransport` |
| `ws` | Transporte WebSocket | `WebSocketTransport` personalizado |
| `ws-ide` | WebSocket interno de extensión IDE | `WebSocketTransport` (con `X-Claude-Code-Ide-Authorization`) |

En escenarios especiales, los servidores MCP de Chrome Extension y Computer Use se ejecutan en **modo en proceso (In-Process)** a través de `createLinkedTransportPair()` estableciendo un canal en memoria, evitando el overhead de ~325 MB de un subproceso.

El transporte HTTP tiene un detalle de ingeniería importante: cada solicitud POST debe llevar la cabecera `Accept: application/json, text/event-stream` (requerida por la especificación MCP Streamable HTTP). Claude Code la inyecta uniformemente mediante `wrapFetchWithTimeout` para prevenir que ciertos entornos de runtime la omitan:

```typescript
// client.ts (aproximadamente línea 460)
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// en wrapFetchWithTimeout:
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 Gestión de Configuración MCP

### Formato de Configuración de Servidores

`types.ts` usa Zod para definir 7 esquemas de configuración de servidores, agregados en `McpServerConfigSchema` mediante `z.union([...])`:

```typescript
// types.ts (líneas 28-115, resumen)
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // compatibilidad retroactiva: sin campo type = stdio
  command: z.string().min(1),
  args: z.array(z.string()).default([]),
  env: z.record(z.string(), z.string()).optional(),
}))

export const McpSSEServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('sse'),
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: McpOAuthConfigSchema().optional(),
}))
// + HTTP / WebSocket / SDK / claudeai-proxy ...
```

`ScopedMcpServerConfig` añade a la configuración base los campos `scope` (origen de configuración) y `pluginSource` (identificador de la fuente que proporciona el plugin), utilizados para la verificación de permisos del Channel.

### Fusión de Configuración de Múltiples Fuentes (enterprise > local > project > user > dynamic)

`getClaudeCodeMcpConfigs` (`config.ts`) implementa la fusión de configuración en múltiples capas, con prioridad de mayor a menor:

1. **enterprise** (`managed-mcp.json`): modo exclusivo para empresa; cuando este archivo existe, bloquea todas las demás fuentes
2. **local** (privado del proyecto, almacenado en la configuración global del usuario, vinculado al CWD)
3. **project** (`.mcp.json`, recorre el árbol de directorios hacia arriba, prioridad al más cercano)
4. **user** (campo `mcpServers` en el `~/.claude/config.json` global)
5. **dynamic** (inyección en tiempo de ejecución mediante el parámetro CLI `--mcp-config`)

La configuración de proyecto requiere una **puerta de aprobación del usuario**: la primera vez que se encuentra un servidor en `.mcp.json`, aparece un diálogo de aprobación. `getProjectMcpServerStatus()` lee las configuraciones `enabledMcpjsonServers`/`disabledMcpjsonServers` y devuelve `approved`/`rejected`/`pending`. En modo no interactivo (parámetro `-p`, llamada SDK) con `isSettingSourceEnabled('projectSettings')`, se aprueba automáticamente.

Tras fusionar la configuración, también se realiza **deduplicación**: los servidores Plugin se deduplicarán por "firma" (servidores stdio usan array de comandos, servidores remotos usan URL), para prevenir que el mismo servidor subyacente se conecte dos veces; los Connectors de claude.ai también evitan duplicados con configuraciones manuales mediante el mismo mecanismo.

### Expansión de Variables de Entorno

Los archivos de configuración pueden usar la sintaxis `${ENV_VAR}`, y `expandEnvVarsInString` (`config.ts`/`envExpansion.ts`) expande las variables al leer la configuración. Las variables no definidas se recopilan en una lista `missingVars` y se reportan al usuario.

---

## 11.5 Sistema de Autenticación MCP

### Integración OAuth 2.0

`ClaudeAuthProvider` (`auth.ts`) implementa la interfaz `OAuthClientProvider` del SDK MCP, gestionando todo el ciclo de vida OAuth. El flujo de autenticación sigue el flujo de código de autorización RFC 6749 + PKCE (Proof Key for Code Exchange), recibiendo la respuesta a través de un servidor HTTP local:

1. **Descubrimiento de metadatos**: primero sondea RFC 9728 (`/.well-known/oauth-protected-resource`), si falla retrocede a RFC 8414 (`/.well-known/oauth-authorization-server`), finalmente intenta el descubrimiento sensible a la ruta (manteniendo compatibilidad retroactiva)
2. **DCR (Registro Dinámico de Cliente)**: en la primera autenticación se registra automáticamente el cliente OAuth; `clientId`/`clientSecret` se almacenan en el Keychain del sistema
3. **Intercambio de token**: un puerto local aleatorio recibe el código de autorización y lo intercambia por access_token + refresh_token
4. **Renovación de token**: `checkAndRefreshOAuthTokenIfNeeded()` detecta la expiración antes de la llamada y renueva; en caso de fallo, realiza reintentos inteligentes

**Capa de compatibilidad con Slack**: algunos servidores OAuth (notablemente Slack) devuelven HTTP 200 con un cuerpo de error en el endpoint de tokens, violando las expectativas de RFC 6749. Claude Code reescribe estas respuestas a HTTP 400 mediante `normalizeOAuthErrorBody`, para que la lógica de clasificación de errores del SDK funcione correctamente:

```typescript
// auth.ts (aproximadamente línea 250)
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // detecta si es un OAuthErrorResponse disfrazado de 200
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // normaliza el código de error no estándar de Slack 'invalid_refresh_token' a 'invalid_grant'
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### Soporte para Múltiples Métodos de Autenticación

Además del OAuth estándar, Claude Code también soporta:

- **Step-Up Auth**: algunas operaciones requieren elevar el alcance de permisos; el servidor devuelve HTTP 403 con un nuevo requisito de scope adjunto; Claude Code lo detecta y reactiva el flujo OAuth
- **XAA (Cross-App Access / SEP-990)**: en escenarios empresariales, un único IdP (soporta OIDC) autoriza múltiples servidores MCP con un solo inicio de sesión, usando el flujo combinado de RFC 8693 (Token Exchange) + RFC 7523 (JWT Bearer), sin necesidad de abrir ventanas de navegador para cada servidor individual
- **Static Headers**: inyección de cabeceras de autenticación estáticas a través de archivos de configuración o scripts `headersHelper` (adecuado para autenticación con API Key)

### Gestión de Tokens

Los datos de token se almacenan en el almacenamiento seguro del sistema (macOS Keychain / Linux Secret Service), con clave `${serverName}|${SHA256(config)[:16]}`, asegurando que servidores con el mismo nombre pero configuraciones diferentes usen slots de token independientes.

`auth-cache` (`mcp-needs-auth-cache.json`) registra los servidores que recientemente devolvieron 401, con TTL de 15 minutos, evitando sondear repetidamente servidores que inevitablemente fallarán en cada inicio. La lectura de caché se comparte mediante Promise (`authCachePromise`), previniendo N lecturas concurrentes del mismo archivo durante la conexión masiva.

---

## 11.6 Ejecución de Herramientas MCP

### Flujo de Ejecución de MCPTool

Cuando el LLM decide llamar a `mcp__slack__send_message`, el flujo de ejecución es:

1. **Enrutamiento**: se llama a la función `call()` registrada por `fetchToolsForClient`, con el input JSON generado por el LLM como parámetro
2. **Comprobación de reconexión**: `ensureConnectedClient(client)` verifica si la conexión sigue siendo válida, reconectando si es necesario
3. **Notificación de progreso**: se emite el evento `mcp_progress: started` a través del callback `onProgress`
4. **Llamada a herramienta**: `callMCPToolWithUrlElicitationRetry` (que encapsula `callMCPTool`) envía la solicitud `tools/call` al servidor
5. **Procesamiento de resultado**: se maneja especialmente el contenido de imágenes y binarios grandes (se persisten en disco, se transmite la referencia); el contenido de texto excesivamente grande se trunca
6. **Notificación de progreso**: se emite el evento `mcp_progress: completed` (con duración)

Lógica de reintento transparente para expiración de sesión:

```typescript
// client.ts (aproximadamente línea 2100)
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // reintento automático una vez
    }
    throw error
  }
}
```

### classifyForCollapse — Clasificación de Resultados de Herramientas para Plegado de Contexto

`classifyForCollapse.ts` mantiene dos Sets estáticos: `SEARCH_TOOLS` (aproximadamente 100 nombres de herramientas) y `READ_TOOLS` (aproximadamente 300 nombres de herramientas), cubriendo más de 40 servidores MCP mainstream (Slack, GitHub, Linear, Datadog, Sentry, Jira, Asana, Gmail, Grafana, PagerDuty, etc.).

Regla de clasificación: el nombre de la herramienta primero pasa por `normalize()` (conversión uniforme de camelCase/kebab-case a snake_case), luego se comprueba si está en alguno de los dos Sets:

```typescript
// classifyForCollapse.ts (líneas 587-598)
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')
    .replace(/-/g, '_')
    .toLowerCase()
}

export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean } {
  const normalized = normalize(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
  }
}
```

**Intención de diseño**: los resultados de herramientas de tipo Search/Read generalmente son extensos, pero su valor para el razonamiento posterior del LLM es limitado (estado intermedio de recuperación). Una vez marcados, la capa de UI puede plegar estos resultados en el historial de conversación, ahorrando espacio visual y ventana de contexto. Nótese que la clasificación es **conservadora** (las herramientas desconocidas no se pliegan), y **solo se basa en el nombre de la herramienta**, sin distinguir el nombre del servidor, porque los nombres de herramientas de los principales servidores son identificadores estables entre instancias.

### Control de Permisos y Sandbox

Antes de ejecutar una herramienta MCP se llama a `checkPermissions()`, que devuelve el estado `passthrough` (siempre requiere mostrar el prompt de permisos), con información que incluye un acceso directo para añadir el nombre de esa herramienta a la lista de reglas `allow`.

El timeout de las llamadas a herramientas se controla mediante la variable de entorno `MCP_TOOL_TIMEOUT`, con valor por defecto de `100_000_000` milisegundos (aproximadamente 27,8 horas, casi "ilimitado"), permitiendo que los servidores MCP con operaciones lentas terminen normalmente.

---

## 11.7 Sistema de Channel MCP

El sistema Channel es un uso extendido de MCP: permite que plataformas de mensajería externas (Telegram, Discord, iMessage, Slack, etc.) envíen mensajes a sesiones de Claude Code en curso (feature flag: `KAIROS`/`KAIROS_CHANNELS`, gate en tiempo de ejecución: `tengu_harbor`).

### Gestión de Permisos de Channel

`channelPermissions.ts` implementa un mecanismo de **delegación de permisos**: cuando Claude Code se encuentra con una operación que requiere aprobación del usuario, puede enviar simultáneamente un prompt a través del servidor Channel al teléfono del usuario; cuando el usuario responde `yes <ID de 5 letras>`, el servidor lo parsea y notifica a Claude Code de la aprobación mediante el evento `notifications/claude/channel/permission`.

El ID de 5 letras usa un alfabeto de 25 caracteres (sin la letra `l` para evitar confusión con `1`/`I`), generado mediante hash FNV-1a, con filtrado de palabras ofensivas (lista `ID_AVOID_SUBSTRINGS`, aproximadamente 24 palabras), asegurando que no aparezca contenido inapropiado en mensajes de trabajo:

```typescript
// channelPermissions.ts (líneas 86-110)
export function shortRequestId(toolUseID: string): string {
  let candidate = hashToId(toolUseID)
  for (let salt = 0; salt < 10; salt++) {
    if (!ID_AVOID_SUBSTRINGS.some(bad => candidate.includes(bad))) {
      return candidate
    }
    candidate = hashToId(`${toolUseID}:${salt}`)
  }
  return candidate
}
```

El servidor Channel debe declarar tanto `capabilities.experimental['claude/channel']` como `capabilities.experimental['claude/channel/permission']` para convertirse en intermediario de permisos, previniendo la apertura accidental de fronteras de seguridad.

### Mecanismo de Notificación de Channel

`channelNotification.ts` define la lógica completa de control de entrada para mensajes entrantes (`gateChannelServer`), verificando en orden:

1. Declaración de capacidades del servidor (`claude/channel`)
2. Interruptor en tiempo de ejecución (`tengu_harbor`)
3. Autenticación OAuth (solo soporta inicio de sesión con cuenta claude.ai, no API Key)
4. Política del equipo/empresa (`channelsEnabled: true`)
5. Parámetro `--channels` de la sesión (Channels declarados de confianza explícitamente por el usuario)
6. Verificación del origen en Marketplace (previene que `slack@evil` suplante a `slack@anthropic`)

Los mensajes entrantes se envuelven en el formato `<channel source="serverName" meta_key="value">content</channel>` e inyectan en la cola de la sesión. Después de que `SleepTool` realiza el polling (aproximadamente cada 1 segundo), el modelo puede decidir cómo responder.

### Manejo de Elicitation

`elicitationHandler.ts` gestiona las solicitudes de interacción iniciadas activamente por el servidor (especificación MCP Elicitation). Soporta dos modos:

- **Modo formulario**: el servidor solicita al usuario rellenar un formulario (el campo `requestedSchema` define el JSON Schema)
- **Modo URL**: el servidor solicita al usuario visitar una URL para completar la operación (como autorización OAuth)

Flujo de procesamiento: primero se ejecuta el sistema de Hooks (respuesta programática); si los Hooks no responden, la solicitud se añade a la cola `AppState.elicitation.queue` esperando que la UI renderice el formulario o abra el navegador; cuando el usuario completa la operación, el callback `respond()` activa la respuesta:

```typescript
// elicitationHandler.ts (líneas 69-90)
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. primero intentar Hook (respuesta programática)
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. mostrar UI, esperar respuesta del usuario
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

El modo URL también soporta `ElicitationCompleteNotificationSchema`: cuando el servidor completa la operación, notifica activamente a Claude Code, el elemento de cola correspondiente se marca como `completed: true`, y la UI actualiza su estado de visualización en consecuencia.
