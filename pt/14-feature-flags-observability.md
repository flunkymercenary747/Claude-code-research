# Capítulo 14: Feature Flags e Observabilidade

## 14.1 Visão Geral e Posicionamento

O sistema de observabilidade do Claude Code é um sistema multinível e multi-objetivo, cobrindo desde o ajuste de funcionalidades em tempo de compilação até o rastreamento de comportamento em tempo de execução. Todo o sistema é composto por três pilares:

1. **Sistema de Feature Flags**: design de trilho duplo — chamadas `feature()` em tempo de compilação (implementando Dead Code Elimination via `bun:bundle`) e configuração dinâmica GrowthBook em tempo de execução; o primeiro controla os limites de funcionalidades entregues a diferentes grupos de usuários; o segundo suporta ajuste de interruptores de funcionalidades sem republicação.

2. **Pipeline de Observabilidade**: baseado no padrão OpenTelemetry, suportando três protocolos de exportação gRPC/HTTP/Protobuf, coletando uniformemente três tipos de sinais — Metrics, Logs, Traces — e fornecendo uma camada de depuração interna via formato de rastreamento Perfetto.

3. **Coleta de Analytics**: roteamento de dois caminhos — Datadog (monitoramento externo) + log de eventos 1P First-Party (BigQuery/proto interno), identificando todos os eventos de negócio com o prefixo `tengu_*`, usando configuração dinâmica de amostragem do GrowthBook para controlar o volume de dados.

O princípio central de design desse sistema é **isolamento em camadas**: privacidade do usuário em primeiro lugar (por padrão não registra conteúdo de código e caminhos de arquivo), diferença entre builds internos e externos (ant-only vs external), graceful degradation (cada camada tem kill-switch).

---

## 14.2 Bases Teóricas

### Desenvolvimento Orientado por Feature Flag

Feature Flags (sinalizadores de funcionalidade) permitem que equipes desenvolvam funcionalidades em diferentes estágios em paralelo no mesmo codebase, habilitando-as conforme necessário. O Claude Code adota um mecanismo de flag em duas camadas:

- **Flags em tempo de compilação**: via chamadas `feature()` fornecidas por `bun:bundle`, fazendo Dead Code Elimination durante o empacotamento. Blocos inteiros de código inexistentes na versão externa são completamente removidos, reduzindo não apenas o tamanho do pacote, mas também prevenindo a engenharia reversa da lógica interna.
- **Flags em tempo de execução**: obtidas dinamicamente do servidor via SDK GrowthBook, suportando cenários de teste A/B, implantação gradual, kill-switch de emergência, etc.

### Os Três Pilares da Observabilidade

A comunidade OpenTelemetry define a observabilidade como três grandes sinais (Three Pillars of Observability):

- **Metrics (métricas)**: dados numéricos de série temporal, como latência de API, consumo de tokens. O Claude Code usa `@opentelemetry/sdk-metrics` exportando via PeriodicExportingMetricReader a cada 60 segundos.
- **Logs (registros)**: registro estruturado de eventos. Todas as chamadas `logEvent()` são exportadas em lote via `LoggerProvider` + `BatchLogRecordProcessor` do OTel.
- **Traces (rastreamentos)**: cadeias de chamadas distribuídas. O Claude Code estabelece uma árvore hierárquica de Spans Interaction → LLM Request → Tool Call via `sessionTracing.ts`, suportando rastreamento de relações pai-filho em cenários multi-Agent.

### Aplicação de Teste A/B em Ferramentas CLI

Diferente de produtos Web, o teste A/B em ferramentas CLI enfrenta desafios únicos: sem fingerprint de browser, múltiplas plataformas e canais de distribuição, cenários de execução offline. As estratégias de resposta do Claude Code:

- Targetização por dimensão do usuário: `GrowthBookUserAttributes` carrega atributos como `platform`, `subscriptionType`, `rateLimitTier`, suportando experimentos em camadas.
- Cache em disco local: após cada busca bem-sucedida de valores de funcionalidades do servidor, grava em `cachedGrowthBookFeatures` no `~/.claude/config.json`, garantindo que o último valor conhecido seja usado quando offline.
- Deduplicação de exposição: eventos de exposição de experimentos para cada feature são registrados apenas uma vez na mesma sessão (`loggedExposures` Set).

---

## 14.3 Sistema de Feature Flags

### Integração GrowthBook

GrowthBook é uma plataforma open-source de Feature Flag e teste A/B. O Claude Code integra via SDK oficial `@growthbook/growthbook`, com arquivo em `src/services/analytics/growthbook.ts` (1155 linhas).

**Fluxo de inicialização**:

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

