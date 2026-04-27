# Capítulo 2: Query Engine — Núcleo de Interação com LLM

## 2.1 Visão Geral e Posicionamento

**Posicionamento em uma frase:** O Query Engine é o "coração" do Claude Code — ele possui o ciclo de vida completo do diálogo com o LLM, transformando a entrada do usuário em interações agentic de múltiplas rodadas com controle de permissões, execução de ferramentas, compressão de contexto e rastreamento de custos.

**O problema central que resolve:** Como, em um async generator pipeline, orquestrar de forma confiável o ciclo "resposta em streaming do LLM → chamada de ferramenta → injeção de resultado → nova requisição", ao mesmo tempo em que trata pelo menos 12 ramificações de exceção como estouro da janela de contexto, orçamento de tokens, falhas de API, interrupções do usuário e aprovações de permissão.

**Estatísticas de arquivos e volume de código:**

| Arquivo | Linhas | Responsabilidade |
|------|------|------|
| `QueryEngine.ts` | 1.295 | Gerenciamento de estado a nível de sessão, ponto de entrada SDK/headless |
| `query.ts` | 1.729 | Loop principal de consulta, orquestração de execução de ferramentas, recuperação em múltiplas camadas |
| `query/config.ts` | 46 | Snapshot de configuração imutável |
| `query/tokenBudget.ts` | 93 | Decisão de auto-continue com orçamento de tokens |
| `query/stopHooks.ts` | 473 | Hooks ao parar (portão de segurança, pós-processamento) |
| `query/deps.ts` | 40 | Interface de injeção de dependência (amigável a testes) |
| `services/api/claude.ts` | 3.419 | Chamadas de API, análise de streaming, retry, estratégia de cache |
| `cost-tracker.ts` | 323 | Rastreamento de custos e acumulação de uso |
| **Total** | **~7.418** | — |

Essas 7.400+ linhas de código constituem aproximadamente 1,4% do código do Claude Code, mas são o caminho mais crítico de todo o produto — cada interação entre o usuário e Claude deve passar por aqui.

---

## 2.2 Fundamentos Teóricos

### 2.2.1 Async Generator Pipeline (Pipeline de Corrotinas)

A arquitetura central do Query Engine é baseada em **ES2018 Async Generator** como primitivo de processamento em streaming. A assinatura da função `query()` revela esse design:

```typescript
// query.ts:162
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
>
```

Não é uma escolha de sintaxe acidental, mas uma aplicação precisa da **teoria de Coroutines**. Async generators possuem simultaneamente:
- **Avaliação lazy**: orientada pelo consumidor, sem computar antecipadamente todas as respostas de API
- **Comunicação bidirecional**: yield produz stream events, return produz o motivo de encerramento
- **Segurança de recursos**: o bloco `finally` garante a liberação de streams (`releaseStreamResources()` em `claude.ts`)

Callback tradicional ou Promise chain não conseguem satisfazer simultaneamente as necessidades de "saída em streaming para a UI" e "aguardar resultados de execução de ferramentas". Async generators suportam naturalmente essa semântica de "o produtor pausa aguardando o consumidor".

### 2.2.2 State Machine (Máquina de Estados Implícita)

O loop principal do `query.ts` não é chamada recursiva (embora comentários em código antigo ainda mencionem `query_recursive_call`), mas uma **máquina de estados explícita acionada por while(true) + continue**:

