# Informe de Análisis Arquitectónico Completo de Claude Code

> **Línea base del código fuente**: Instantánea del código TypeScript de Claude Code (2026-03-31, ~512K LOC, ~1,900 archivos)
> **Fecha de análisis**: 2026-04-02
> **Escala del informe**: 14 capítulos, 428KB

---

## Contexto del Proyecto

### Origen del Código Fuente

Este informe se basa en la instantánea completa del código TypeScript de Claude Code filtrada el 2026-03-31. La instantánea contiene 512,664 líneas de código TypeScript (`.ts` + `.tsx`), distribuidas en 1,884 archivos que abarcan 35 directorios de nivel superior. El código fuente se almacena en el servidor mini en `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`.

### Metodología de Análisis

Se adopta una arquitectura de **análisis paralelo con 14 sub-agentes**: 5 agentes Opus responsables de los capítulos con mayor complejidad central (visión general de la arquitectura, Query Engine, orquestación de agentes, modelo de seguridad, gestión de contexto), y 9 agentes Sonnet responsables de los capítulos restantes. Cada agente lee el código fuente de forma independiente, extrae patrones arquitectónicos y verifica conclusiones del análisis competitivo; el editor jefe realiza la revisión final unificada.

Esta metodología en sí misma es una validación práctica de la arquitectura multi-agente de Claude Code (Capítulo 4): usar Claude Code para analizar Claude Code.

### Comparación con win4r/cc-notebook

win4r/cc-notebook es el análisis previo de la comunidad sobre el mismo código fuente. Este informe realiza mejoras significativas en las siguientes dimensiones:

- **Capítulo independiente sobre el sistema de herramientas** (Capítulo 3): cc-notebook no analiza el sistema de herramientas por separado; este informe llena ese vacío crítico
- **Verificación a nivel de código fuente**: cada afirmación arquitectónica incluye nombre de archivo, número de línea y fragmento de código, no una descripción de segunda mano
- **Anclaje teórico**: cada capítulo comienza con fundamentos de teoría académica (teoría de la información, teoría de caché, ciencias cognitivas, etc.), colocando la implementación de ingeniería en un marco de conocimiento más amplio

---

## Diagrama de Arquitectura General

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Arquitectura en Capas de Claude Code                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                Capa de Interacción con el Usuario               │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │ Terminal  │  │ Command  │  │  Slash   │  │  Vim Mode     │  │    │
│  │  │ UI (Ink)  │  │ Parser   │  │ Commands │  │  (Máq. Est.)  │  │    │
│  │  │  Cap.12   │  │  Cap.7   │  │  Cap.7   │  │  Cap.12       │  │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────────┘  │    │
│  └───────┼──────────────┼──────────────┼──────────────────────────┘    │
│          │              │              │                                 │
│  ┌───────▼──────────────▼──────────────▼──────────────────────────┐    │
│  │              Capa de Orquestación de Sesiones                   │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │    │
│  │  │ Query Engine │  │    Agent     │  │  Prompt Engineering  │ │    │
│  │  │  (bucle ppal)│  │ Orchestrator │  │  (orq. system prmpt) │ │    │
│  │  │  Cap.2       │  │  Cap.4       │  │  Cap.5               │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │    │
│  └─────────┼────────────────┼──────────────────────┼─────────────┘    │
│            │                │                      │                    │
│  ┌─────────▼────────────────▼──────────────────────▼─────────────┐    │
│  │                  Capa de Ejecución de Capacidades              │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │ Tool     │  │  Skill   │  │  MCP     │  │  Bash/Read/  │  │    │
│  │  │ System   │  │  System  │  │ Client   │  │  Edit/Grep   │  │    │
│  │  │  Cap.3   │  │  Cap.6   │  │  Cap.11  │  │  (herr. int.)│  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │               Capa de Persistencia de Estado                   │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │    │
│  │  │ Context Mgmt │  │   Memory     │  │  Session Resume  │    │    │
│  │  │ (compresión) │  │  System      │  │  (reanud. ses.)  │    │    │
│  │  │  Cap.8       │  │  Cap.9       │  │  Cap.9           │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘    │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                       Capa de Infraestructura                  │    │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ Security │  │ Cost Tracker │  │ Feature  │  │  OTel    │ │    │
│  │  │ & Perms  │  │ & Model Sel. │  │  Flags   │  │ Telemetry│ │    │
│  │  │  Cap.10  │  │  Cap.13      │  │  Cap.14  │  │  Cap.14  │ │    │
│  │  └──────────┘  └──────────────┘  └──────────┘  └──────────┘ │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dependencias Principales entre Subsistemas