Design chave: `memoize` garante que o cliente GrowthBook seja inicializado apenas uma vez durante todo o ciclo de vida do processo; quando a autenticação muda (login/logout), o cliente é destruído e recriado via `refreshGrowthBookAfterAuthChange()`, em vez de tentar atualizar `apiHostRequestHeaders` (o SDK não suporta atualização após inicialização).

**Modelo de atributos do usuário** (`growthbook.ts:31-46`):

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

**Estratégia de atualização**:
- Usuários externos: atualização a cada 6 horas (`6 * 60 * 60 * 1000`)
- Funcionários internos (ant): atualização a cada 20 minutos

**Arquitetura de cache** (três níveis de prioridade):
1. Map `remoteEvalFeatureValues` em memória (valor mais recente no processo)
2. Cache em disco `cachedGrowthBookFeatures` em `~/.claude/config.json` (persistência entre processos)
3. `cachedStatsigGates` antigo (camada de compatibilidade de migração, sendo gradualmente eliminada)

**Workaround de compatibilidade de API** (`growthbook.ts:320-390`): a resposta remoteEval do servidor usa o campo `value`, enquanto o SDK espera `defaultValue`; o código tem lógica explícita de conversão de formato com comentário TODO aguardando correção do servidor.

**Substituição por variável de ambiente** (apenas para usuários internos ant):
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_auto_mode_config": {"enabled": "opt-in"}}'
```

### Lista Completa de Feature Flags em Tempo de Compilação (80+)

Implementados via chamadas `feature()` do `bun:bundle` para eliminação de código morto; a seguir estão todos os flags em tempo de compilação extraídos do código-fonte:

| Nome do Flag | Localização | Funcionalidade Controlada |
|-------------|------------|--------------------------|
| `PERFETTO_TRACING` | `perfettoTracing.ts` | Rastreamento Perfetto (ant-only) |
| `ENHANCED_TELEMETRY_BETA` | `sessionTracing.ts` | Beta de telemetria aprimorada |
| `KAIROS` | `commands.ts`, `EnterPlanModeTool` | Sistema de modo automático/tarefas agendadas |
| `KAIROS_BRIEF` | `commands.ts` | Modo simplificado KAIROS |
| `KAIROS_CHANNELS` | `ExitPlanModeTool`, `EnterPlanModeTool` | Suporte a canais KAIROS |
| `KAIROS_DREAM` | `commands.ts` | Modo sonho KAIROS |
| `KAIROS_GITHUB_WEBHOOKS` | `commands.ts` | Assinatura de webhooks GitHub |
| `KAIROS_PUSH_NOTIFICATION` | `commands.ts` | Notificações push KAIROS |
| `AGENT_TRIGGERS` | `ScheduleCronTool/prompt.ts` | Triggers de Agent (tarefas agendadas) |
| `AGENT_TRIGGERS_REMOTE` | — | Triggers de Agent remotos |
| `AGENT_MEMORY_SNAPSHOT` | — | Snapshots de memória do Agent |
| `TRANSCRIPT_CLASSIFIER` | `resetAutoModeOptInForDefaultOffer.ts`, `NotebookEditTool`, `ExitPlanModeV2Tool` | Classificador de diálogos |
| `VERIFICATION_AGENT` | `TaskUpdateTool` | Agent de verificação |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | `builtInAgents.ts` | Agents integrados de exploração/planejamento |
| `COORDINATOR_MODE` | `builtInAgents.ts` | Modo coordenador |
| `FORK_SUBAGENT` | `commands.ts` | Fork de subagente |
| `BUDDY` | `commands.ts` | Funcionalidade Buddy |
| `UDS_INBOX` | `SendMessageTool`, `commands.ts` | Caixa de entrada Unix Domain Socket |
| `BRIDGE_MODE` | `commands.ts` | Modo bridge (CCR) |
| `DAEMON` | `commands.ts` | Modo daemon |
| `VOICE_MODE` | `commands.ts` | Modo de voz |
| `ULTRAPLAN` | `commands.ts` | Comando UltraPlan |
| `ULTRATHINK` | — | Funcionalidade UltraThink |
| `TORCH` | `commands.ts` | Comando TORCH (carregamento dinâmico) |
| `MCP_SKILLS` | `commands.ts` | Suporte a skills MCP |
| `CHICAGO_MCP` | `metadata.ts` | Servidor MCP Chicago integrado (computer-use) |
| `WORKFLOW_SCRIPTS` | `commands.ts` | Scripts de workflow |
| `CCR_REMOTE_SETUP` | `commands.ts` | Comando de setup remoto CCR |
| `CCR_AUTO_CONNECT` | — | Conexão automática CCR |
| `CCR_MIRROR` | — | Modo mirror CCR |
| `PROACTIVE` | `commands.ts` | Modo proativo |
| `EXPERIMENTAL_SKILL_SEARCH` | `commands.ts` | Busca experimental de skills |
| `HISTORY_SNIP` | `commands.ts` | Funcionalidade de trecho de histórico |
| `HISTORY_PICKER` | — | Seletor de histórico |
| `WEB_BROWSER_TOOL` | — | Ferramenta de browser Web |
| `QUICK_SEARCH` | — | Busca rápida |
| `MONITOR_TOOL` | — | Ferramenta de monitoramento |
| `OVERFLOW_TEST_TOOL` | — | Ferramenta de teste de overflow |
| `BREAK_CACHE_COMMAND` | — | Comando de ponto de interrupção de cache |
| `TREE_SITTER_BASH` | — | Parsing Bash Tree-sitter |
| `TREE_SITTER_BASH_SHADOW` | — | Comparação shadow Tree-sitter |
| `BASH_CLASSIFIER` | — | Classificador de segurança Bash |
| `TERMINAL_PANEL` | — | Painel de terminal |
| `NATIVE_CLIPBOARD_IMAGE` | — | Suporte a imagem na área de transferência nativa |
| `NATIVE_CLIENT_ATTESTATION` | — | Attestation de cliente nativo |
| `AUTO_THEME` | — | Tema automático |
| `POWERSHELL_AUTO_MODE` | — | Modo automático PowerShell |
| `TOKEN_BUDGET` | — | Exibição de orçamento de tokens |
| `STREAMLINED_OUTPUT` | — | Modo de saída simplificada |
| `CONNECTOR_TEXT` | — | Texto de conector |
| `CONTEXT_COLLAPSE` | — | Colapso de contexto |
| `COMPACTION_REMINDERS` | — | Lembretes de compressão |
| `CACHED_MICROCOMPACT` | — | Micro-compressão com cache |
| `REACTIVE_COMPACT` | — | Compressão reativa |
| `PROMPT_CACHE_BREAK_DETECTION` | — | Detecção de ponto de interrupção de Prompt Cache |
| `EXTRACT_MEMORIES` | — | Extração automática de memória |
| `LODESTONE` | — | Funcionalidade Lodestone |
| `TEAMMEM` | — | Memória da equipe |
| `TEMPLATES` | — | Funcionalidade de templates |
| `FILE_PERSISTENCE` | — | Persistência de arquivos |
| `BG_SESSIONS` | — | Sessões em segundo plano |
| `DOWNLOAD_USER_SETTINGS` | — | Download de configurações do usuário |
| `UPLOAD_USER_SETTINGS` | — | Upload de configurações do usuário |
| `NEW_INIT` | — | Novo fluxo de inicialização |
| `HARD_FAIL` | — | Modo de falha rígida |
| `SLOW_OPERATION_LOGGING` | — | Log de operações lentas |
| `SHOT_STATS` | — | Estatísticas de requisições |
| `MEMORY_SHAPE_TELEMETRY` | — | Telemetria de forma da memória |
| `COWORKER_TYPE_TELEMETRY` | — | Telemetria de tipo de colaborador |
| `ANTI_DISTILLATION_CC` | — | Proteção anti-destilação |
| `RUN_SKILL_GENERATOR` | — | Gerador de skills |
| `SKILL_IMPROVEMENT` | — | Melhoria de skills |
| `REVIEW_ARTIFACT` | — | Artefato de revisão de código |
| `MESSAGE_ACTIONS` | — | Ações de mensagem |
| `AWAY_SUMMARY` | — | Resumo de ausência |
| `COMMIT_ATTRIBUTION` | — | Atribuição de commit |
| `UNATTENDED_RETRY` | — | Nova tentativa sem supervisão |
| `IS_LIBC_GLIBC` / `IS_LIBC_MUSL` | — | Detecção de tipo libc (injetado em tempo de build) |

### Feature Flags em Tempo de Execução vs Tempo de Compilação

| Dimensão | Tempo de compilação (`feature()`) | Tempo de execução (GrowthBook) |
|----------|----------------------------------|-------------------------------|
| Momento de execução | Fase de empacotamento | Carregamento assíncrono após início do processo |
| Retenção de código | Branches removidos não existem no produto final | Código existe, mas lógica controlada pelo valor do flag |
| Modo de atualização | Publicar nova versão | Push do servidor, efetivo em no mínimo 20 minutos |
| Uso típico | Funcionalidades ant-only, ferramentas experimentais, código de diferença de plataforma | Teste A/B, implantação gradual, kill-switch, configuração dinâmica |
| Modo de substituição | Variáveis de build | Variável de ambiente `CLAUDE_INTERNAL_FC_OVERRIDES` (apenas ant) |

### Mecanismo de Dead Code Elimination

A função `feature()` do `bun:bundle` é uma função especial integrada do empacotador Bun, que na fase de build substitui diretamente `feature('X')` por `true` ou `false` com base nas definições de build-time, depois elimina as branches constantemente falsas via constant folding e dead code elimination.

Exemplo (`perfettoTracing.ts:216-220`):
```typescript
// na build externa, este bloco if inteiro é completamente removido
if (feature('PERFETTO_TRACING')) {
  isEnabled = true
  // ... todo o código de inicialização Perfetto
}
```

Esse mecanismo não apenas reduz o tamanho do pacote, mas também evita que o código de ferramentas internas seja exposto nos produtos externos.

### Feature Flags de Execução Importantes Conhecidos

A seguir estão alguns flags GrowthBook `tengu_*` conhecidos e suas funcionalidades:

| Nome do Flag | Tipo | Descrição da Funcionalidade |
|-------------|------|-----------------------------|
| `tengu_auto_mode_config` | Object | Configuração do modo automático (enabled/opt-in) |
| `tengu_1p_event_batch_config` | Object | Configuração de exportação em lote de eventos 1P |
| `tengu_event_sampling_config` | Object | Dicionário de configuração de taxa de amostragem de eventos |
| `tengu_log_datadog_events` | Boolean | Interruptor de relatório de eventos Datadog |
| `tengu_frond_boric` | Object | Kill-switch de sink Analytics (desabilita por nome de sink) |
| `tengu_quartz_lantern` | Boolean | Controle de comportamento de escrita atômica do FileWriteTool |
| `tengu_hive_evidence` | Boolean | Controle de comportamento de atualização de tarefas/escrita de Todo |
| `tengu_plum_vx3` | Boolean | Interruptor para WebSearchTool usar modelo Haiku |
| `tengu_kairos_cron` | Object | Configuração de tarefas agendadas KAIROS |
| `tengu_kairos_cron_durable` | Boolean | Suporte a tarefas agendadas persistentes |
| `tengu_agent_list_attach` | Boolean | Comportamento de anexação de lista do AgentTool |
| `tengu_amber_stoat` | Boolean | Controle de disponibilidade de agentes integrados |
| `tengu_slim_subagent_claudemd` | Boolean | Carregamento simplificado de CLAUDE.md de subagente |
| `tengu_glacier_2xr` | Boolean | Controle de decisão de modo ToolSearch |
| `tengu_max_version_config` | Object | Limite de versão máxima (kill-switch de atualização forçada) |
| `tengu_prompt_cache_1h_config` | Object | Configuração de Prompt Cache de 1 hora |
| `tengu_sm_compact_config` | Object | Configuração de compressão do Session Memory |
| `tengu_ant_model_override` | String | Substituição de modelo exclusiva para ant |
| `enhanced_telemetry_beta` | Boolean | Interruptor beta de telemetria aprimorada |

---

## 14.4 Sistema de Observabilidade

### Integração OpenTelemetry

O Claude Code implementa completamente o suporte a três sinais OpenTelemetry; a entrada central está em `src/utils/telemetry/instrumentation.ts` (825 linhas).

**Inicialização bootstrap** (`instrumentation.ts:bootstrapTelemetry()`):
Em builds ant, lê configuração das variáveis com prefixo `ANT_OTEL_*` e mapeia para variáveis padrão `OTEL_*`. Para usuários externos, segue a especificação de variáveis de ambiente OTel padrão; a temporalidade padrão é definida como `delta` (incremental, não acumulativa).

**Configuração de exportadores dos três sinais** (design com lazy loading):

```typescript
// instrumentation.ts:169-190 (simplificado)
// Exportadores OTLP/Prometheus usam dynamic import com lazy loading
// Evita que @grpc/grpc-js (~700KB) seja carregado quando desnecessário
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