```typescript
// query.ts:218 (definição do tipo State)
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

A cada `continue`, todo o State é reconstruído (atualização imutável); o campo `transition` registra o motivo da transição. É uma **Mealy Machine** clássica — a saída depende da combinação do estado atual e do evento de entrada.

### 2.2.3 Backpressure e Gestão de Recursos

O backpressure é crucial no streaming de LLM. A abordagem do Claude Code:

1. **Limitação natural por for-await-of**: o consumo do stream em `claude.ts` usa `for await (const part of stream)`, e a velocidade de processamento do consumidor determina diretamente a taxa de envio do produtor
2. **Stream idle watchdog** (`claude.ts:2397`): se nenhum chunk for recebido em 90 segundos, o stream é abortado ativamente e faz fallback para não-streaming
3. **Garantia de ciclo de vida do Generator**: o bloco `finally` garante que `releaseStreamResources()` seja executado em todos os caminhos de saída (incluindo `.return()` e exceções)

### 2.2.4 Por que Essas Teorias São Especialmente Importantes em Cenários de LLM

A chamada de API HTTP tradicional usa um modelo de "requisição-resposta" com tratamento de erros simples. O loop agentic de LLM enfrenta desafios únicos:

- **Uma única chamada pode durar 10 minutos** (limite non-streaming)
- **A resposta pode acionar novos I/Os durante a transmissão** (chamadas de ferramentas)
- **A janela de contexto é um recurso escasso com estado**, exigindo equilíbrio entre "perda de informação por compressão" e "crash por estouro"
- **O custo acumula em tempo real**, podendo ser interrompido a qualquer momento

Essas restrições tornam o Event Loop + máquina de estados + Backpressure suportes teóricos indispensáveis.

---

## 2.3 Arquitetura e Estruturas de Dados

### 2.3.1 Interface Central da Classe QueryEngine

```typescript
// QueryEngine.ts:155-166
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()
```

**Ponto de design:** QueryEngine é por-conversa. Cada chamada de `submitMessage()` é um novo turn na mesma conversa; o estado é mantido entre turns.

### 2.3.2 Definições de Tipos Principais

**QueryEngineConfig** (`QueryEngine.ts:95-153`) — configuração imutável passada na construção:

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**QueryConfig** (`query/config.ts:18-31`) — snapshot imutável do ambiente por-query:

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

Observe a intenção de design dos comentários do código-fonte (`config.ts:9-12`): "Intentionally excludes feature() gates -- those are tree-shaking boundaries and must stay inline at the guarded blocks for dead-code elimination." Esta é a fronteira clara entre otimização de compilação bun:bundle e configuração em tempo de execução.

### 2.3.3 Diagrama de Dependências entre Módulos

```
                    +-----------------+
                    |   SDK / REPL    |
                    +--------+--------+
                             |
                             | submitMessage()
                             v
                    +--------+--------+
                    |  QueryEngine.ts |  <-- gestão de estado de sessão
                    |  (1.295 linhas) |  <-- mutableMessages, totalUsage
                    +--------+--------+
                             |
                             | query()
                             v
                    +--------+--------+
                    |    query.ts     |  <-- loop principal e máquina de estados
                    |  (1.729 linhas) |  <-- State, loop while(true)
                    +--------+--------+
                             |
              +--------------+------------------+
              |              |                  |
              v              v                  v
    +---------+---+  +-------+--------+  +------+-------+
    | query/      |  | query/         |  | query/       |
    | config.ts   |  | tokenBudget.ts |  | stopHooks.ts |
    | (46 linhas) |  | (93 linhas)    |  | (473 linhas) |
    +-------------+  +----------------+  +--------------+
              |              |
              v              v
    +---------+--------------+--------+
    |         query/deps.ts           |
    |   QueryDeps (interface DI)      |
    +---------+-----------------------+
              |
              |  callModel / autocompact / microcompact
              v
    +---------+-----------+     +----------+--------+
    | services/api/       |     | cost-tracker.ts   |
    | claude.ts           |     | (323 linhas)      |
    | (3.419 linhas)      |     | addToTotalSession |
    | queryModelWith      |     | Cost, formatCost  |
    | Streaming           |     +-------------------+
    +---------+-----------+
              |
              v
    +---------+-----------+
    | withRetry / stream  |
    | análise SSE         |
    | fallback            |
    | non-streaming       |
    +---------------------+