```
Query Engine ──────► Claude API (claude.ts, 3,419 líneas)
     │
     ├──► Tool System ──► 40+ herramientas integradas
     │         │
     │         ├──► MCP Client ──► Servidores MCP externos
     │         │
     │         └──► Skill System ──► Definiciones de flujo de trabajo en Markdown
     │
     ├──► Agent Orchestrator ──► Motor compartido runAgent()
     │         │
     │         ├──► Subagent (en proceso)
     │         ├──► Coordinator Mode (notificación asíncrona)
     │         └──► Team Mode (buzón de archivos / Tmux)
     │
     ├──► Context Management ──► micro-compresión → compresión completa (degradación progresiva)
     │
     ├──► Security Layer ──► 8 capas de defensa en profundidad
     │         │
     │         ├──► Análisis AST (parser Bash)
     │         ├──► Sandbox (sandbox-exec / bwrap)
     │         └──► Permission Rules + Hooks
     │
     └──► Cost Tracker ──► facturación en tiempo real por modelo
```

### Flujo de Datos (Ciclo de vida de una interacción de usuario)

```
Entrada del usuario → Command Parser → Enrutamiento REPL
                               │
                    ┌──────────┴──────────┐
                    ▼                      ▼
              Comando slash           Entrada en lenguaje natural
              (ejecución inmediata)        │
                                           ▼
                                   Prompt Assembly
                                   (system prompt + CLAUDE.md
                                    + descripciones de herramientas + historial)
                                           │
                                           ▼
                              ┌── Bucle principal Query Engine ──┐
                              │                                   │
                              ▼                                   │
                        Solicitud streaming Claude API            │
                              │                                   │
                              ▼                                   │
                     ┌── Determinación tipo respuesta ──┐        │
                     │                                  │        │
                     ▼                                  ▼        │
                  Salida de texto               Llamada a herramienta│
                  (render en UI)                     │            │
                                                     ▼            │
                                             Verificación permisos│
                                                     │            │
                                              ┌──────┴──────┐    │
                                              ▼              ▼    │
                                           Permitir       Denegar/Preguntar│
                                              │              │    │
                                              ▼              │    │
                                         Ejecución herram.  │    │
                                              │              │    │
                                              ▼              ▼    │
                                         Resultado inyectado ──►  │
                                              │                   │
                                              ▼                   │
                                     Verificación ventana contexto→ ¿Comprimir?│
                                              │                   │
                                              └───────────────────┘
                                                    (bucle hasta que el modelo pare)
```

---

## Estadísticas de Tamaño del Código

### Distribución de Líneas de Código por Subsistema

| Subsistema | Líneas de código centrales | Archivos centrales | Porcentaje |
|--------|-------------|-----------|------|
| Seguridad y permisos (Cap.10) | ~25,000 | 30+ | 4.9% |
| Integración MCP (Cap.11) | ~12,310 | 10+ | 2.4% |
| Orquestación de agentes (Cap.4) | ~8,700 | 12 | 1.7% |
| Query Engine (Cap.2) | ~7,418 | 8 | 1.4% |
| Sistema de memoria (Cap.9) | ~5,700 | 17 | 1.1% |
| Gestión de contexto (Cap.8) | ~6,000 | 13+ | 1.2% |
| Sistema de herramientas (Cap.3) | ~4,000+ | 40+ directorio | 0.8%+ |
| Capa API (claude.ts) | 3,419 | 1 | 0.7% |
| **Subtotal anterior** | **~72,500+** | — | **~14%** |
| Resto (UI/renderizado/Skill/comandos/config, etc.) | ~440,000 | — | ~86% |
| **Total** | **512,664** | **1,884** | **100%** |

