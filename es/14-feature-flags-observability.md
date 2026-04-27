# Capítulo 14: Feature Flags y Observabilidad

## 14.1 Descripción General y Posicionamiento

El sistema de observabilidad de Claude Code es un sistema multinivel y multiobjetivo, que cubre desde la criba de características en tiempo de compilación hasta el rastreo de comportamiento en tiempo de ejecución. Todo el sistema se sustenta en tres grandes pilares:

1. **Sistema de Feature Flags**: diseño de doble carril: llamadas `feature()` en tiempo de compilación (que implementan Dead Code Elimination mediante `bun:bundle`) junto con configuración dinámica de GrowthBook en tiempo de ejecución. Las primeras controlan los límites de funcionalidad distribuidos a distintos grupos de usuarios; las segundas permiten ajustar interruptores de funcionalidad sin necesidad de una nueva publicación.

2. **Pipeline de observabilidad**: basado en el estándar OpenTelemetry, soporta tres protocolos de exportación gRPC/HTTP/Protobuf, recopila de forma unificada las tres señales Metrics, Logs y Traces, y proporciona una capa de depuración interna mediante el formato de rastreo Perfetto.

3. **Recopilación de Analytics**: enrutamiento de doble vía: Datadog (monitorización externa) + registro de eventos 1P de primera parte (BigQuery/proto interno), identificando todos los eventos de negocio con el prefijo `tengu_*`, y usando la configuración de muestreo dinámico de GrowthBook para controlar el volumen de datos.

El principio de diseño central de este sistema es el **aislamiento por capas**: privacidad del usuario primero (sin registrar contenido de código ni rutas de archivos por defecto), diferencias entre construcciones internas y externas (ant-only vs external), y graceful degradation (cada capa tiene su kill-switch).

---

## 14.2 Fundamentos Teóricos

### Desarrollo Impulsado por Feature Flags

Los Feature Flags permiten al equipo desarrollar funcionalidades en distintas fases dentro del mismo código base y habilitarlas según sea necesario. Claude Code adopta un mecanismo de flag de dos capas:

- **Flags en tiempo de compilación**: mediante las llamadas `feature()` proporcionadas por `bun:bundle`, se realiza Dead Code Elimination durante el empaquetado. Los bloques de código que no existen en la versión externa se eliminan completamente, lo que no solo reduce el tamaño del paquete sino que también evita que la lógica interna sea ingeniería inversa.
- **Flags en tiempo de ejecución**: obtenidos dinámicamente desde el servidor a través del SDK de GrowthBook, soportando escenarios de pruebas A/B, lanzamientos graduales y kill-switches de emergencia.

### Los Tres Pilares de la Observabilidad

La comunidad OpenTelemetry define la observabilidad en tres señales (Three Pillars of Observability):

- **Metrics (métricas)**: datos numéricos de series temporales, como latencia de API y consumo de tokens. Claude Code usa `@opentelemetry/sdk-metrics` y exporta mediante PeriodicExportingMetricReader cada 60 segundos.
- **Logs (registros)**: registros de eventos estructurados. Todas las llamadas a `logEvent()` finalmente se exportan en lotes a través de `LoggerProvider` + `BatchLogRecordProcessor` de OTel.
- **Traces (trazas)**: cadenas de llamadas distribuidas. Claude Code establece un árbol de Spans jerárquico Interaction → LLM Request → Tool Call mediante `sessionTracing.ts`, soportando el rastreo de relaciones padre-hijo en escenarios multi-Agent.

### Aplicación de Pruebas A/B en Herramientas CLI

A diferencia de los productos web, las pruebas A/B en herramientas CLI se enfrentan a desafíos únicos: sin huella de navegador, múltiples plataformas y canales de distribución, y escenarios de ejecución sin conexión. Las estrategias de respuesta de Claude Code:

- Segmentación por dimensión de usuario: `GrowthBookUserAttributes` lleva atributos como `platform`, `subscriptionType`, `rateLimitTier`, soportando experimentos estratificados.
- Caché en disco local: tras cada obtención exitosa de valores de características desde el servidor, se escribe en `cachedGrowthBookFeatures` dentro de `~/.claude/config.json`, asegurando que el último valor conocido esté disponible sin conexión.
- Deduplicación de exposición: los eventos de exposición de experimentos para cada feature dentro de la misma sesión se registran solo una vez (`loggedExposures` Set).

---

## 14.3 Sistema de Feature Flags

### Integración de GrowthBook