```

**O design de injeção de dependência em `deps.ts`** (`deps.ts:18-37`) merece destaque especial:

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

"Using `typeof fn` keeps signatures in sync with the real implementations automatically" — esta é a melhor prática de injeção de dependência em TypeScript: sem necessidade de escrever interfaces manualmente; as assinaturas rastreiam as implementações automaticamente.

---

## 2.4 Algoritmos Centrais e Fluxos

### 2.4.1 Fluxo Completo de Execução do Loop Principal de query()

```
                submitMessage() [QueryEngine.ts]
                        |
                processUserInput()
                        |
                    entrada de query()
                        |
                +=======+========+  <--- início do loop principal while(true)
                |                |
                v                |
        snip/microcompact/       |
        context-collapse/        |
        autocompact              |
                |                |
                v                |
        verificação de           |
        blocking-limit           |
                |                |
                v                |
        callModel() [streaming]  |
                |                |
        +-------+--------+      |
        | consumo de      |      |
        | eventos stream  |      |
        | coleta tool_use |      |
        | exec. streaming |      |
        +-------+--------+      |
                |                |
        needsFollowUp?           |
        /          \             |
      NÃO          SIM           |
       |              \          |
       v               v        |
  +---------+   runTools() ou   |
  | lógica  |   getRemainingR() |
  | de      |         |         |
  | retomada|         v         |
  +---------+  getAttachments() |
       |     pré-busca memória  |
       |     descoberta skill   |
       |              |         |
       v              v         |
  handleStopHooks()   |         |
       |         maxTurns?      |
       |              |         |
       v         state = next   |
  checkTokenBudget()  |         |
       |              +---------+
       v
  return Terminal { reason }
```

### 2.4.2 Loop de Chamadas de Ferramentas (Tool-call Loop)

A inovação central das chamadas de ferramentas é a **streaming tool execution** — começar a executar ferramentas enquanto a resposta em streaming do LLM *ainda está em andamento*:

```typescript
// query.ts:443 (inicialização do StreamingToolExecutor)
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

No loop de consumo em streaming (`query.ts:536`):

```typescript
if (
  streamingToolExecutor &&
  !toolUseContext.abortController.signal.aborted
) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
```

Ferramentas são enviadas para execução ao `content_block_stop`, não após o término de toda a resposta do assistente. Isso significa que, se o LLM gerar 3 blocos tool_use, os dois primeiros podem ter terminado de executar enquanto o terceiro ainda está sendo transmitido.

### 2.4.3 Implementação Concreta do Processamento em Streaming

O `queryModel()` em `claude.ts` implementa manualmente a análise do stream SSE, **contornando deliberadamente o BetaMessageStream do SDK Anthropic**:

```typescript
// claude.ts comentário (aprox. linha 2180)
// Use raw stream instead of BetaMessageStream to avoid O(n^2) partial JSON parsing
// BetaMessageStream calls partialParse() on every input_json_delta, which we don't need
// since we handle tool input accumulation ourselves
```

Modelo de acumulação de estado em streaming:

```
message_start → partialMessage = part.message, usage = initial
    |
content_block_start → contentBlocks[index] = { type, input: '' }
    |
content_block_delta → contentBlocks[index].input += delta.partial_json
    |               → contentBlocks[index].text += delta.text
    |               → contentBlocks[index].thinking += delta.thinking
    |
content_block_stop → yield AssistantMessage (por bloco!)
    |
message_delta → usage = updateUsage(usage, part.usage)
    |          → stopReason = part.delta.stop_reason
    |          → cost = calculateUSDCost(); addToTotalSessionCost()
    |
message_stop → (encerramento)
```

Design-chave: **cada content block produz independentemente um AssistantMessage via yield**. Isso significa que quando uma resposta do LLM contém texto + tool_use, a UI pode exibir o texto imediatamente após sua conclusão, sem esperar pelo JSON do tool_use terminar.

### 2.4.4 5 Camadas de Retry e Recuperação de Erros

A arquitetura de recuperação de erros do Claude Code é defesa em profundidade, com 5 camadas:

**1ª Camada: withRetry (nível de API)** — `withRetry()` em `claude.ts` trata erros retentáveis como 429 (rate limit), 529 (overload), 5xx, incluindo backoff exponencial e fallback de modelo.

**2ª Camada: Fallback Streaming → Non-streaming** — Quando a conexão streaming é interrompida (`claude.ts:2592`):