### Top 20 Archivos Centrales (por líneas)

| Rango | Archivo | Líneas | Subsistema |
|------|------|------|-----------|
| 1 | `services/api/claude.ts` | 3,419 | Llamadas API/streaming/reintentos/caché |
| 2 | `services/mcp/client.ts` | 3,348 | Gestión de conexiones MCP |
| 3 | `services/mcp/auth.ts` | 2,465 | Autenticación OAuth MCP |
| 4 | `services/teamMemorySync/` (5 archivos) | 2,167 | Sincronización de memoria de equipo |
| 5 | `query.ts` | 1,729 | Bucle principal de consulta |
| 6 | `memdir/` (7 archivos) | 1,736 | Gestión de directorio de memoria |
| 7 | `services/tools/toolExecution.ts` | 1,745 | Motor de ejecución de herramientas |
| 8 | `services/mcp/config.ts` | 1,578 | Gestión de configuración MCP |
| 9 | `inProcessRunner.ts` | 1,552 | Backend InProcess del agente |
| 10 | `AgentTool.tsx` | 1,397 | Punto de entrada unificado del agente |
| 11 | `QueryEngine.ts` | 1,295 | Gestión de estado a nivel de sesión |
| 12 | `teammateMailbox.ts` | 1,183 | Protocolo de buzón de archivos |
| 13 | `utils/collapseReadSearch.ts` | 1,109 | Compresión de resultados de lectura/búsqueda |
| 14 | `spawnMultiAgent.ts` | 1,093 | Generación de múltiples agentes |
| 15 | `utils/toolResultStorage.ts` | 1,040 | Almacenamiento en frío de resultados de herramientas |
| 16 | `runAgent.ts` | 973 | Motor de ejecución de agentes |
| 17 | `SendMessageTool.ts` | 917 | Enrutamiento de mensajes de 5 vías |
| 18 | `UI.tsx` (Agent) | 871 | Renderizado de progreso del agente |
| 19 | `Tool.ts` | 792 | Abstracción central de herramientas |
| 20 | `extractMemories/` (2 archivos) | 769 | Extracción de memoria |

---

## Guía de Capítulos

### Capítulo 1: Visión General de la Arquitectura y Flujo de Inicio
**Archivo**: [01-architecture-overview.md](01-architecture-overview.md)

**Perspectiva clave**: Claude Code es una aplicación TUI de terminal basada en React/Ink, con Bun como runtime preferido, que impulsa el API de Claude a través de un bucle REPL para completar tareas de programación agentic.

**Hallazgos clave**:
- Selección de stack tecnológico precisa: Bun + TypeScript + React/Ink + Zod v4 + OpenTelemetry, cada elección con razones de ingeniería claras
- 512,664 líneas de código distribuidas en 1,884 archivos y 35 directorios de nivel superior; un proyecto de ingeniería maduro de gran escala
- El sistema de feature flags adopta un esquema dual GrowthBook + bun:bundle, eliminando funciones internas en tiempo de compilación

**Prioridad recomendada**: ★★★★★ — Lectura obligatoria como punto de partida para establecer el modelo mental global

---

### Capítulo 2: Query Engine — Núcleo de Interacción LLM
**Archivo**: [02-query-engine.md](02-query-engine.md)

**Perspectiva clave**: El Query Engine es el "latido" de Claude Code; con 7,400 líneas de código (solo el 1.4%), impulsa la ruta más crítica del producto: cada interacción del usuario con Claude debe pasar por aquí.