GrowthBook es una plataforma de Feature Flags y pruebas A/B de código abierto. Claude Code se integra mediante el SDK oficial `@growthbook/growthbook`, en el archivo `src/services/analytics/growthbook.ts` (1.155 líneas).

**Flujo de inicialización**:

```typescript
// growthbook.ts:529-600 (simplificado)
export const initializeGrowthBook = memoize(
  async (): Promise<GrowthBook | null> => {
    let clientWrapper = getGrowthBookClient()
    // ...
    await clientWrapper.initialized
    setupPeriodicGrowthBookRefresh()
    return clientWrapper.client
  },
)
```

Diseño clave: `memoize` garantiza que el cliente GrowthBook se inicialice solo una vez durante el ciclo de vida del proceso; cuando la autenticación cambia (inicio/cierre de sesión), el cliente se destruye y recrea mediante `refreshGrowthBookAfterAuthChange()`, en lugar de intentar actualizar `apiHostRequestHeaders` (el SDK no soporta actualizaciones después de la inicialización).

**Modelo de atributos de usuario** (`growthbook.ts:31-46`):

```typescript
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

**Estrategia de actualización**:
- Usuarios externos: actualización cada 6 horas (`6 * 60 * 60 * 1000`)
- Empleados internos (ant): actualización cada 20 minutos

**Arquitectura de caché** (tres niveles de prioridad):
1. Map `remoteEvalFeatureValues` en memoria (valor más reciente en el proceso)
2. Caché en disco `cachedGrowthBookFeatures` en `~/.claude/config.json` (persistencia entre procesos)
3. `cachedStatsigGates` heredado (capa de compatibilidad para migración, en proceso de eliminación)

**Workaround de compatibilidad de API** (`growthbook.ts:320-390`): la respuesta remoteEval del servidor usa el campo `value`, mientras que el SDK espera `defaultValue`; el código incluye lógica explícita de conversión de formato, con un comentario TODO esperando la corrección del servidor.

**Sobreescritura mediante variable de entorno** (solo para usuarios internos ant):
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### Listado Completo de Feature Flags en Tiempo de Compilación (80+)

Implementados mediante llamadas `feature()` de `bun:bundle` para Dead Code Elimination; a continuación se presenta el listado completo extraído del código fuente:

| Nombre del Flag | Ubicación | Funcionalidad controlada |
|---|---|---|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Rastreo Perfetto (ant-only) |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | Beta de telemetría mejorada |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | Sistema de modo automático/tareas programadas |
| `KAIROS_BRIEF` | `commands.ts` | Modo resumido de KAIROS |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | Soporte de canales KAIROS |
| `KAIROS_DREAM` | `commands.ts` | Modo sueño de KAIROS |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | Suscripción a webhooks de GitHub |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | Notificaciones push de KAIROS |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Disparadores de Agent (tareas programadas) |
| `AGENT_TRIGGERS_REMOTE` | — | Disparadores remotos de Agent |
| `AGENT_MEMORY_SNAPSHOT` | — | Instantáneas de memoria de Agent |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | Clasificador de conversaciones |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | Agente de verificación |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | Agentes integrados de exploración/planificación |
| `COORDINATOR_MODE` | `builtInAgents.ts` | Modo coordinador |
| `FORK_SUBAGENT` | `commands.ts` | Fork de sub-agente |
| `BUDDY` | `commands.ts` | Funcionalidad Buddy |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Bandeja de entrada Unix Domain Socket |
| `BRIDGE_MODE` | `commands.ts` | Modo puente (CCR) |
| `DAEMON` | `commands.ts` | Modo daemon |
| `VOICE_MODE` | `commands.ts` | Modo de voz |
| `ULTRAPLAN` | `commands.ts` | Comando UltraPlan |
| `ULTRATHINK` | — | Funcionalidad UltraThink |
| `TORCH` | `commands.ts` | Comando TORCH (carga dinámica) |
| `MCP_SKILLS` | `commands.ts` | Soporte de Skills MCP |
| `CHICAGO_MCP` | `metadata.ts` | Servidor MCP integrado Chicago (computer-use) |
| `WORKFLOW_SCRIPTS` | `commands.ts` | Scripts de workflow |
| `CCR_REMOTE_SETUP` | `commands.ts` | Comando de configuración remota CCR |
| `CCR_AUTO_CONNECT` | — | Conexión automática CCR |
| `CCR_MIRROR` | — | Modo espejo CCR |
| `PROACTIVE` | `commands.ts` | Modo proactivo |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | Búsqueda experimental de Skills |
| `HISTORY_SNIP` | `commands.ts` | Función de fragmentos de historial |
| `HISTORY_PICKER` | — | Selector de historial |
| `WEB_BROWSER_TOOL` | — | Herramienta de navegador web |
| `QUICK_SEARCH` | — | Búsqueda rápida |
| `MONITOR_TOOL` | — | Herramienta de monitorización |
| `OVERFLOW_TEST_TOOL` | — | Herramienta de prueba de desbordamiento |
| `BREAK_CACHE_COMMAND` | — | Comando de punto de interrupción de caché forzado |
| `TREE_SITTER_BASH` | — | Análisis Bash con Tree-sitter |
| `TREE_SITTER_BASH_SHADOW` | — | Comparación sombra con Tree-sitter |
| `BASH_CLASSIFIER` | — | Clasificador de seguridad Bash |
| `TERMINAL_PANEL` | — | Panel de terminal |
| `NATIVE_CLIPBOARD_IMAGE` | — | Soporte de imágenes nativo del portapapeles |
| `NATIVE_CLIENT_ATTESTATION` | — | Atestación nativa del cliente |
| `AUTO_THEME` | — | Tema automático |
| `POWERSHELL_AUTO_MODE` | — | Modo automático PowerShell |
| `TOKEN_BUDGET` | — | Visualización del presupuesto de tokens |
| `STREAMLINED_OUTPUT` | — | Modo de salida simplificado |
| `CONNECTOR_TEXT` | — | Texto de conector |
| `CONTEXT_COLLAPSE` | — | Colapso de contexto |
| `COMPACTION_REMINDERS` | — | Recordatorios de compactación |
| `CACHED_MICROCOMPACT` | — | Microcompresión en caché |
| `REACTIVE_COMPACT` | — | Compresión reactiva |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Detección de puntos de interrupción de Prompt Cache |
| `EXTRACT_MEMORIES` | — | Extracción automática de memorias |
| `LODESTONE` | — | Funcionalidad Lodestone |
| `TEAMMEM` | — | Memoria de equipo |
| `TEMPLATES` | — | Funcionalidad de plantillas |
| `FILE_PERSISTENCE` | — | Persistencia de archivos |
| `BG_SESSIONS` | — | Sesiones en segundo plano |
| `DOWNLOAD_USER_SETTINGS` | — | Descarga de configuración de usuario |
| `UPLOAD_USER_SETTINGS` | — | Subida de configuración de usuario |
| `NEW_INIT` | — | Nuevo flujo de inicialización |
| `HARD_FAIL` | — | Modo de fallo duro |
| `SLOW_OPERATION_LOGGING` | — | Registro de operaciones lentas |
| `SHOT_STATS` | — | Estadísticas de solicitudes |
| `MEMORY_SHAPE_TELEMETRY` | — | Telemetría de forma de memoria |
| `COWORKER_TYPE_TELEMETRY` | — | Telemetría de tipo de colaborador |
| `ANTI_DISTILLATION_CC` | — | Protección anti-destilación |
| `RUN_SKILL_GENERATOR` | — | Generador de Skills |
| `SKILL_IMPROVEMENT` | — | Mejora de Skills |
| `REVIEW_ARTIFACT` | — | Artefacto de revisión de código |
| `MESSAGE_ACTIONS` | — | Acciones de mensajes |
| `AWAY_SUMMARY` | — | Resumen de ausencia |
| `COMMIT_ATTRIBUTION` | — | Atribución de commits |
| `UNATTENDED_RETRY` | — | Reintento desatendido |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | Detección del tipo de libc (inyectada en tiempo de compilación) |

### Feature Flags en Tiempo de Ejecución vs. en Tiempo de Compilación

| Dimensión | Tiempo de compilación (`feature()`) | Tiempo de ejecución (GrowthBook) |
|---|---|---|
| Momento de ejecución | Fase de empaquetado | Carga asíncrona tras inicio del proceso |
| Conservación del código | Las ramas eliminadas no existen en el producto final | El código existe pero la lógica está controlada por el valor del flag |
| Modo de actualización | Publicar nueva versión | Publicación desde el servidor; efectivo en como mínimo 20 minutos |
| Usos típicos | Funcionalidades ant-only, herramientas experimentales, código específico de plataforma | Pruebas A/B, lanzamiento gradual, kill-switch, configuración dinámica |
| Modo de sobreescritura | Variables de compilación | Variable de entorno `CLAUDE_INTERNAL_FC_OVERRIDES` (solo ant) |

### Mecanismo de Dead Code Elimination

La función `feature()` de `bun:bundle` es una función especial integrada del empaquetador Bun; durante la fase de compilación, reemplaza directamente `feature('X')` con `true` o `false` según las definiciones de tiempo de compilación, y luego el plegado de constantes y la eliminación de código muerto descartan las ramas que siempre son falsas.

Ejemplo (`perfettoTracing.ts:216-220`):
```typescript
// en la construcción externa, este bloque if se elimina completamente
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... todo el código de inicialización de Perfetto
}
```

Este mecanismo no solo reduce el tamaño del paquete, sino que también impide que el código de herramientas internas quede expuesto en los productos externos.

### Feature Flags en Tiempo de Ejecución Conocidos e Importantes

A continuación, algunos flags de GrowthBook `tengu_*` conocidos y su funcionalidad:

| Nombre del Flag | Tipo | Descripción funcional |
|---|---|---|
| `tengu_auto_mode_config` | Object | Configuración del modo automático (enabled/opt-in) |
| `tengu_1p_event_batch_config` | Object | Configuración de exportación en lote de eventos 1P |
| `tengu_event_sampling_config` | Object | Diccionario de configuración de tasa de muestreo de eventos |
| `tengu_log_datadog_events` | Boolean | Interruptor de reporte de eventos a Datadog |
| `tengu_frond_boric` | Object | Kill-switch de sink de Analytics (deshabilitar por nombre de sink) |
| `tengu_quartz_lantern` | Boolean | Control del comportamiento de escritura atómica de FileWriteTool |
| `tengu_hive_evidence` | Boolean | Control del comportamiento de actualización de tareas/escritura en Todo |
| `tengu_plum_vx3` | Boolean | Interruptor para que WebSearchTool use el modelo Haiku |
| `tengu_kairos_cron` | Object | Configuración de tareas programadas de KAIROS |
| `tengu_kairos_cron_durable` | Boolean | Soporte de tareas programadas duraderas |
| `tengu_agent_list_attach` | Boolean | Comportamiento de adjunto de lista en AgentTool |
| `tengu_amber_stoat` | Boolean | Control de disponibilidad de agentes integrados |
| `tengu_slim_subagent_claudemd` | Boolean | Carga simplificada de CLAUDE.md en sub-agentes |
| `tengu_glacier_2xr` | Boolean | Control de decisiones del modo ToolSearch |
| `tengu_max_version_config` | Object | Límite de versión máxima (kill-switch de actualización forzada) |
| `tengu_prompt_cache_1h_config` | Object | Configuración de caché de prompt de 1 hora |
| `tengu_sm_compact_config` | Object | Configuración de compresión de Session Memory |
| `tengu_ant_model_override` | String | Sobreescritura de modelo exclusiva para ant |
| `enhanced_telemetry_beta` | Boolean | Interruptor beta de telemetría mejorada |

---

## 14.4 Sistema de Observabilidad

### Integración de OpenTelemetry

Claude Code implementa completamente el soporte de las tres señales de OpenTelemetry, con el punto de entrada principal en `src/utils/telemetry/instrumentation.ts` (825 líneas).

**Bootstrap de inicialización** (`instrumentation.ts:bootstrapTelemetry()`):
En las construcciones ant, lee la configuración de variables con prefijo `ANT_OTEL_*` y las mapea a las variables `OTEL_*` estándar. Para los usuarios externos, sigue la especificación de variables de entorno OTel estándar; la temporalidad predeterminada se establece en `delta` (incremental, no acumulativa).

**Configuración de exportadores de las tres señales** (diseño de carga perezosa):

```typescript
// instrumentation.ts:169-190 (simplificado)
// Los exportadores OTLP/Prometheus usan dynamic import de carga perezosa
// para evitar que @grpc/grpc-js (~700KB) se cargue cuando no es necesario
case 'grpc': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-grpc'
  )
  exporters.push(new OTLPMetricExporter())
  break
}
case 'http/protobuf': {
  const { OTLPMetricExporter } = await import(
    '@opentelemetry/exporter-metrics-otlp-proto'
  )
  exporters.push(new OTLPMetricExporter(httpConfig))
  break
}
```

Se soportan los tres protocolos de transporte: `grpc`, `http/json`, `http/protobuf`, seleccionados mediante la variable de entorno `OTEL_EXPORTER_OTLP_PROTOCOL`.

**Atributos de recurso**: el nombre del servicio es `claude-code`, llevando atributos de arquitectura de plataforma, versión de WSL, tipo de suscripción, versión del servicio, etc., rellenados automáticamente por `envDetector`, `hostDetector` y `osDetector`.

### Transmisión de Datos gRPC

gRPC es el protocolo de transporte recomendado para escenarios empresariales, proporcionando streaming bidireccional y codificación protobuf con tipado fuerte. En Claude Code:

- El exportador gRPC (`@opentelemetry/exporter-metrics-otlp-grpc`) se carga de forma perezosa para evitar afectar al tiempo de inicio
- La configuración mTLS se soporta a través de `getMTLSConfig()`; se pueden usar certificados autofirmados en redes internas empresariales
- El soporte de proxy se gestiona de forma transparente mediante `getProxyUrl()` + `HttpsProxyAgent`

Los subprocesos no heredan las variables de entorno relacionadas con OTEL (`subprocessEnv.ts`):
```typescript
// subprocessEnv.ts:24-28
// para backends de monitorización; leídas en proceso por el SDK OTEL, los subprocesos nunca las necesitan
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Rastreo Perfetto