```typescript
// Faz fallback para modo non-streaming com retries
const result = yield* executeNonStreamingRequest(...)
```

Inclui stream idle watchdog (timeout de 90s sem dados) e fallback de criação de stream 404.

**3ª Camada: Recuperação de max_output_tokens** — 3 recuperações progressivas em `query.ts`:

```typescript
// query.ts (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3)
// 1ª etapa: escalar para 64k tokens (ESCALATED_MAX_TOKENS)
// 2ª-4ª etapas: injetar meta message pedindo "Resume directly — no apology, no recap"
const recoveryMessage = createUserMessage({
  content:
    `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
    `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,
})
```

**4ª Camada: Recuperação de Prompt-too-long** — Três níveis em cascata:
1. Drenagem de colapso de contexto (esvaziar colapsos pendentes)
2. Compactação reativa (compactação emergencial completa)
3. Expor o erro e sair (evitar loop infinito)

O código-fonte tem um guarda explícito contra loops infinitos (comentário em `query.ts`):

```
// Resetting to false here caused an infinite loop: compact → still too long → error →
// stop hook blocking → compact → … burning thousands of API calls.
```

**5ª Camada: Fallback de Modelo** — Em `query.ts:673-719`, quando `FallbackTriggeredError` é capturado, muda para um modelo de fallback e tenta novamente toda a requisição.

### 2.4.5 Contagem de Tokens e Gestão de Orçamento

O sistema de orçamento de tokens é dividido em dois mecanismos independentes:

**Mecanismo A: API task_budget** — orçamento de tokens percebido pelo servidor, rastreado além dos limites de compactação:

```typescript
// query.ts:270-280 (rastreamento de taskBudgetRemaining além de compactações)
if (params.taskBudget) {
  const preCompactContext =
    finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

**Mecanismo B: auto-continue de orçamento de tokens no lado do cliente** (`tokenBudget.ts`) — continua automaticamente quando a saída do turn não atingiu 90% do orçamento:

```typescript
// tokenBudget.ts:46-62
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // Subagentes não fazem auto-continue
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }
  // ...
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD
  // ...
}
```

Note a **detecção de retornos decrescentes**: para automaticamente quando há 3+ continuações consecutivas e cada incremento é < 500 tokens, prevenindo que o modelo desperdice orçamento em saídas de baixa eficiência.

### 2.4.6 Tratamento do Thinking Mode

Os "comentários mágicos" no código-fonte resumem as 3 regras de ferro do thinking mode (`query.ts:105-118`):

```
1. A message that contains a thinking or redacted_thinking block
   must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant
   trajectory (a single turn, or if that turn includes a tool_use block
   then also its subsequent tool_result and the following assistant message)
```

Lógica de construção dos parâmetros de thinking em `claude.ts` (aprox. linha 2242):

```typescript
if (
  !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING) &&
  modelSupportsAdaptiveThinking(options.model)
) {
  thinking = {
    type: 'adaptive',
  } satisfies BetaMessageStreamParams['thinking']
} else {
  let thinkingBudget = getMaxThinkingTokensForModel(options.model)
  // ...
  thinkingBudget = Math.min(maxOutputTokens - 1, thinkingBudget)
  thinking = {
    budget_tokens: thinkingBudget,
    type: 'enabled',
  }
}
```

Modelos que suportam adaptive thinking o usam preferencialmente (sem necessidade de orçamento predefinido); caso contrário, recuam para enabled + budget_tokens.

### 2.4.7 Gerenciamento de Limites do Prompt Cache

A estratégia de cache do Claude Code é impressionante. O design central está em `addCacheBreakpoints()` (`claude.ts:3045`):

```
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn... with one marker they're freed immediately.
```

**Apenas um marcador cache_control**, posicionado na última mensagem (ou na penúltima quando `skipCacheWrite` está ativo) — resultado de uma otimização colaborativa com o gerenciador de páginas KV cache (Mycro) da equipe de inferência.

O cache com TTL de 1h também tem um mecanismo refinado de "session stability latching" (`claude.ts:380-420`) — a elegibilidade, uma vez determinada, é fixada para a sessão inteira, prevenindo que mudanças de configuração do GrowthBook durante a sessão causem inversão do TTL do cache_control e invalidem o cache.

---

## 2.5 Análise das Decisões de Design

### 2.5.1 Tradeoffs Principais

**Tradeoff 1: Execução em streaming vs. garantia de completude**

O StreamingToolExecutor começa a executar ferramentas enquanto o LLM ainda está gerando, trazendo otimização significativa de latência, mas introduzindo complexidade — se o streaming falhar no meio, ferramentas já executadas precisam ser descartadas:

```typescript
// query.ts:534-538 (limpeza durante fallback de streaming)
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