**Hallazgos clave**:
- La arquitectura central es un Async Generator Pipeline (tubería de corrutinas), orquestando respuestas streaming, llamadas a herramientas, compresión de contexto y seguimiento de costos en un único async generator
- Maneja al menos 12 ramas de excepción: desbordamiento de ventana de contexto, agotamiento de presupuesto de tokens, fallos de API, interrupciones del usuario, aprobaciones de permisos, etc.
- El mecanismo de presupuesto de tokens y decisión de auto-continue garantiza que las tareas largas no se interrumpan por las paradas del modelo

**Prioridad recomendada**: ★★★★★ — Clave para entender el funcionamiento de todo el producto

---

### Capítulo 3: Sistema de Herramientas
**Archivo**: [03-tool-system.md](03-tool-system.md)

**Perspectiva clave**: El sistema de herramientas es el único canal entre la intención del LLM y el mundo real: no un accesorio, sino un activo de ingeniería central.

**Hallazgos clave**:
- Patrón de Self-Describing Tools: cada herramienta declara sus capacidades mediante campos como `searchHint`, `prompt`, `userFacingSchema` para selección dinámica por el modelo
- 40+ subdirectorios de herramientas, todos orquestados por `toolExecution.ts` (1,745 líneas)
- cc-notebook no analiza este sistema por separado; este capítulo llena el vacío crítico

**Prioridad recomendada**: ★★★★☆ — Lectura obligatoria para desarrolladores de agentes

---

### Capítulo 4: Orquestación de Agentes y Arquitectura Multi-Agente
**Archivo**: [04-agent-orchestration.md](04-agent-orchestration.md)

**Perspectiva clave**: Tres modos de colaboración progresivos (Subagent / Coordinator / Team Mode) comparten el mismo motor central `runAgent()`, logrando diferentes comportamientos mediante combinación de parámetros — una de las decisiones de diseño más elegantes.

**Hallazgos clave**:
- 8,700 líneas de código, 12 módulos centrales; el subsistema más complejo de toda la arquitectura del producto
- El Team Mode usa el protocolo de buzón de archivos (`teammateMailbox.ts`, 1,183 líneas) para colaboración paralela persistente
- El Coordinator Mode usa XML `<task-notification>` para comunicación completamente asíncrona, con AbortController independiente para aislamiento

**Prioridad recomendada**: ★★★★★ — Ejemplo de libro de texto para diseño de sistemas multi-agente

---

### Capítulo 5: Ingeniería de Prompts
**Archivo**: [05-prompt-engineering.md](05-prompt-engineering.md)

**Perspectiva clave**: La ingeniería de prompts es el subsistema con mayor complejidad implícita en todo el sistema: un ajuste de 3 líneas en `systemPromptSection` puede afectar simultáneamente el comportamiento del modelo, la tasa de aciertos de Prompt Cache, la facturación de tokens y la consistencia entre sesiones.

**Hallazgos clave**:
- Un system prompt de 8,000+ tokens no es una "descripción", sino una programación precisa del comportamiento
- La estrategia de capas de Prompt Cache (`cacheScope: 'org'` / `'global'`) reduce el costo de tokens de millones de solicitudes API en un orden de magnitud
- El marcador `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` delimita con precisión el límite del prefijo compartido, maximizando los aciertos de caché global

**Prioridad recomendada**: ★★★★★ — Lectura obligatoria para ingenieros de prompts; ejemplo de diseño de prompts a escala comercial

---

### Capítulo 6: Sistema de Skills
**Archivo**: [06-skill-system.md](06-skill-system.md)

**Perspectiva clave**: Un Skill es un "SOP para la IA" — sedimenta los flujos de trabajo de expertos humanos en formato Markdown, dando a la IA capacidad de ejecución profesional reproducible.

**Hallazgos clave**:
- Fusión declarativa + ejecutiva: el Frontmatter declara metadatos (permisos, modelo, condiciones de activación), el cuerpo son instrucciones de ejecución
- Carga desde múltiples fuentes con fusión por prioridad: bundled / nivel-usuario / nivel-proyecto / nivel-Plugin / fuente MCP
- Mecanismo de activación condicional: activación automática según rutas de archivo mediante frontmatter `paths`, con soporte para descubrimiento dinámico