Perfetto es un framework de rastreo a nivel de sistema de alto rendimiento desarrollado por Google. Claude Code implementa una capa de compatibilidad con el formato Chrome Trace Event (`src/utils/telemetry/perfettoTracing.ts`, 1.120 líneas, ant-only).

**Cómo habilitarlo**:
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # escribe en ~/.claude/traces/trace-<session-id>.json
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # escribe en la ruta especificada
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # escritura periódica cada 60 segundos
```

**Tipos de Spans rastreados**:

| Nombre del Span | Categoría | Información que lleva |
|---|---|---|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | número de intento |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**Gestión de memoria** (límite de 100.000 eventos):
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// al alcanzar el límite, elimina la mitad más antigua, amortizando el coste en O(1)
// inserta una marca trace_truncated para que el hueco sea visible en ui.perfetto.dev
```

**Rastreo jerárquico multi-Agent**: cada Agent (incluidos los sub-Agents) se mapea a un ID de proceso independiente, y las relaciones jerárquicas se registran mediante eventos de metadatos `parent_agent`, presentándose como carriles independientes en la UI de Perfetto.

**Estrategia de escritura** (triple garantía):
1. Callback asíncrono de `cleanup registry` (salida normal)
2. Handler `process.on('beforeExit')` (respaldo)
3. Escritura síncrona en `process.on('exit')` (última línea de defensa; no se puede usar async aquí)