Esse problema já causou um bug (veja o comentário `claude.ts:2575` referenciando inc-4258: execução dupla de ferramentas).

**Tradeoff 2: Estabilidade do cache vs. flexibilidade dinâmica**

Múltiplos beta headers usam o padrão "sticky-on latch" (`claude.ts:2102-2126`) — uma vez ativado, permanece ativo por toda a sessão, mesmo que a funcionalidade seja desativada:

```typescript
// claude.ts:2104 comentário
// Sticky-on latches for dynamic beta headers. Each header, once first
// sent, keeps being sent for the rest of the session so mid-session
// toggles don't change the server-side cache key and bust ~50-70K tokens.
```

É uma troca explícita de "taxa de acerto do cache tem prioridade sobre flexibilidade funcional".

**Tradeoff 3: Máquina de estados vs. recursão**

O loop principal evoluiu de chamadas recursivas `query()` para `while(true)` + reconstrução de State. Os checkpoints ainda têm o nome `query_recursive_call`, mas a implementação já é iterativa. Benefícios:
- Sem risco de stack overflow (conversas longas podem ter centenas de turns)
- A reconstrução do State é explícita, facilitando a depuração
- O campo `transition` fornece uma trilha de auditoria completa das transições de estado

### 2.5.2 Problemas Conhecidos Revelados pelos Comentários do Código-Fonte

1. **Duplicação de text delta no SDK** (`claude.ts:2350`):

   ```
   // awkwardly, the sdk sometimes returns text as part of a
   // content_block_start message, then returns the same text
   // again in a content_block_delta message
   ```

2. **Conflito entre fallback non-streaming e streaming tool execution** (`claude.ts:2575`):

   ```
   // The mid-stream fallback causes double tool execution when streaming
   // tool execution is active: the partial stream starts a tool, then the
   // non-streaming retry produces the same tool_use and runs it again. See inc-4258.
   ```

3. **Desvio na contagem de tokens do task budget além das compactações** (`query.ts:268`):

   ```
   // After a compact, the server sees only the summary and would under-count
   // spend; remaining tells it the pre-compact final window that got summarized away.
   ```

### 2.5.3 Diferenças de Design em Relação ao LangChain e Soluções Similares

| Dimensão | Claude Code Query Engine | LangChain AgentExecutor |
|------|--------------------------|-------------------------|
| Primitivo de streaming | ES Async Generator (nativo) | Callback + Stream wrapper |
| Gerenciamento de estado | State struct explícito + atualização imutável | Dict de AgentState mutável |
| Execução de ferramentas | Paralelismo em streaming (StreamingToolExecutor) | await sequencial |
| Retry | 5 camadas de defesa em profundidade + fallback de modelo | max_iterations simples |
| Injeção de dependência | QueryDeps + sincronização de assinatura via typeof | duck typing em tempo de execução |
| Cache | Cooperação profunda com cache KV de inferência | Nenhum (chamada de API como caixa preta) |

A diferença mais fundamental: o Claude Code é **inference-aware** — ele entende o mecanismo físico do Prompt Cache (gerenciador de páginas Mycro) e otimiza com base nisso, enquanto frameworks de código aberto só podem tratar a API como caixa preta.

---

## 2.6 Padrões Transferíveis

### 2.6.1 Padrões de Engenharia Gerais Extraídos do Query Engine