**Prioridad recomendada**: ★★★★☆ — Tiene valor de referencia directa para el diseño de Skills de Doramagic

---

### Capítulo 7: Sistema de Comandos
**Archivo**: [07-command-system.md](07-command-system.md)

**Perspectiva clave**: El sistema de comandos refleja una clara separación de responsabilidades: los comandos son responsables de "disparar", las herramientas de "ejecutar", el LLM de "decidir".

**Hallazgos clave**:
- Tres tipos de comandos: PromptCommand (inyectar en conversación LLM), LocalCommand (ejecución local), LocalJSXCommand (con renderizado UI)
- Diseño de carga perezosa: los comandos se cargan de forma diferida mediante `load(): Promise<Module>`, distribuyendo el costo de inicio a la primera llamada
- Dos extensiones clave del patrón Command clásico de GoF: carga perezosa + valores de retorno tipados

**Prioridad recomendada**: ★★☆☆☆ — Aplicación de patrones de diseño convencionales, leer según necesidad

---

### Capítulo 8: Gestión de Contexto
**Archivo**: [08-context-management.md](08-context-management.md)

**Perspectiva clave**: La gestión de contexto es esencialmente un problema de compresión de información: Claude Code diseña un sistema de degradación progresiva de sin pérdida a con pérdida, manteniendo la continuidad de sesiones de horas dentro de una ventana de 200K tokens.

**Hallazgos clave**:
- Tres capas de estrategia de compresión: micro-compresión (cache_edits, sin pérdida) → colapso de resultados de herramientas (collapseReadSearch) → compresión completa con Fork Agent (con pérdida)
- Las 9 secciones del prompt de resumen definen una función de distorsión implícita: qué información no puede perderse
- Invariantes de seguridad: los pares tool_use/tool_result no pueden separarse, protección recursiva, mecanismo Circuit Breaker

**Prioridad recomendada**: ★★★★☆ — Referencia clave para el desarrollo de agentes de sesiones largas

---

### Capítulo 9: Sistema de Memoria
**Archivo**: [09-memory-system.md](09-memory-system.md)

**Perspectiva clave**: El sistema de memoria permite a Claude mantener continuidad entre sesiones: de la tricotomía cognitiva de memoria de trabajo/memoria episódica/memoria semántica, mapeada a una arquitectura de tres capas de ventana de contexto/Session Memory/Persistent Memory.

**Hallazgos clave**:
- 5,700 líneas de código implementan el ciclo completo de vida de la memoria entre sesiones: extracción, almacenamiento, sincronización, carga
- La sincronización de memoria de equipo (`teamMemorySync/`, 2,167 líneas) es el módulo individual más grande, que soporta escenarios de colaboración multi-persona
- El directorio `memdir/` implementa una estructura de organización de memoria similar a un sistema de archivos

**Prioridad recomendada**: ★★★☆☆ — Valioso como referencia para agentes que requieren memoria a largo plazo

---

### Capítulo 10: Modelo de Seguridad y Permisos
**Archivo**: [10-security-permission.md](10-security-permission.md)

**Perspectiva clave**: El sistema de seguridad es el subsistema con mayor cantidad de código (~25,000 líneas), implementando 8 capas de defensa en profundidad: desde análisis AST hasta sandbox del SO, cada capa independiente y acumulable.

**Hallazgos clave**:
- Modelo de amenaza único: el propio modelo de IA puede ser guiado por prompt injection para ejecutar operaciones peligrosas
- Parser de Bash AST implementado en TypeScript puro, para entender la estructura de comandos (las expresiones regulares no pueden distinguir `echo "rm -rf /"` de `rm -rf /`)
- Doble sandbox: verificación de permisos en capa de aplicación + aislamiento en capa del SO (macOS sandbox-exec / Linux bwrap); incluso si la capa de aplicación falla, el SO actúa como respaldo