Os três protocolos de transporte são suportados: `grpc`, `http/json`, `http/protobuf`, selecionados via variável de ambiente `OTEL_EXPORTER_OTLP_PROTOCOL`.

**Atributos de recurso**: nome do serviço é `claude-code`; carrega atributos de arquitetura de plataforma, versão WSL, tipo de assinatura, versão do serviço, etc., preenchidos automaticamente via `envDetector`, `hostDetector`, `osDetector`.

### Transmissão de Dados gRPC

gRPC é o protocolo de transporte recomendado para cenários empresariais, fornecendo transmissão de streaming bidirecional e codificação protobuf fortemente tipada. No Claude Code:

- Exportador gRPC (`@opentelemetry/exporter-metrics-otlp-grpc`) como dependência lazy loaded, evitando impacto no tempo de inicialização
- Configuração mTLS suportada via `getMTLSConfig()`, certificados auto-assinados podem ser usados em cenários de rede interna empresarial
- Suporte a proxy tratado de forma transparente via `getProxyUrl()` + `HttpsProxyAgent`

Subprocessos não herdam variáveis de ambiente relacionadas ao OTEL (`subprocessEnv.ts`):
```typescript
// subprocessEnv.ts:24-28
// for monitoring backends; read in-process by OTEL SDK, subprocesses never need them
'OTEL_EXPORTER_OTLP_HEADERS',
'OTEL_EXPORTER_OTLP_LOGS_HEADERS',
'OTEL_EXPORTER_OTLP_METRICS_HEADERS',
'OTEL_EXPORTER_OTLP_TRACES_HEADERS',
```