**Padrão 1: Immutable State + Transition Label**

```typescript
state = {
  ...oldState,
  messages: [...messages, ...newToolResults],
  transition: { reason: 'next_turn' },
}
continue
```

Registrar o **motivo** de cada transição de estado no próprio state torna depuração e telemetria preocupações de primeira classe. Qualquer sistema que exija tomada de decisão em múltiplas etapas pode adotar esse padrão.

**Padrão 2: Typed Dependency Injection via `typeof`**

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}
```

Sem necessidade de escrever interfaces manualmente; assinaturas sincronizam automaticamente com as implementações. Aplicável a qualquer sistema que precise fazer mock de I/O pesado.

**Padrão 3: Withholding Pattern (Atraso na Exposição de Erros)**

Para erros recuperáveis (prompt-too-long, max_output_tokens), não fazer yield para o consumidor imediatamente; decidir se expõe depois que a lógica de recuperação for executada:

```typescript
let withheld = false
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}
if (!withheld) {
  yield yieldMessage
}
```

Isso previne que consumidores do SDK recebam sinais de erro falsos quando "o erro já foi recuperado".

**Padrão 4: Session-stable Latching**

Para itens de configuração que afetam chaves de cache, uma vez ativados, bloquear para toda a sessão:

```typescript
let fastModeHeaderLatched = getFastModeHeaderLatched() === true
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true) // escreve no bootstrap state
}
```

Aplicável a qualquer cenário onde "mudança de configuração causaria invalidação de recursos caros".

### 2.6.2 O Que o Doramagic Pode Aprender

O pipeline `flow_controller` do Doramagic enfrenta problemas de orquestração similares ao Query Engine; pode aprender:

1. **Padrão State + Transition**: A máquina de 12 estados do Doramagic pode usar estruturas similares `{ state, transition: { reason } }`, simplificando a depuração e suportando auditoria de logs nativamente
2. **Dependency Injection via typeof**: A camada de chamadas de LLM do Doramagic pode usar o padrão QueryDeps, injetando modelos falsos nos testes sem precisar fazer mock de módulos inteiros
3. **Detecção de retornos decrescentes** (`tokenBudget.ts`): O Soul Extractor do Doramagic pode usar a mesma estratégia de "parar quando N incrementos consecutivos ficam abaixo do limite" durante refinamento iterativo, evitando que o LLM desperdice tokens em saídas de baixa qualidade

---

## 2.7 Índice do Código-Fonte

| Arquivo | Linhas | Responsabilidade em uma frase |
|------|------|----------|
| `QueryEngine.ts` | 1.295 | Dono a nível de sessão: mantém mutableMessages, totalUsage; traduz submitMessage() em chamadas query(); lida com roteamento de mensagens SDK |
| `query.ts` | 1.729 | Máquina de estados do loop principal: while(true) orquestrando compactação → chamada de API → execução de ferramentas → stop hooks → verificação de orçamento |
| `query/config.ts` | 46 | Snapshot imutável de QueryConfig: sessionId + 4 gates de runtime (feature() gates intencionalmente excluídos para preservar tree-shaking) |
| `query/tokenBudget.ts` | 93 | Auto-continue de orçamento de tokens no lado do cliente: limite de 90% de completude + parada antecipada por retornos decrescentes |
| `query/stopHooks.ts` | 473 | Orquestração de hooks ao final do turn: Stop hooks → TaskCompleted hooks → TeammateIdle hooks, suporte a reinjeção de erros de bloqueio |
| `query/deps.ts` | 40 | Interface de injeção de 4 dependências de I/O: callModel, microcompact, autocompact, uuid |
| `services/api/claude.ts` | 3.419 | Ciclo de vida completo da API: construção de parâmetros → criação de stream → análise SSE → acumulação de content block → cálculo de custo → fallback non-streaming → gerenciamento de breakpoints de cache |
| `cost-tracker.ts` | 323 | Acumulação de custo a nível de sessão: rastreamento de uso por modelo, persistência/retomada de sessão, saída formatada |