**Prioridad recomendada**: ★★★★★ — Lectura obligatoria para ingenieros de seguridad; referencia para el diseño de seguridad en agentes de IA

---

### Capítulo 11: Integración MCP
**Archivo**: [11-mcp-integration.md](11-mcp-integration.md)

**Perspectiva clave**: Claude Code es un cliente MCP puro; 12,310 líneas de código implementan gestión completa de conexiones, autenticación OAuth, descubrimiento de herramientas y registro dinámico.

**Hallazgos clave**:
- Las herramientas MCP se registran dinámicamente con el formato `mcp__<serverName>__<toolName>`, compartiendo el mismo framework de ejecución que las herramientas integradas
- La autenticación OAuth 2.0 incluye soporte para XAA (Cross-Application Access, acceso entre aplicaciones)
- Soporte para cuatro capas de transporte: stdio/SSE/HTTP Streamable/WebSocket

**Prioridad recomendada**: ★★★☆☆ — Lectura obligatoria para desarrolladores del ecosistema MCP

---

### Capítulo 12: UI de Terminal y Motor de Renderizado
**Archivo**: [12-terminal-ui.md](12-terminal-ui.md)

**Perspectiva clave**: Claude Code realiza modificaciones profundas al framework React/Ink: renderizado de doble búfer, motor ANSI diff, aceleración de desplazamiento por hardware, posicionamiento de cursor con conciencia IME, logrando una experiencia interactiva cercana a GUI en un terminal de texto puro.

**Hallazgos clave**:
- El intervalo de cuadro del pipeline de renderizado es `FRAME_INTERVAL_MS = 16` (teóricamente 62.5fps), alineado con la frecuencia de actualización del monitor
- Implementación completa del modo Vim: máquina de estados independiente, cubriendo modos NORMAL/INSERT, conjunto completo de operators/motions/textObjects
- Renderizador Ink personalizado (`src/ink/`) que reemplaza el pipeline de renderizado nativo de Ink

**Prioridad recomendada**: ★★☆☆☆ — Referencia profunda para desarrolladores de frameworks TUI; otros lectores pueden omitirlo

---

### Capítulo 13: Selección de Modelo y Control de Costos
**Archivo**: [13-model-cost.md](13-model-cost.md)

**Perspectiva clave**: Tres filosofías de diseño: prioridad a la intención del usuario (cadena de cobertura múltiple), total transparencia de costos (impresión obligatoria de tarifas), sin degradación silenciosa (el Overload Fallback debe mostrar alertas).

**Hallazgos clave**:
- Cadena de prioridad de selección de modelo: comando `/model` → flag `--model` → variable de entorno → archivo de configuración; la capa superior cubre la inferior
- Tabla de precios exactos integrada (`modelCost.ts`), con seguimiento de costos en tiempo real por modelo
- El Overload Fallback de Opus → Sonnet nunca cambia silenciosamente; debe mostrar advertencia al usuario

**Prioridad recomendada**: ★★★☆☆ — Referencia práctica para el diseño de enrutamiento de sistemas multi-modelo

---

### Capítulo 14: Feature Flags y Observabilidad
**Archivo**: [14-feature-flags-observability.md](14-feature-flags-observability.md)

**Perspectiva clave**: El Feature Flag de doble vía (DCE en tiempo de compilación bun:bundle + GrowthBook en tiempo de ejecución) junto con los tres pilares de OpenTelemetry, construye un sistema de observabilidad de extremo a extremo desde la eliminación en compilación hasta el seguimiento en ejecución.

**Hallazgos clave**:
- Los flags de tiempo de compilación eliminan completamente el código interno mediante Dead Code Elimination, evitando la ingeniería inversa
- Analytics de doble vía: Datadog (monitoreo externo) + registro de eventos internos 1P (BigQuery interno), nombres de eventos con prefijo `tengu_*`
- Diseño de privacidad primero: por defecto no se registra el contenido del código ni las rutas de archivos