### Rastreamento Perfetto

Perfetto é um framework de rastreamento de sistema de alto desempenho desenvolvido pelo Google. O Claude Code implementa uma camada de compatibilidade com o formato Chrome Trace Event (`src/utils/telemetry/perfettoTracing.ts`, 1120 linhas, ant-only).

**Modo de habilitação**:
```bash
CLAUDE_CODE_PERFETTO_TRACE=1          # grava em ~/.claude/traces/trace-<session-id>.json
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my.json  # grava no caminho especificado
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=60  # grava periodicamente a cada 60 segundos
```

**Tipos de Spans rastreados**:

| Nome do Span | Categoria | Informações carregadas |
|-------------|-----------|----------------------|
| `API Call` | `api` | model, prompt_tokens, ttft_ms, ttlt_ms, itps, otps, cache_hit_rate_pct |
| `Request Setup` | `api,setup` | request_setup_ms, attempt_count |
| `Attempt N (retry)` | `api,retry` | número do attempt |
| `First Token` | `api,ttft` | ttft_ms, itps |
| `Sampling` | `api,sampling` | sampling_ms, otps |
| `Tool: <name>` | `tool` | tool_name, success, result_tokens, duration_ms |
| `Waiting for User Input` | `user_input` | context, decision, source |
| `Interaction` | `interaction` | user_prompt_length, duration_ms |