### Session Tracing con OpenTelemetry

`src/utils/telemetry/sessionTracing.ts` (927 líneas) es el punto de entrada de telemetría mejorada orientada a usuarios externos, basado en Spans OTel estándar en lugar del formato Perfetto.

**Condiciones de habilitación** (`sessionTracing.ts:170-185`):
```typescript
export function isEnhancedTelemetryEnabled(): boolean {
  if (feature('ENHANCED_TELEMETRY_BETA')) {
    const env = process.env.CLAUDE_CODE_ENHANCED_TELEMETRY_BETA
      ?? process.env.ENABLE_ENHANCED_TELEMETRY_BETA
    if (isEnvTruthy(env)) return true
    if (isEnvDefinedFalsy(env)) return false
    return (
      process.env.USER_TYPE === 'ant' ||
      getFeatureValue_CACHED_MAY_BE_STALE('enhanced_telemetry_beta', false)
    )
  }
  return false
}
```

**Propagación de contexto con AsyncLocalStorage**: cada Interaction y Tool Call usa un almacenamiento ALS independiente para almacenar SpanContext, garantizando que los Spans no se mezclen en escenarios de concurrencia multi-Agent. Los WeakRef previenen fugas de memoria de Spans de larga duración; los Spans huérfanos de más de 30 minutos se limpian en intervalos de 60 segundos.