**Gerenciamento de memória** (limite de 100.000 eventos):
```typescript
// perfettoTracing.ts:95-100
const MAX_EVENTS = 100_000
// ao atingir o limite, remove a metade mais antiga, amortizando custo O(1)
// insere marcador trace_truncated para tornar a lacuna visível em ui.perfetto.dev
```

**Rastreamento hierárquico multi-Agent**: cada Agent (incluindo subagentes) é mapeado para um process ID independente, com a relação hierárquica registrada via eventos de metadata `parent_agent`, aparecendo como swimlanes independentes na UI do Perfetto.

**Estratégia de escrita** (tripla garantia):
1. Callback assíncrono do `cleanup registry` (saída normal)
2. Handler `process.on('beforeExit')` (reserva)
3. Escrita síncrona `process.on('exit')` (última linha de defesa; neste ponto, não é possível usar async)

### Rastreamento de Sessão OpenTelemetry

`src/utils/telemetry/sessionTracing.ts` (927 linhas) é a entrada de telemetria aprimorada voltada para usuários externos, baseada em Spans OTel padrão em vez do formato Perfetto.

**Condições de habilitação** (`sessionTracing.ts:170-185`):
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

**Propagação de contexto AsyncLocalStorage**: cada Interaction e Tool Call usa armazenamento ALS independente para armazenar SpanContext, garantindo que Spans não se misturem em cenários de múltiplos Agents concorrentes. WeakRef previne vazamento de memória de spans de longa duração; limpeza de 60 segundos elimina Spans órfãos com mais de 30 minutos.

**Sistema de eventos logEvent**

Todos os eventos de negócio são despachados uniformemente pela função `logEvent()` em `src/services/analytics/index.ts`:

```typescript
// index.ts (simplificado)
export function logEvent(
  eventName: string,
  metadata: LogEventMetadata,  // permite apenas boolean | number | undefined
): void {
  if (sink === null) {
    eventQueue.push({ eventName, metadata, async: false })
    return
  }
  sink.logEvent(eventName, metadata)
}
```

Design chave: o tipo de metadata exclui intencionalmente `string`, forçando desenvolvedores a usar a conversão de tipo `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`, que é um tipo `never` incapaz de conter valores reais — apenas força os desenvolvedores a escrever uma anotação de tipo declarando que verificaram manualmente que a string não contém código ou caminhos.

---

## 14.5 Coleta de Analytics

### Arquitetura de Roteamento de Dois Caminhos

Todos os eventos são roteados via `sink.ts` para dois backends:

```
logEvent() → sink.logEventImpl()
    ├─→ Datadog (se o gate tengu_log_datadog_events estiver aberto)
    │     transmite apenas eventos dentro da whitelist DATADOG_ALLOWED_EVENTS
    │     remove chaves com prefixo _PROTO_* (campos marcados como PII)
    └─→ Logger First-Party 1P (BatchLogRecordProcessor OpenTelemetry)
          envia para /api/event_logging/batch
          preserva chaves _PROTO_* (roteadas para colunas protegidas BigQuery)
```

**Integração Datadog** (`datadog.ts`):
- Endpoint: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- Envio em lote: 100 entradas/lote, intervalo de atualização de 15 segundos
- Timeout de rede: 5 segundos
- Mecanismo de whitelist: aproximadamente 50 eventos centrais (Set `DATADOG_ALLOWED_EVENTS`)
- Condições de desabilitação: clouds de terceiros Bedrock/Vertex/Foundry, ambientes de teste, usuário escolheu no-telemetry

**Log de Eventos 1P (FirstPartyEventLoggingExporter)**:
- Usa interface padrão `LogRecordExporter` do OpenTelemetry
- Exportação em lote: padrão 200 entradas/lote, atraso de agendamento de 5 segundos
- Nova tentativa em caso de falha: backoff exponencial (base 500ms, máximo 30s, máximo 8 vezes)
- Fila de falhas persistida: eventos com falha gravados em `~/.claude/telemetry/1p_failed_events.<batch-uuid>.jsonl`, nova tentativa no próximo início
- Serialização proto: usa tipo protobuf gerado `ClaudeCodeInternalEvent`

### Rastreamento de Comportamento do Usuário

Aproximadamente 400+ nomes de eventos `tengu_*` cobrem o ciclo de vida completo de interação do usuário. Categorias de eventos centrais:

**Ciclo de vida da sessão**: `tengu_started`, `tengu_init`, `tengu_exit`, `tengu_cancel`

**Chamadas de API**: `tengu_api_query`, `tengu_api_success`, `tengu_api_error`, `tengu_api_retry`

**Uso de ferramentas**: `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_in_prompt_permanent`

**Requisições de permissão**: `tengu_internal_bash_tool_use_permission_request`, `tengu_tool_use_show_permission_request`, `tengu_tool_use_granted_by_classifier`

**Autenticação OAuth**: `tengu_oauth_flow_start`, `tengu_oauth_success`, `tengu_oauth_token_refresh_*` (rastreamento completo da máquina de estados de bloqueio)

**Servidores MCP**: `tengu_mcp_server_connection_succeeded`, `tengu_mcp_server_connection_failed`, `tengu_mcp_oauth_flow_*`

**Mecanismo de atualização**: `tengu_binary_download_attempt`, `tengu_native_update_complete`, `tengu_binary_download_failure`

### Coleta de Métricas de Performance

Os Spans de API Call em `sessionTracing.ts` calculam as seguintes métricas derivadas:

```typescript
// perfettoTracing.ts (endLLMRequestPerfettoSpan simplificado)
const itps = ttftMs > 0
  ? Math.round((promptTokens / (ttftMs / 1000)) * 100) / 100
  : undefined  // Input Tokens Per Second (taxa de processamento de tokens de entrada)

const otps = samplingMs > 0
  ? Math.round((outputTokens / (samplingMs / 1000)) * 100) / 100
  : undefined  // Output Tokens Per Second (taxa de amostragem)

const cacheHitRate = promptTokens > 0
  ? Math.round((cacheReadTokens / promptTokens) * 10000) / 100
  : undefined  // taxa de acerto de Cache (percentagem)
```

### Controle de Amostragem de Eventos

Controla a taxa de amostragem de cada evento via configuração dinâmica do GrowthBook `tengu_event_sampling_config`:

```typescript
// firstPartyEventLogger.ts (shouldSampleEvent simplificado)
// retorna null = amostragem 100% (sem configuração)
// retorna 0 = descartar completamente
// retorna rate (0-1) = amostragem aleatória, e escreve sample_rate no metadata
const config: EventSamplingConfig = {
  'tengu_api_success': { sample_rate: 0.1 },  // exemplo: amostragem de 10%
}
```

### Relatório de Erros

Sistema de eventos de erro em múltiplas camadas:
- `tengu_uncaught_exception`, `tengu_unhandled_rejection`: erros não capturados a nível de processo
- `tengu_api_error`, `tengu_query_error`: erros de chamada de API
- `tengu_streaming_error`: erros de resposta em streaming
- `tengu_atomic_write_error`: erros de escrita de arquivo
- `tengu_compact_failed`: falha na compressão de sessão

---

## 14.6 Diagnóstico e Depuração

### Comando /doctor

`src/commands/doctor/index.ts` registra o comando `/doctor`:

```typescript
const doctor: Command = {
  name: 'doctor',
  description: 'Diagnose and verify your Claude Code installation and settings',
  isEnabled: () => !isEnvTruthy(process.env.DISABLE_DOCTOR_COMMAND),
  type: 'local-jsx',
  load: () => import('./doctor.js'),
}
```

Este comando é executado como tipo `local-jsx` (renderiza componentes React diretamente dentro do REPL), verificando: integridade da instalação, status de conexão do servidor MCP, validade da configuração de keybindings, dependências de ambiente (ripgrep, etc.).

### Sistema de Rastreamento de Diagnóstico

Em cenários de integração com IDE, o Claude Code recebe informações de diagnóstico de código via Language Server Protocol. Quando um arquivo é salvo (evento `didSave`), o TypeScript Server envia novas mensagens de diagnóstico; o sistema as injeta como tags XML `<new-diagnostics>` para passar ao modelo:

```typescript
// messages.ts:3812-3821
case 'diagnostics': {
  content: `<new-diagnostics>The following new diagnostic issues were detected:\n\n${diagnosticSummary}</new-diagnostics>`,
}
```

### Diagnóstico de Memória Heap

`src/utils/heapDumpService.ts` fornece capacidade de diagnóstico de memória a nível de processo; ao acionar um heap dump, coleta sincronamente um snapshot de uso de memória, saindo para `~/Desktop/<session-id>-diagnostics.json` com `heapUsed`, `external`, `rss` e sugestões de análise. Evento analytics correspondente: `tengu_heap_dump`.

### Log de Recuperação de Erros

`src/utils/telemetry/bigqueryExporter.ts` implementa um exportador de métricas BigQuery, integrado com o pipeline OTel Metrics, usado para monitoramento de performance de longo prazo e planejamento de capacidade interno da ant. A fila de persistência `1p_failed_events` garante que eventos críticos não sejam perdidos mesmo em caso de falha de rede.

---

## 14.7 Análise das Decisões de Design

### Prós e Contras dos Flags em Tempo de Compilação

**Vantagens**:
1. **Zero overhead em tempo de execução**: branches removidos não existem no produto, sem qualquer custo de julgamento condicional
2. **Isolamento de segurança**: código de funcionalidades ant-only é completamente invisível para usuários externos, não pode ser engenharia reversa
3. **Otimização do tamanho do pacote**: módulos grandes (como `@grpc/grpc-js` ~700KB) só existem nas builds que precisam deles
4. **Segurança de tipos**: a verificação de tipos do TypeScript atua antes do empacotamento, sem afetar o runtime

**Desvantagens**:
1. **Dependência de publicação**: mudar o estado de um flag requer publicar uma nova versão; não pode ser hot-update
2. **Explosão da matriz de testes**: N flags em tempo de compilação teoricamente requerem 2^N combinações de build para testes
3. **Complexidade de depuração**: quando usuários externos relatam problemas, alguns caminhos de código simplesmente não existem em suas builds

### Equilíbrio entre Privacidade e Observabilidade

O Claude Code adota múltiplas camadas de defesa para proteção de privacidade:

1. **Proteção pelo sistema de tipos**: `LogEventMetadata` permite apenas `boolean | number | undefined`, proibindo strings diretas. Para registrar uma string, é necessário declarar explicitamente `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`; este é um tipo `never` que não pode conter valores reais — apenas força os desenvolvedores a escrever uma anotação de tipo, indicando que verificaram manualmente que a string não contém código ou caminhos.

2. **Desensibilização de nomes de ferramentas MCP**: o formato de nome de ferramenta MCP `mcp__<server>__<tool>` pode vazar configuração de serviço privado do usuário; por padrão é desensibilizado para `mcp_tool`. Apenas servidores dentro do registro oficial de MCP, servidores declarados explicitamente como integrados, ou nomes de servidores na entrada `cowork` são preservados como nomes originais.

3. **Campos marcados como PII**: chaves de metadata com prefixo `_PROTO_*` indicam campos sensíveis a PII, roteados apenas para colunas protegidas do BigQuery 1P; `sink.ts` remove esses campos antes de encaminhar ao Datadog.

4. **Desabilitação em clouds de terceiros**: para clientes empresariais usando Bedrock/Vertex/Foundry, todos os analytics do lado Anthropic (incluindo Datadog e 1P) são desabilitados por padrão.

### Razão para Lazy Loading de Telemetria

Pacotes relacionados a OTLP (gRPC ~700KB, proto ~300KB) usam `import()` dinâmico com lazy loading porque:

1. **Sensibilidade ao tempo de inicialização**: a métrica de performance primária de ferramentas CLI é Time-to-First-Output; qualquer inicialização desnecessária deve ser adiada
2. **Protocolos mutuamente exclusivos**: um processo usa apenas um protocolo de transporte; import estático de todas as variantes (6 pacotes) é puro desperdício
3. **Compatibilidade com otimizações Bun**: lazy loading está alinhado com o padrão de otimização de resolução de módulos do Bun; após análise estática, empacota sob demanda

---

## 14.8 Padrões Transferíveis

Os seguintes padrões de design têm alto valor de referência para outros projetos:

### 1. Sistema de Tipos Previne Vazamento de PII

Via tipo marker `never`, força desenvolvedores a confirmar explicitamente em tempo de compilação que não contém informações sensíveis. O custo é zero (sem overhead em runtime); a proteção é 100% eficaz (contornar requer asserção de tipo explícita). Aplicável a qualquer sistema com necessidades de upload de dados.

### 2. Arquitetura de Feature Flag de Dois Níveis

Tempo de compilação (separação de camadas de código) + tempo de execução (controle de comportamento) em trilha dupla, correspondendo a diferentes necessidades de velocidade de publicação:
- Funcionalidades estruturais (presença/ausência de módulos inteiros) → tempo de compilação
- Ajuste de comportamento (parâmetros, proporções, seleção de algoritmo) → tempo de execução

### 3. Padrão de Kill-Switch de Sink

A configuração GrowthBook `tengu_frond_boric` permite desabilitar independentemente qualquer backend analytics por nome (`datadog`, `firstParty`) sem publicar uma nova versão. Este é um padrão de circuit breaker de emergência genérico, adequado para todos os sistemas de eventos com múltiplos sinks downstream.

### 4. Persistência e Nova Tentativa de Eventos com Falha

Quando a exportação de eventos 1P falha, grava no arquivo JSONL local; nova tentativa no próximo início. Isso garante que dados críticos de telemetria não sejam perdidos em caso de falha de rede, especialmente adequado para ferramentas que rodam em ambientes offline ou de rede instável.

### 5. Deduplicação de Exposição de Experimentos

Eventos de exposição de experimentos do GrowthBook (usados para análise de resultados de teste A/B) são deduplicados via Set a nível de sessão, garantindo que a exposição de cada feature seja registrada apenas uma vez no lado analítico, prevenindo contagens infladas de exposição de múltiplas chamadas ao mesmo flag.

---

## 14.9 Índice de Código-Fonte

| Caminho do arquivo (relativo a `src/`) | Linhas | Responsabilidade Central |
|----------------------------------------|--------|--------------------------|
| `services/analytics/growthbook.ts` | 1155 | Integração SDK GrowthBook, leitura de Feature Flags, registro de exposição A/B |
| `services/analytics/index.ts` | 173 | API pública logEvent, fila de eventos, definição da interface Sink |
| `services/analytics/sink.ts` | 114 | Implementação de roteamento de dois caminhos (Datadog + 1P), inicialização |
| `services/analytics/datadog.ts` | 307 | Envio de log em lote Datadog, filtragem por whitelist |
| `services/analytics/firstPartyEventLogger.ts` | 449 | Inicialização do LoggerProvider OTel, controle de amostragem |
| `services/analytics/firstPartyEventLoggingExporter.ts` | 806 | Exportação HTTP de eventos 1P, nova tentativa persistida, serialização proto |
| `services/analytics/metadata.ts` | 973 | Enriquecimento de metadados de eventos, desensibilização de nomes de ferramentas MCP, tratamento de PII |
| `services/analytics/config.ts` | 38 | Lógica compartilhada isAnalyticsDisabled() |
| `services/analytics/sinkKillswitch.ts` | 25 | Kill-Switch a nível de Sink (tengu_frond_boric) |
| `utils/telemetry/instrumentation.ts` | 825 | Inicialização do SDK OTel, configuração dos três sinais (Metrics/Logs/Traces) |
| `utils/telemetry/sessionTracing.ts` | 927 | Gerenciamento de Spans OTel, propagação de contexto AsyncLocalStorage |
| `utils/telemetry/perfettoTracing.ts` | 1120 | Rastreamento no formato Chrome Trace Perfetto (ant-only) |
| `utils/telemetry/betaSessionTracing.ts` | 491 | Atributos estendidos de rastreamento beta |
| `utils/telemetry/bigqueryExporter.ts` | 252 | Exportador de métricas BigQuery |
| `utils/telemetry/pluginTelemetry.ts` | 289 | Encapsulamento de telemetria de plugin |
| `utils/telemetry/events.ts` | 75 | Definições de tipo de eventos OTel |
| `commands/doctor/index.ts` | 12 | Registro do comando /doctor |
| `commands.ts` | — | Ponto centralizado de chamadas `feature()` em tempo de compilação |