**Sistema de eventos logEvent**

Todos los eventos de negocio se despachan uniformemente a través de `logEvent()` en `src/services/analytics/index.ts`:

```typescript
// index.ts (simplificado)
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // solo permite boolean | number | undefined
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

Diseño clave: el tipo de metadata excluye deliberadamente `string`, obligando a los desarrolladores a usar la conversión de tipo `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`, previniendo a nivel de tipos el registro accidental de contenido de código o rutas de archivos.

---

## 14.5 Recopilación de Analytics

### Arquitectura de Enrutamiento de Doble Vía

Todos los eventos se enrutan a dos backends a través de `sink.ts`:

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog (si el gate tengu_log_datadog_events está abierto)
    │     solo transmite eventos en la lista blanca DATADOG_ALLOWED_EVENTS
    │     elimina las claves _PROTO_* (campos marcados como PII)
    └─→ 1P First-Party Logger (OpenTelemetry BatchLogRecordProcessor)
          envía a /api/event_logging/batch
          conserva las claves _PROTO_* (enrutadas a columnas protegidas de BigQuery)
```

**Integración con Datadog** (`datadog.ts`):
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Envío en lotes: 100 por lote, intervalo de actualización de 15 segundos
- Timeout de red: 5 segundos
- Mecanismo de lista blanca: aproximadamente 50 eventos principales (Set `DATADOG_ALLOWED_EVENTS`)
- Condiciones de deshabilitación: Bedrock/Vertex/Foundry en la nube de terceros, entornos de prueba, el usuario elige no-telemetry

**Registro de eventos 1P (FirstPartyEventLoggingExporter)**:
- Usa la interfaz estándar `LogRecordExporter` de OpenTelemetry
- Exportación en lotes: 200 por lote por defecto, retraso de planificación de 5 segundos
- Reintento ante fallos: backoff exponencial (base 500ms, máximo 30s, máximo 8 veces)
- Cola de fallos persistente: los eventos fallidos se escriben en `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl`; se reintentan en el próximo inicio
- Serialización Proto: usa el tipo protobuf `ClaudeCodeInternalEvent` generado

### Seguimiento del Comportamiento del Usuario

Más de 400 nombres de eventos `tengu_*` cubren el ciclo de vida completo de la interacción del usuario. Categorías de eventos principales:

**Ciclo de vida de la sesión**: `tengu_started`, `tengu_init`, `tengu_exit`, `tengu_cancel`

**Llamadas a la API**: `tengu_api_query`, `tengu_api_success`, `tengu_api_error`, `tengu_api_retry`

**Uso de herramientas**: `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_in_prompt_permanent`

**Solicitudes de permisos**: `tengu_internal_bash_tool_use_permission_request`, `tengu_tool_use_show_permission_request`, `tengu_tool_use_granted_by_classifier`

**Autenticación OAuth**: `tengu_oauth_flow_start`, `tengu_oauth_success`, `tengu_oauth_token_refresh_*` (rastreo completo de la máquina de estados de bloqueo)

**Servidores MCP**: `tengu_mcp_server_connection_succeeded`, `tengu_mcp_server_connection_failed`, `tengu_mcp_oauth_flow_*`

**Mecanismo de actualización**: `tengu_binary_download_attempt`, `tengu_native_update_complete`, `tengu_binary_download_failure`

### Recopilación de Métricas de Rendimiento

Los API Call Spans en `sessionTracing.ts` calculan las siguientes métricas derivadas:

```typescript
// perfettoTracing.ts (endLLMRequestPerfettoSpan simplificado)
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second (velocidad de procesamiento de tokens de entrada)

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second (velocidad de muestreo)

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // Tasa de aciertos de caché (porcentaje)
```

### Control de Muestreo de Eventos

La tasa de muestreo de cada evento se controla mediante la configuración dinámica de GrowthBook `tengu_event_sampling_config`:

```typescript
// firstPartyEventLogger.ts (shouldSampleEvent simplificado)
// devuelve null = 100% de muestreo (sin configuración)
// devuelve 0 = descarte completo
// devuelve rate (0-1) = muestreo aleatorio; escribe sample_rate en metadata
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // ejemplo: 10% de muestreo
}
```

### Reporte de Errores

Sistema de eventos de error multinivel:
- `tengu_uncaught_exception`, `tengu_unhandled_rejection`: errores no capturados a nivel de proceso
- `tengu_api_error`, `tengu_query_error`: errores en llamadas a la API
- `tengu_streaming_error`: errores en respuestas de streaming
- `tengu_atomic_write_error`: errores de escritura en archivos
- `tengu_compact_failed`: fallos de compresión de sesión

---

## 14.6 Diagnóstico y Depuración

### Comando /doctor

`src/commands/doctor/index.ts` registra el comando `/doctor`:

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

Este comando se ejecuta de tipo `local-jsx` (renderizando directamente un componente React dentro del REPL), y verifica: integridad de la instalación, estado de conexión de servidores MCP, validez de la configuración de atajos de teclado y dependencias del entorno (ripgrep, etc.).

### Sistema de Rastreo de Diagnósticos

En escenarios de integración con IDE, Claude Code recibe información de diagnóstico de código a través de Language Server Protocol. Cuando se guarda un archivo (evento `didSave`), el TypeScript Server envía nuevos mensajes de diagnóstico, que el sistema inyecta como tags XML `<new-diagnostics>` para pasarlos al modelo:

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### Diagnóstico de Memoria del Heap

`src/utils/heapDumpService.ts` proporciona capacidades de diagnóstico de memoria a nivel de proceso; al activar un heap dump, recoge de forma síncrona una instantánea del uso de memoria y la exporta a `~/Desktop/<session-id>-diagnostics.json`, incluyendo `heapUsed`, `external`, `rss` y sugerencias de análisis. Evento de analytics correspondiente: `tengu_heap_dump`.

### Registros de Recuperación de Errores

`src/utils/telemetry/bigqueryExporter.ts` implementa un exportador de métricas de BigQuery, integrado con el pipeline de métricas OTEL, para monitorización de rendimiento a largo plazo y planificación de capacidad interna de ant. La cola de persistencia `1p_failed_events` garantiza que no se pierdan eventos clave aunque haya fallos de red.

---

## 14.7 Análisis de Decisiones de Diseño

### Ventajas e Inconvenientes de los Flags en Tiempo de Compilación

**Ventajas**:
1. **Sin overhead en tiempo de ejecución**: las ramas de código eliminadas no existen en el producto final; sin ningún overhead de evaluación condicional
2. **Aislamiento de seguridad**: el código de funcionalidades ant-only es completamente invisible para los usuarios externos; no puede hacerse ingeniería inversa
3. **Optimización del tamaño del paquete**: los módulos grandes (como `@grpc/grpc-js` ~700KB) solo existen en las construcciones que los necesitan
4. **Seguridad de tipos**: la comprobación de tipos de TypeScript actúa antes del empaquetado, sin afectar al tiempo de ejecución

**Inconvenientes**:
1. **Dependencia de publicación**: cambiar el estado de un flag requiere publicar una nueva versión; no se puede actualizar en caliente
2. **Explosión de la matriz de pruebas**: N flags en tiempo de compilación requieren teóricamente 2^N combinaciones de construcciones para pruebas
3. **Complejidad de depuración**: cuando los usuarios externos reportan problemas, ciertas rutas de código simplemente no existen en su construcción

### El Equilibrio entre Privacidad y Observabilidad

Claude Code adopta múltiples líneas de defensa en la protección de la privacidad:

1. **Protección del sistema de tipos**: `LogEventMetadata` solo permite `boolean | number | undefined`; para registrar una cadena hay que declarar explícitamente `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`. Este es un tipo `never` que no puede contener ningún valor real; solo obliga al desarrollador a escribir la anotación de tipo, indicando que ha verificado manualmente que la cadena no contiene código ni rutas.

2. **Anonimización de nombres de herramientas MCP**: el formato de nombre de herramienta MCP `mcp__<server>__<tool>` puede revelar configuraciones de servicios privados del usuario; por defecto se anonimiza como `mcp_tool`. Solo se conserva el nombre original para servidores en el punto de entrada `cowork`, servidores dentro del registro oficial de MCP, o servidores declarados explícitamente como integrados.

3. **Campos marcados como PII**: las claves de metadata con prefijo `_PROTO_*` indican campos sensibles de PII, enrutadas solo a las columnas protegidas de BigQuery de 1P; `sink.ts` elimina estos campos antes de reenviarlos a Datadog.

4. **Deshabilitación en nubes de terceros**: para clientes empresariales que usan Bedrock/Vertex/Foundry, todos los analytics del lado de Anthropic (incluyendo Datadog y 1P) están deshabilitados por defecto.

### La Razón de la Carga Perezosa de Telemetría

Los paquetes relacionados con OTLP (gRPC ~700KB, proto ~300KB) se cargan de forma perezosa mediante `import()` dinámico, porque:

1. **Sensibilidad al tiempo de inicio**: la métrica de rendimiento principal de las herramientas CLI es el Time-to-First-Output; cualquier inicialización innecesaria debe posponerse
2. **Protocolos mutuamente excluyentes**: un proceso solo usará un protocolo de transporte; importar estáticamente todas las variantes (6 paquetes) es puro desperdicio
3. **Compatibilidad con la optimización de Bun**: la carga perezosa es compatible con el patrón de optimización de resolución de módulos de Bun; los analiza estáticamente y los empaqueta bajo demanda

---

## 14.8 Patrones Transferibles

Los siguientes patrones de diseño tienen alto valor de referencia para otros proyectos:

### 1. Sistema de Tipos para Prevenir Fugas de PII

Mediante un tipo marcador de tipo `never`, se obliga en tiempo de compilación a que los desarrolladores confirmen explícitamente que no hay información sensible. El coste es cero (sin overhead en tiempo de ejecución); la protección es del 100% (eludirlo requiere una aserción de tipo explícita). Aplicable a cualquier sistema con necesidades de reporte de datos.

### 2. Arquitectura de Feature Flags de Doble Nivel

Tiempo de compilación (capas de código) + tiempo de ejecución (control de comportamiento), correspondiendo a diferentes necesidades de velocidad de publicación:
- Funcionalidades estructurales (presencia/ausencia de módulos enteros) → tiempo de compilación
- Ajuste de comportamiento (parámetros, proporciones, selección de algoritmo) → tiempo de ejecución

### 3. Patrón Sink Kill-Switch

La configuración de GrowthBook `tengu_frond_boric` permite deshabilitar de forma independiente cualquier backend de analytics por nombre (`datadog`, `firstParty`) sin necesidad de publicar una nueva versión. Este es un patrón de disyuntor de emergencia genérico, adecuado para todos los sistemas de eventos con múltiples sinks descendentes.

### 4. Reintento con Persistencia de Eventos Fallidos

Cuando falla la exportación de eventos 1P, se escriben en un archivo JSONL local y se reintentan en el siguiente inicio. Esto garantiza que los datos de telemetría críticos no se pierdan durante fallos de red, especialmente adecuado para herramientas que se ejecutan en entornos de red inestables o sin conexión.

### 5. Deduplicación de Exposición de Experimentos

Los eventos de exposición de experimentos de GrowthBook (para análisis de resultados de pruebas A/B) se deduplicarán a través de un Set a nivel de sesión, asegurando que la exposición de un mismo feature se registre solo una vez en el lado del análisis, previniendo que múltiples llamadas al mismo flag inflen artificialmente el recuento de exposiciones.

---

## 14.9 Índice de Código Fuente

| Ruta del archivo (relativa a `src/`) | Líneas | Responsabilidad principal |
|---|---|---|
| `services/analytics/growthbook.ts` | 1.155 | Integración del SDK GrowthBook, lectura de Feature Flags, registro de exposiciones A/B |
| `services/analytics/index.ts` | 173 | API pública de logEvent, cola de eventos, definición de interfaz Sink |
| `services/analytics/sink.ts` | 114 | Implementación de enrutamiento de doble vía (Datadog + 1P), inicialización |
| `services/analytics/datadog.ts` | 307 | Envío en lotes a Datadog, filtrado de lista blanca |
| `services/analytics/firstPartyEventLogger.ts` | 449 | Inicialización de LoggerProvider de OpenTelemetry, control de muestreo |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | Exportación HTTP de eventos 1P, reintento con persistencia, serialización proto |
| `services/analytics/metadata.ts` | 973 | Enriquecimiento de metadatos de eventos, anonimización de nombres de herramientas MCP, manejo de PII |
| `services/analytics/config.ts` | 38 | Lógica compartida de isAnalyticsDisabled() |
| `services/analytics/sinkKillswitch.ts` | 25 | Kill-Switch a nivel de Sink (tengu_frond_boric) |
| `utils/telemetry/instrumentation.ts` | 825 | Inicialización del SDK OTel, configuración de las tres señales (Metrics/Logs/Traces) |
| `utils/telemetry/sessionTracing.ts` | 927 | Gestión de Spans OTel, propagación de contexto con AsyncLocalStorage |
| `utils/telemetry/perfettoTracing.ts` | 1.120 | Rastreo en formato Chrome Trace de Perfetto (ant-only) |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Atributos extendidos de rastreo beta |
| `utils/telemetry/bigqueryExporter.ts` | 252 | Exportador de métricas de BigQuery |
| `utils/telemetry/pluginTelemetry.ts` | 289 | Encapsulación de telemetría de plugins |
| `utils/telemetry/events.ts` | 75 | Definiciones de tipos de eventos OTel |
| `commands/doctor/index.ts` | 12 | Registro del comando /doctor |
| `commands.ts` | — | Punto centralizado de llamadas a `feature()` en tiempo de compilación |
