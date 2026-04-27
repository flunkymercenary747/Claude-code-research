# Capítulo 8: Gerenciamento de Contexto

## 8.1 Visão Geral e Posicionamento

O gerenciamento de contexto é um dos subsistemas mais críticos da arquitetura do Claude Code. Uma sessão de programação típica pode durar horas, envolver centenas de chamadas de ferramentas e gerar centenas de milhares de tokens de histórico de conversa. Sem gerenciamento, a janela de contexto se esgota após 20-30 rodadas de interação, causando interrupção da sessão.

O problema central que o sistema de gerenciamento de contexto do Claude Code resolve é: **como manter a continuidade da sessão e a integridade das informações dentro de uma janela de contexto limitada (normalmente 200K tokens), minimizando ao mesmo tempo a perda de informações percebida pelo usuário?**

O sistema é composto por 11 arquivos no diretório `services/compact/`, totalizando cerca de 3.900 linhas de código TypeScript, auxiliados por dois módulos utilitários essenciais: `utils/collapseReadSearch.ts` (1.109 linhas) e `utils/toolResultStorage.ts` (1.040 linhas). O design de todo o subsistema reflete três princípios fundamentais:

1. **Degradação Gradual** (Graceful Degradation): da micro-compressão de custo zero à compressão total com perda, aumentando progressivamente a intensidade da intervenção
2. **Cache em Primeiro Lugar** (Cache-First): cada decisão de compressão prioriza a preservação do prompt cache
3. **Garantias de Segurança** (Safety Invariants): os pares tool_use/tool_result não podem ser cortados, proteção contra recursão, mecanismo de circuit breaker

## 8.2 Fundamentos Teóricos

### 8.2.1 Perspectiva da Teoria da Informação: Compressão com Perda vs. sem Perda

O gerenciamento de contexto é essencialmente um **problema de compressão de informação**. O sistema de múltiplas camadas do Claude Code corresponde a diferentes estratégias de compressão:

- **Compressão sem Perda** (Lossless): o caminho `cache_edits` da micro-compressão — exclui cópias de cache do lado do servidor de resultados antigos de ferramentas via o mecanismo de cache editing da API, sem alterar o conteúdo das mensagens locais. O modelo vê o placeholder `[Old tool result content cleared]`, mas os dados originais são preservados em disco (`toolResultStorage.ts`). A informação não é perdida, apenas movida do armazenamento quente para o frio.
- **Compressão com Perda** (Lossy): a compressão total usa um Fork Agent para gerar um resumo, comprimindo dezenas de milhares de tokens de conversa para alguns milhares. Trata-se de um processo de redução de dimensionalidade irreversível — detalhes de código, rastreamentos de erros e raciocínios intermediários podem ser perdidos.

Do ponto de vista da Rate-Distortion Theory, o design do Claude Code implica uma **função de medida de distorção**: as 9 seções do prompt de resumo (veja a seção 8.6) definem quais dimensões de informação são mais intolerantes à distorção — "user messages" (preservação completa) tem prioridade mais alta que "key technical concepts" (resumo permitido).

### 8.2.2 Teoria de Cache: Localidade Temporal e Espacial

O mecanismo de whitelist da micro-compressão reflete a suposição de **Localidade Temporal** (Temporal Locality) da teoria clássica de cache:

> Resultados de ferramentas usados recentemente têm maior probabilidade de serem referenciados em seguida.

A whitelist em `microCompact.ts` (`COMPACTABLE_TOOLS`) é uma política de eviction — apenas resultados de ferramentas específicas (Read, Shell, Grep, Glob, WebFetch, WebSearch, Edit, Write) podem ser removidos, pois suas saídas são regeneráveis (a ferramenta pode ser re-executada para obtê-las novamente). Texto digitado manualmente pelo usuário, imagens e outros conteúdos não regeneráveis nunca são removidos.

```typescript
// microCompact.ts:30-41 — whitelist de ferramentas compressíveis
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

O parâmetro `keepRecent` (padrão: manter os 5 mais recentes) implementa diretamente a política de eviction LRU (Least Recently Used).

### 8.2.3 Padrão Circuit Breaker

O mecanismo de circuit breaker em `autoCompact.ts` é uma adaptação precisa do clássico Circuit Breaker Pattern de sistemas distribuídos para aplicações LLM. Esse padrão originou-se do livro *Release It!* de Michael Nygard; seu modelo de três estados (Closed → Open → Half-Open) é implementado no Claude Code da seguinte forma:

```typescript
// autoCompact.ts:70-73 — limiar do circuit breaker
// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

Esse comentário revela dados reais de desastre antes da existência do circuit breaker: **1.279 sessões entraram em loops de falhas consecutivas com 50 ou mais ocorrências**, com o pior caso atingindo 3.272 tentativas de falha em uma única sessão, desperdiçando cerca de 250K chamadas de API por dia globalmente. A introdução do circuit breaker limitou o número máximo de tentativas a 3.

| Estado | Comportamento | Código correspondente |
|--------|--------------|----------------------|
| Closed (normal) | `consecutiveFailures < 3`, compressão tentada normalmente | Caminho padrão de `autoCompactIfNeeded` |
| Open (disparado) | `consecutiveFailures >= 3`, compressão ignorada | `tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| Half-Open (sondagem) | Após compressão bem-sucedida, `consecutiveFailures` reinicia para 0 | `consecutiveFailures: 0` no sucesso |

## 8.3 Visão Geral da Arquitetura

### 8.3.1 Arquitetura Geral do Sistema de Compressão em Múltiplas Camadas

O gerenciamento de contexto do Claude Code adota um design de **5 camadas de defesa**, ordenadas da menor para a maior intervenção:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Requisição do Usuário                     │
├─────────────────────────────────────────────────────────────────┤
│ Layer 0: Tool Result Storage (Camada Preventiva)                 │
│   Resultados grandes → persistência em disco + preview de 2KB   │
│   Gatilho: resultado > limiar (padrão 50K chars)                │
│   Custo: zero de contexto (armazenado em disco, preview no ctx) │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Microcompact (Micro-compressão)                         │
│   Caminho A: gatilho temporal — limpa conteúdo de results antigos│
│   Caminho B: cache editing — API cache_edits remove cache server │
│   Gatilho: antes de cada chamada de API                         │
│   Custo: mínimo (results substituídos por placeholder, recuper.) │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Auto-Compact (Compressão Automática)                    │
│   Session Memory → Fork Agent → resumo completo                 │
│   Gatilho: tokens > effectiveContextWindow - 13K                │
│   Custo: alto (resumo com perda, perde detalhes, 1 chamada API) │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Manual /compact (Compressão Manual)                     │
│   Ativado pelo usuário, suporta Partial Compact                 │
│   Gatilho: comando do usuário                                   │
│   Custo: igual ao acima                                         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Reactive Compact (Compressão Reativa)                   │
│   API retorna prompt_too_long → trunca e tenta novamente        │
│   Gatilho: erro 413                                             │
│   Custo: máximo (truncamento de emergência + resumo)            │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3.2 Comparação de Gatilhos, Custos e Perda de Informação por Camada

| Camada | Gatilho | Momento | Latência | Perda de Informação | Custo de API |
|--------|---------|---------|----------|--------------------|--------------| 
| L0: Tool Result Storage | Resultado único > limiar | Após execução da ferramenta | I/O de disco | Zero (original em disco) | Zero |
| L1a: Time-based MC | > 60min desde último assistant | Antes da chamada API | Zero (operação local) | Baixa (results antigos removidos) | Zero |
| L1b: Cached MC | Ferramentas compressíveis acima do limiar | Antes da chamada API | Zero (cache_edits) | Baixa (igual ao acima) | Zero |
| L2: Auto-Compact | tokens > threshold | Entre rodadas | 5-15s (chamada API) | Alta (resumo com perda) | 1 chamada API |
| L3: Manual Compact | Usuário /compact | Acionado pelo usuário | Igual ao acima | Média-alta (usuário pode orientar) | 1 chamada API |
| L4: Reactive Compact | prompt_too_long 413 | Após falha de API | 10-30s (retry) | Máxima (truncamento + resumo) | 1-4 chamadas API |

### 8.3.3 Fluxo de Dados

```
Array de mensagens (Message[])
    │
    ▼
microcompactMessages()  ──→ [gatilho temporal?] ──S──→ limpeza de conteúdo → retorna
    │ N                      │
    │                  [cache editing?] ──S──→ pendingCacheEdits → retorna
    │ N                      │
    ▼                        ▼
shouldAutoCompact()     sem compressão, retorna diretamente
    │ S
    ▼
trySessionMemoryCompaction() ──→ [tem session memory?]
    │ N                              │ S
    ▼                                ▼
compactConversation()           calculateMessagesToKeepIndex()
    │                                │
    ▼                                ▼
streamCompactSummary()          buildPostCompactMessages()
    │ (Fork Agent)
    ▼
formatCompactSummary()
    │
    ▼
buildPostCompactMessages()
    │
    ▼
Novo array: [boundary, summary, preserved, attachments, hooks]
```

## 8.4 Camada 1: Micro-compressão (Microcompact)

A micro-compressão é a primeira linha de defesa do gerenciamento de contexto. Ela é executada **antes de cada chamada de API** (ponto de entrada `microcompactMessages`), com o objetivo de liberar espaço de contexto ao mínimo custo.

### 8.4.1 Whitelist de Ferramentas Compressíveis

A micro-compressão opera apenas nas saídas de ferramentas específicas. O princípio de design por trás da whitelist é: **remover apenas conteúdo regenerável**.

```typescript
// microCompact.ts:30-41
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // leitura de arquivo — pode ser relido
  ...SHELL_TOOL_NAMES,     // comandos Shell — podem ser re-executados
  GREP_TOOL_NAME,          // busca — pode ser repetida
  GLOB_TOOL_NAME,          // correspondência de arquivo — pode ser repetida
  WEB_SEARCH_TOOL_NAME,    // busca web — pode ser repetida
  WEB_FETCH_TOOL_NAME,     // fetch web — pode ser repetido
  FILE_EDIT_TOOL_NAME,     // edição de arquivo — resultado salvo em disco
  FILE_WRITE_TOOL_NAME,    // escrita de arquivo — igual ao acima
])
```

Observe que `apiMicrocompact.ts` também define uma distinção mais granular:

```typescript
// apiMicrocompact.ts:23-33
const TOOLS_CLEARABLE_RESULTS = [   // limpa conteúdo de tool_result
  ...SHELL_TOOL_NAMES, GLOB_TOOL_NAME, GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME, WEB_FETCH_TOOL_NAME, WEB_SEARCH_TOOL_NAME,
]
const TOOLS_CLEARABLE_USES = [       // limpa input de tool_use
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME,
]
```

A distinção aqui é sutil: para Read/Grep/Shell, o que é removido é a **saída** (tool_result); para Edit/Write, o que é removido é a **entrada** (tool_use input), pois a entrada de operações de edição (conteúdo de diff) é volumosa, mas o resultado já foi persistido em disco.

### 8.4.2 Detalhamento dos Dois Sub-caminhos

A micro-compressão possui dois caminhos de execução mutuamente exclusivos, orquestrados pela função `microcompactMessages()`:

```typescript
// microCompact.ts:287-317 — lógica de orquestração
export async function microcompactMessages(
  messages: Message[],
  toolUseContext?: ToolUseContext,
  querySource?: QuerySource,
): Promise<MicrocompactResult> {
  clearCompactWarningSuppression()

  // Caminho A: gatilho temporal — maior prioridade, encurta os demais
  const timeBasedResult = maybeTimeBasedMicrocompact(messages, querySource)
  if (timeBasedResult) {
    return timeBasedResult
  }

  // Caminho B: cache editing — apenas thread principal, apenas modelos suportados
  if (feature('CACHED_MICROCOMPACT')) {
    const mod = await getCachedMCModule()
    const model = toolUseContext?.options.mainLoopModel ?? getMainLoopModel()
    if (
      mod.isCachedMicrocompactEnabled() &&
      mod.isModelSupportedForCacheEditing(model) &&
      isMainThreadSource(querySource)
    ) {
      return await cachedMicrocompactPath(messages, querySource)
    }
  }

  return { messages }
}
```

**Caminho A: Time-based Microcompact (Gatilho Temporal)**

Acionado quando o usuário retorna após deixar a sessão por mais tempo que o limiar configurado (padrão: 60 minutos). O motivo do design está claramente articulado em `timeBasedMCConfig.ts`:

```typescript
// timeBasedMCConfig.ts:14-17
// 60 is the safe choice: the server's 1h cache TTL is guaranteed expired
// for all users, so we never force a miss that wouldn't have happened.
```

O TTL do prompt cache do lado do servidor é de 1 hora. A ausência do usuário por mais de 1 hora significa que o **cache inevitavelmente expirou** e todo o prefix do prompt precisa ser reescrito. Limpar resultados antigos de ferramentas neste momento é "gratuito" — não gera custo adicional de invalidação de cache.

Lógica-chave do gatilho temporal:

```typescript
// microCompact.ts:381-389 — avaliação do gatilho temporal
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) {
    return null
  }
  const gapMinutes =
    (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  return { gapMinutes, config }
}
```

A estratégia de limpeza após o gatilho temporal também usa LRU (`keepRecent` padrão 5), mas com uma proteção de limite:

```typescript
// microCompact.ts:414-416
// Floor at 1: slice(-0) returns the full array (paradoxically keeps
// everything), and clearing ALL results leaves the model with zero working
// context.
const keepRecent = Math.max(1, config.keepRecent)
```

Esse `Math.max(1, ...)` previne a armadilha do JavaScript onde `slice(-0)` retorna o array completo quando `keepRecent=0` — um caso clássico de "programação defensiva para evitar ambiguidade semântica".

Após o gatilho temporal, o estado de cache editing também precisa ser reiniciado:

```typescript
// microCompact.ts:459-464
// Cached-MC state (module-level) holds tool IDs registered on prior turns.
// We just content-cleared some of those tools AND invalidated the server
// cache by changing prompt content. If cached-MC runs next turn with the
// stale state, it would try to cache_edit tools whose server-side entries
// no longer exist. Reset it.
resetMicrocompactState()
```

**Caminho B: Cached Microcompact (Cache Editing)**

Este é um caminho de otimização avançado interno da Anthropic (`feature('CACHED_MICROCOMPACT')`), que utiliza o mecanismo `cache_edits` da API para excluir resultados de ferramentas do cache do lado do servidor **sem modificar o conteúdo das mensagens locais**.

```typescript
// microCompact.ts:327-370 — núcleo do caminho de cache editing
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined,
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const state = ensureCachedMCState()
  const config = mod.getCachedMCConfig()

  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  // registra resultados de ferramentas
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && 
            compactableToolIds.has(block.tool_use_id) &&
            !state.registeredTools.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  const toolsToDelete = mod.getToolResultsToDelete(state)
  if (toolsToDelete.length > 0) {
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    if (cacheEdits) {
      pendingCacheEdits = cacheEdits
    }
    // ...
    return {
      messages,  // mensagens inalteradas!
      compactionInfo: {
        pendingCacheEdits: { trigger: 'auto', deletedToolIds: toolsToDelete, ... },
      },
    }
  }
  return { messages }
}
```

Decisão de design fundamental: **o array de mensagens não é alterado** — `return { messages }` retorna a referência original. O cache editing ocorre na camada de API (via parâmetro `cache_edits`), e o estado local permanece intacto. Isso significa que, se a chamada de API falhar ou for repetida, não há efeitos colaterais locais.

### 8.4.3 Gerenciamento de Estado do Cache Editing

O caminho de cache editing mantém três grupos de estado-chave:

```typescript
// microCompact.ts:43-49 — estado no nível do módulo
let cachedMCModule: typeof import('./cachedMicrocompact.js') | null = null
let cachedMCState: import('./cachedMicrocompact.js').CachedMCState | null = null
let pendingCacheEdits: import('./cachedMicrocompact.js').CacheEditsBlock | null = null
```

O gerenciamento do ciclo de vida desses três estados é sutil:

- `pendingCacheEdits` é descartável após uso — `consumePendingCacheEdits()` o lê e limpa (`microCompact.ts:80-84`); o chamador deve fixá-lo (pin) após enviá-lo na requisição de API.
- `pinnedCacheEdits` é acumulativo — cada cache edit bem-sucedido é fixado em uma posição específica de user message; requisições subsequentes devem enviá-lo novamente na mesma posição para garantir cache hit.
- `cachedMCState` é reiniciado após compressão (`resetMicrocompactState()`) ou após gatilho temporal.

```typescript
// microCompact.ts:78-105 — consumo de estado e pin
export function consumePendingCacheEdits() {
  const edits = pendingCacheEdits
  pendingCacheEdits = null
  return edits
}

export function getPinnedCacheEdits() {
  if (!cachedMCState) return []
  return cachedMCState.pinnedEdits
}

export function pinCacheEdits(
  userMessageIndex: number,
  block: import('./cachedMicrocompact.js').CacheEditsBlock,
): void {
  if (cachedMCState) {
    cachedMCState.pinnedEdits.push({ userMessageIndex, block })
  }
}
```

### 8.4.4 Funções Auxiliares de Estimativa de Token

O módulo de micro-compressão fornece a função de estimativa de token compartilhada por todo o sistema:

```typescript
// microCompact.ts:155-194 — estimateMessageTokens
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue
    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        totalTokens += calculateToolResultTokens(block)
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // fixo: 2000
      } else if (block.type === 'thinking') {
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'tool_use') {
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {}),
        )
      }
      // ...
    }
  }
  return Math.ceil(totalTokens * (4 / 3))  // fator de padding conservador 4/3
}
```

A fórmula central de `roughTokenCountEstimation` é extremamente simples: `Math.round(content.length / 4)` (`tokenEstimation.ts:203-207`). `estimateMessageTokens` então multiplica esse resultado por um fator conservador de 4/3, equivalendo a `text.length / 3`. Essa estratégia duplamente conservadora garante que a probabilidade de subestimação seja extremamente baixa.

## 8.5 Camada 2: Compressão Automática (Auto-Compact)

### 8.5.1 Fórmula de Cálculo do Limiar

O limiar de acionamento da compressão automática é calculado pela seguinte fórmula:

```
autoCompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

effectiveContextWindow = min(contextWindow, env.CLAUDE_CODE_AUTO_COMPACT_WINDOW)
                       - min(maxOutputTokens, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
```

Derivação com valores concretos (exemplo: Claude Opus 200K):

```
contextWindow = 200,000
maxOutputTokens = 16,384 (ou valor específico do modelo)
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20,000 (baseado em p99.99 = 17,387)
reservedTokensForSummary = min(16384, 20000) = 16,384
effectiveContextWindow = 200,000 - 16,384 = 183,616
autoCompactThreshold = 183,616 - 13,000 = 170,616
```

```typescript
// autoCompact.ts:40-43 — constantes-chave
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

A escolha de `AUTOCOMPACT_BUFFER_TOKENS = 13.000` é um compromisso de engenharia: muito pequeno e a compressão ocorre com demasiada frequência (cada compressão consome 5-15 segundos e uma chamada de API); muito grande e o contexto disponível é desperdiçado. 13K equivale aproximadamente a 3-5 rodadas de conversa comum.

### 8.5.2 Árvore de Decisão de shouldAutoCompact

```typescript
// autoCompact.ts:127-178 — cadeia de decisão completa
export async function shouldAutoCompact(
  messages, model, querySource, snipTokensFreed = 0
): Promise<boolean> {
  // 1. Proteção contra recursão: session_memory e compact não acionam
  if (querySource === 'session_memory' || querySource === 'compact') return false

  // 2. Proteção de context collapse: marble_origami (ctx-agent) não aciona
  if (feature('CONTEXT_COLLAPSE')) {
    if (querySource === 'marble_origami') return false
  }

  // 3. Verificação de configuração: usuário habilitou?
  if (!isAutoCompactEnabled()) return false

  // 4. Modo reativo: se habilitado, suprime compressão proativa
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) return false
  }

  // 5. Modo context collapse: collapse É gerenciamento de contexto, compressão não deve interferir
  if (feature('CONTEXT_COLLAPSE')) {
    if (isContextCollapseEnabled()) return false
  }

  // 6. Contagem de tokens + comparação com limiar
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

Essa árvore de decisão expõe múltiplas estratégias de gerenciamento de contexto que o Claude Code está experimentando em paralelo:
- **Reactive Compact** (`tengu_cobalt_raccoon`): não comprime proativamente, aguarda a API reportar prompt_too_long
- **Context Collapse** (`CONTEXT_COLLAPSE`): gerencia contexto de forma contínua com limites de 90% para envio e 95% para bloqueio
- **Auto Compact** (padrão atual): comprime proativamente ao atingir o limiar

As três são mutuamente exclusivas, controladas por feature flags.

### 8.5.3 Mecanismo de Circuit Breaker

```typescript
// autoCompact.ts:219-272 — autoCompactIfNeeded com circuit breaker
export async function autoCompactIfNeeded(
  messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed
) {
  if (isEnvTruthy(process.env.DISABLE_COMPACT)) return { wasCompacted: false }

  // verificação do circuit breaker
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }  // estado disparado, pula diretamente
  }

  if (!await shouldAutoCompact(messages, model, querySource, snipTokensFreed)) {
    return { wasCompacted: false }
  }

  // tenta compressão via Session Memory primeiro
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages, toolUseContext.agentId, recompactionInfo.autoCompactThreshold)
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    // ...
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // compressão tradicional
  try {
    const compactionResult = await compactConversation(...)
    runPostCompactCleanup(querySource)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const nextFailures = (tracking?.consecutiveFailures ?? 0) + 1
    if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
      logForDebugging(
        `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`,
        { level: 'warn' })
    }
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

### 8.5.4 Fluxo de Execução de autoCompactIfNeeded

Ordem completa de execução:

1. **Verificação de variável de ambiente**: `DISABLE_COMPACT` → desabilita globalmente
2. **Verificação do circuit breaker**: `consecutiveFailures >= 3` → pula
3. **Verificação de limiar**: `shouldAutoCompact()` → múltiplas verificações em camadas
4. **Compressão via Session Memory** (caminho preferido): usa session memory existente em vez de chamada de API
5. **Compressão tradicional via Fork Agent** (fallback): geração completa de resumo orientada por API
6. **Tratamento de falhas**: incrementa contador do circuit breaker, passa para a próxima rodada

## 8.6 Camada 3: Compressão Tradicional (Full Compact)

### 8.6.1 Mecanismo Fork Agent

O núcleo da compressão tradicional é a geração de um resumo de conversa via Fork Agent. A função `streamCompactSummary()` (`compact.ts:1136-1396`) implementa uma estratégia de fallback em dois níveis:

**Primeiro nível: Prompt Cache Sharing Fork**

```typescript
// compact.ts:1179-1247 — fork com compartilhamento de cache
if (promptCacheSharingEnabled) {
  try {
    const result = await runForkedAgent({
      promptMessages: [summaryRequest],
      cacheSafeParams,
      canUseTool: createCompactCanUseTool(),
      querySource: 'compact',
      forkLabel: 'compact',
      maxTurns: 1,
      skipCacheWrite: true,
      overrides: { abortController: context.abortController },
    })
    // ...
  }
}
```

O Fork Agent reutiliza o prompt cache completo da conversa principal (system prompt + tools + context messages), acrescentando apenas uma requisição de resumo. Decisões de design fundamentais:

1. `maxTurns: 1` — não permite interação em múltiplas rodadas
2. `canUseTool: createCompactCanUseTool()` — rejeita todas as chamadas de ferramentas
3. `skipCacheWrite: true` — não escreve no cache (fork temporário)
4. **maxOutputTokens não definido** — o comentário explica: defini-lo altera a thinking config, causando incompatibilidade de cache key

```typescript
// compact.ts:1181-1186
// DO NOT set maxOutputTokens here. The fork piggybacks on the main thread's
// prompt cache by sending identical cache-key params (system, tools, model,
// messages prefix, thinking config). Setting maxOutputTokens would clamp
// budget_tokens via Math.min(budget, maxOutputTokens-1) in claude.ts,
// creating a thinking config mismatch that invalidates the cache.
```

**Segundo nível: Streaming Fallback**

Quando o Fork Agent falha, o sistema recorre a uma chamada direta de API em modo streaming, onde **é possível** definir `maxOutputTokensOverride`:

```typescript
// compact.ts:1317-1320
maxOutputTokensOverride: Math.min(
  COMPACT_MAX_OUTPUT_TOKENS,
  getMaxOutputTokensForModel(context.options.mainLoopModel),
),
```

O fallback de streaming também suporta retry configurado (`tengu_compact_streaming_retry`), com no máximo `MAX_COMPACT_STREAMING_RETRIES = 2` tentativas.

### 8.6.2 Pipeline de Pré-processamento

As mensagens passam por três etapas de pré-processamento antes da compressão:

```typescript
// compact.ts:1293-1300 — cadeia de pré-processamento
normalizeMessagesForAPI(
  stripImagesFromMessages(
    stripReinjectedAttachments([
      ...getMessagesAfterCompactBoundary(messages),
      summaryRequest,
    ]),
  ),
  context.options.tools,
)
```

1. `getMessagesAfterCompactBoundary` — pega apenas as mensagens após a última compressão
2. `stripReinjectedAttachments` — remove anexos `skill_discovery` / `skill_listing` (que serão reinjetados após a compressão)
3. `stripImagesFromMessages` — substitui blocos de imagem pelo marcador de texto `[image]` (`compact.ts:144-199`)

A razão para `stripImagesFromMessages` é prática:

```typescript
// compact.ts:133-138
// Images are not needed for generating a conversation summary and can
// cause the compaction API call itself to hit the prompt-too-long limit,
// especially in CCD sessions where users frequently attach images.
```

Usuários CCD (Claude Code Desktop) frequentemente anexam capturas de tela; sem remover as imagens, a própria chamada de API de compressão pode falhar por prompt muito longo.

### 8.6.3 Formato de 9 Seções da Saída de Resumo

`prompt.ts` define as 9 seções estruturadas que o resumo deve seguir:

```
1. Primary Request and Intent    — intenção do usuário
2. Key Technical Concepts        — conceitos técnicos
3. Files and Code Sections       — arquivos e trechos de código
4. Errors and fixes              — erros e correções
5. Problem Solving               — resolução de problemas
6. All user messages             — todas as mensagens do usuário (não tool results)
7. Pending Tasks                 — tarefas pendentes
8. Current Work                  — trabalho atual
9. Optional Next Step            — próximo passo (opcional)
```

A seção 6 é particularmente importante — "List ALL user messages that are not tool results". Isso garante que, mesmo após a compressão da conversa, as expressões originais do usuário sejam preservadas integralmente. Esta é a garantia de **perda zero de informação do feedback do usuário**.

A seção 9 possui uma restrição cuidadosamente elaborada:

```
// prompt.ts — restrição da seção 9
IMPORTANT: ensure that this step is DIRECTLY in line with the user's most
recent explicit requests, and the task you were working on immediately before
this summary request. If your last task was concluded, then only list next
steps if they are explicitly in line with the users request. Do not start on
tangential requests or really old requests that were already completed without
confirming with the user first.
```

Isso impede que o modelo "aja por conta própria" após a compressão — somente os próximos passos explicitamente solicitados pelo usuário são registrados.

### 8.6.4 Design Anti-bypass de NO_TOOLS_PREAMBLE

O Fork Agent herda o conjunto completo de ferramentas da conversa principal (para compatibilidade de cache key), mas o agente de compressão não deve usar nenhuma ferramenta. Isso cria uma contradição: as ferramentas existem no schema, mas não devem ser chamadas.

A solução é uma **rejeição de ferramentas em três camadas**:

```typescript
// prompt.ts:16-24 — primeira camada: declaração forte no início do prompt
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

// prompt.ts:260-263 — segunda camada: lembrete repetido no final do prompt
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'

// compact.ts:1125-1133 — terceira camada: rejeição em nível de código
export function createCompactCanUseTool(): CanUseToolFn {
  return async () => ({
    behavior: 'deny' as const,
    message: 'Tool use is not allowed during compaction',
  })
}
```

Os comentários revelam a razão real para essas três camadas:

```typescript
// prompt.ts:12-17
// on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a 
// tool call despite the weaker trailer instruction. With maxTurns: 1, a 
// denied tool call means no text output → falls through to the streaming 
// fallback (2.79% on 4.6 vs 0.01% on 4.5).
```

No Sonnet 4.6, apenas instruções no prompt têm 2,79% de probabilidade de ainda tentar uma chamada de ferramenta (no 4.5 é apenas 0,01%). `createCompactCanUseTool` é a última salvaguarda em nível de código.

### 8.6.5 Pós-processamento (formatCompactSummary)

```typescript
// prompt.ts:270-288 — formatCompactSummary
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary
  // remove rascunho <analysis> — raciocínio intermediário que melhora qualidade, não precisa ser mantido
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, '')
  // extrai conteúdo de <summary>
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${summaryMatch[1].trim()}`)
  }
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')
  return formattedSummary.trim()
}
```

O design da tag `<analysis>` é um truque de Chain-of-Thought: o modelo primeiro "faz um rascunho" na área de análise, depois produz o resultado final em `<summary>`. A existência da área de análise melhora a qualidade do resumo, mas é removida da saída final — pois contém raciocínio intermediário redundante que desperdiçaria espaço de contexto em rodadas subsequentes.

### 8.6.6 Sequência de Mensagens Pós-compressão e Reinjeção de Anexos

Após a compressão, a nova sequência de mensagens é construída por `buildPostCompactMessages()`:

```typescript
// compact.ts:329-337
export function buildPostCompactMessages(result: CompactionResult): Message[] {
  return [
    result.boundaryMarker,        // system message: marca o limite de compressão
    ...result.summaryMessages,    // user message: conteúdo do resumo
    ...(result.messagesToKeep ?? []),  // mensagens originais preservadas
    ...result.attachments,        // anexos de arquivo + skills + planos
    ...result.hookResults,        // resultados dos hooks SessionStart
  ]
}
```

A reinjeção de anexos é um processo complexo (`compact.ts:532-585`), incluindo:

1. **Anexos de arquivo**: os 5 arquivos acessados mais recentemente, sujeitos a um orçamento de 50K tokens, com até 5K tokens por arquivo
2. **Arquivos de plano**: se houver um plano ativo
3. **Instruções do modo de plano**: se estiver no plan mode
4. **Conteúdo de skills**: conteúdo das skills invocadas, ordenado por uso mais recente, até 5K tokens cada, orçamento total de 25K tokens
5. **Deferred Tools Delta**: redeclara o schema de ferramentas carregadas sob demanda
6. **Agent Listing Delta**: redeclara a lista de agentes
7. **MCP Instructions Delta**: redeclara as instruções do servidor MCP

### 8.6.7 Mecanismo de Retry PTL (Recuperação de Prompt-Too-Long)

Quando a própria chamada de API de compressão falha por prompt muito longo, o sistema tenta novamente com truncamento progressivo:

```typescript
// compact.ts:242-290 — truncateHeadForPTLRetry
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  // primeiro remove mensagens marcadoras de retries anteriores
  const input = messages[0]?.type === 'user' && messages[0].isMeta &&
    messages[0].message.content === PTL_RETRY_MARKER
    ? messages.slice(1) : messages

  const groups = groupMessagesByApiRound(input)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount: number
  if (tokenGap !== undefined) {
    // truncamento preciso: baseado no gap de tokens retornado pela API
    let acc = 0
    dropCount = 0
    for (const g of groups) {
      acc += roughTokenCountEstimationForMessages(g)
      dropCount++
      if (acc >= tokenGap) break
    }
  } else {
    // truncamento impreciso: descarta 20% dos grupos de mensagens
    dropCount = Math.max(1, Math.floor(groups.length * 0.2))
  }

  dropCount = Math.min(dropCount, groups.length - 1)  // mantém ao menos um grupo
  if (dropCount < 1) return null

  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant') {
    return [
      createUserMessage({ content: PTL_RETRY_MARKER, isMeta: true }),
      ...sliced,
    ]
  }
  return sliced
}
```

O limite de tentativas é `MAX_PTL_RETRIES = 3`. A estratégia de truncamento tem dois caminhos:
- **Caminho preciso**: o erro de API inclui o gap de tokens → descarta grupos na ordem do gap
- **Caminho impreciso** (Vertex/Bedrock e outros formatos de erro não padronizados): descarta 20% a cada tentativa

Observe o tratamento de limite na linha 283: após descartar o grupo 0, a sequência de mensagens pode começar com uma assistant message, violando a restrição da API (a primeira mensagem deve ser user). O sistema insere uma user message sintética de marcação para corrigir isso.

### 8.6.8 Dois Sentidos da Compressão Parcial (Partial Compact)

`partialCompactConversation()` (`compact.ts:772-1106`) suporta dois sentidos:

```
Sentido 'from': 
  [preservado após compressão] | pivot | [mensagens resumidas]
  → preserva prompt cache (preservados à frente, cache prefix inalterado)

Sentido 'up_to':
  [mensagens resumidas] | pivot | [preservado após compressão]
  → invalida prompt cache (resumo à frente, prefix muda)
```

O sentido `up_to` possui uma lógica de limpeza adicional — as antigas compact boundary e summary devem ser removidas das mensagens preservadas:

```typescript
// compact.ts:791-799
const messagesToKeep =
  direction === 'up_to'
    ? allMessages.slice(pivotIndex)
        .filter(m =>
          m.type !== 'progress' &&
          !isCompactBoundaryMessage(m) &&
          !(m.type === 'user' && m.isCompactSummary))
    : allMessages.slice(0, pivotIndex).filter(m => m.type !== 'progress')
```

O comentário explica o motivo: no modo `up_to`, o resumo vem antes das mensagens preservadas, e a antiga boundary induziria `findLastCompactBoundaryIndex` a uma varredura reversa incorreta.

## 8.7 Camada 4: Compressão via Session Memory

### 8.7.1 Ideia Central e Vantagens

A compressão via Session Memory (`sessionMemoryCompact.ts`) é uma alternativa otimizada à compressão tradicional. A ideia central: usar a session memory extraída continuamente em background (um resumo incremental da conversa) no lugar de um resumo gerado em tempo real pelo Fork Agent.

Vantagens:
- **Zero chamadas de API adicionais**: a session memory é mantida continuamente por um agente em background; no momento da compressão, é usada diretamente
- **Menor latência**: não é necessário aguardar 5-15 segundos pela resposta da API
- **Preservação mais granular**: é possível calcular com precisão quantas mensagens recentes manter

### 8.7.2 Algoritmo Detalhado de calculateMessagesToKeepIndex

Este é o algoritmo central da compressão via Session Memory (`sessionMemoryCompact.ts:262-327`), que determina quantas mensagens são mantidas após a compressão:

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
): number {
  const config = getSessionMemoryCompactConfig()

  // começa a partir de lastSummarizedIndex + 1 (session memory já cobre o que vem antes)
  let startIndex = lastSummarizedIndex >= 0 
    ? lastSummarizedIndex + 1 
    : messages.length

  // calcula tokens e contagem de text messages no intervalo preservado atual
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
  }

  // já atingiu o máximo → não expande
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // já satisfaz os dois requisitos mínimos → não expande
  if (totalTokens >= config.minTokens && 
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // expande para trás, mas não ultrapassa o último compact boundary
  const idx = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  const floor = idx === -1 ? 0 : idx + 1

  for (let i = startIndex - 1; i >= floor; i--) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) textBlockMessageCount++
    startIndex = i
    if (totalTokens >= config.maxTokens) break
    if (totalTokens >= config.minTokens && 
        textBlockMessageCount >= config.minTextBlockMessages) break
  }

  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

Parâmetros de configuração (podem ser sobrescritos por configuração remota via GrowthBook):

```typescript
// sessionMemoryCompact.ts:60-64
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,           // mínimo de 10K tokens preservados
  minTextBlockMessages: 5,     // mínimo de 5 mensagens com texto
  maxTokens: 40_000,           // máximo de 40K tokens preservados
}
```

O design de dupla restrição do algoritmo (`minTokens` E `minTextBlockMessages`) garante que:
- A expansão não pare devido a poucas mensagens muito grandes (satisfaz tokens, mas mensagens demais pequenas)
- Não sejam preservadas muitas mensagens pequenas com tokens insuficientes no total

**Mecanismo de Floor**: a expansão retroativa não pode ultrapassar o último compact boundary (`floor = lastBoundaryIndex + 1`). O comentário explica o motivo:

```typescript
// sessionMemoryCompact.ts:308-312
// the preserved-segment chain has a disk discontinuity there
// (att[0]→summary shortcut from dedup-skip), which would let the
// loader's tail→head walk bypass inner preserved messages and then
// prune them.
```

A camada de armazenamento em disco tem uma descontinuidade na cadeia de mensagens no compact boundary; ultrapassá-lo faria o percurso reverso do carregador ignorar mensagens preservadas.

### 8.7.3 Correção de Bug em adjustIndexToPreserveAPIInvariants

Esta função (`sessionMemoryCompact.ts:172-260`) é o trecho de código mais sofisticado de todo o sistema de compressão; ela resolve dois problemas de invariantes de API:

**Cenário de Bug 1: tool_result órfão**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ORPHAN_ID]
Index N+2: assistant, message.id: X, content: [tool_use: VALID_ID]
Index N+3: user, content: [tool_result: ORPHAN_ID, tool_result: VALID_ID]

Se startIndex = N+2:
  código antigo verifica apenas tool_results da message N+2 → não encontra → retorna N+2
  após merging por message.id em normalizeMessagesForAPI:
    msg[1]: assistant with [tool_use: VALID_ID]  (ORPHAN tool_use excluído!)
    msg[2]: user with [tool_result: ORPHAN_ID, tool_result: VALID_ID]
  → erro de API: tool_result órfão referencia tool_use inexistente
```

**Cenário de Bug 2: bloco thinking perdido**

```
Index N:   assistant, message.id: X, content: [thinking]
Index N+1: assistant, message.id: X, content: [tool_use: ID]
Index N+2: user, content: [tool_result: ID]

Se startIndex = N+1:
  bloco thinking em N é excluído
  normalizeMessagesForAPI não consegue fazer merge (sem mensagem com mesmo ID para mesclar)
  → bloco thinking perdido permanentemente
```

O código de correção executa dois ajustes:

```typescript
// sessionMemoryCompact.ts:211-260
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[], startIndex: number
): number {
  let adjustedIndex = startIndex

  // Passo 1: trata pares tool_use/tool_result
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }
  if (allToolResultIds.length > 0) {
    const toolUseIdsInKeptRange = new Set<string>()
    // ... coleta tool_use IDs já no intervalo preservado
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id)))
    // busca retroativamente os tool_uses faltantes
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // remove IDs já encontrados
      }
    }
  }

  // Passo 2: trata blocos thinking com message.id compartilhado
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id)
      messageIdsInKeptRange.add(messages[i]!.message.id)
  }
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    if (messages[i]!.type === 'assistant' && messages[i]!.message.id &&
        messageIdsInKeptRange.has(messages[i]!.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

O insight-chave deste código é: as respostas em streaming da API Claude dividem uma única resposta de API em múltiplas assistant messages (compartilhando o mesmo `message.id`, mas com UUIDs distintos), com blocos thinking e tool_use separados. `normalizeMessagesForAPI` mescla essas mensagens pelo `message.id` — se a compressão cortar grupos de mensagens com o mesmo ID, a mesclagem resultará em inconsistências.

### 8.7.4 Fluxo Completo de trySessionMemoryCompaction

```typescript
// sessionMemoryCompact.ts:414-518
export async function trySessionMemoryCompaction(
  messages, agentId?, autoCompactThreshold?
): Promise<CompactionResult | null> {
  // 1. verificações de portão
  if (!shouldUseSessionMemoryCompaction()) return null

  // 2. inicializa configuração remota (apenas na primeira vez)
  await initSessionMemoryCompactConfig()

  // 3. aguarda extração de session memory em andamento ser concluída
  await waitForSessionMemoryExtraction()

  // 4. obtém conteúdo da session memory
  const lastSummarizedMessageId = getLastSummarizedMessageId()
  const sessionMemory = await getSessionMemoryContent()
  if (!sessionMemory) return null
  if (await isSessionMemoryEmpty(sessionMemory)) return null

  // 5. determina o limite
  let lastSummarizedIndex: number
  if (lastSummarizedMessageId) {
    lastSummarizedIndex = messages.findIndex(
      msg => msg.uuid === lastSummarizedMessageId)
    if (lastSummarizedIndex === -1) return null  // ID não encontrado → fallback
  } else {
    // sessão retomada: sem limite → começa do final
    lastSummarizedIndex = messages.length - 1
  }

  // 6. calcula intervalo de preservação
  const startIndex = calculateMessagesToKeepIndex(messages, lastSummarizedIndex)
  const messagesToKeep = messages.slice(startIndex)
    .filter(m => !isCompactBoundaryMessage(m))  // filtra boundaries antigas

  // 7. executa hooks de session start
  const hookResults = await processSessionStartHooks('compact', {...})

  // 8. constrói resultado
  const compactionResult = createCompactionResultFromSessionMemory(
    messages, sessionMemory, messagesToKeep, hookResults, transcriptPath, agentId)

  // 9. verificação de limiar (apenas para autocompact)
  const postCompactTokenCount = estimateMessageTokens(
    buildPostCompactMessages(compactionResult))
  if (autoCompactThreshold !== undefined && 
      postCompactTokenCount >= autoCompactThreshold) {
    return null  // ainda acima do limiar após compressão → fallback para compressão tradicional
  }

  return { ...compactionResult, postCompactTokenCount, truePostCompactTokenCount: postCompactTokenCount }
}
```

### 8.7.5 Parâmetros de Configuração (Configuração Remota via GrowthBook)

Todos os parâmetros-chave da compressão via Session Memory podem ser sobrescritos por configuração remota via GrowthBook:

```typescript
// sessionMemoryCompact.ts:91-109 — inicialização de configuração remota
async function initSessionMemoryCompactConfig(): Promise<void> {
  if (configInitialized) return
  configInitialized = true

  const remoteConfig = await getDynamicConfig_BLOCKS_ON_INIT<
    Partial<SessionMemoryCompactConfig>
  >('tengu_sm_compact_config', {})

  // codificação defensiva: usa apenas valores positivos, ignora valores 0
  const config: SessionMemoryCompactConfig = {
    minTokens: remoteConfig.minTokens && remoteConfig.minTokens > 0
      ? remoteConfig.minTokens : DEFAULT_SM_COMPACT_CONFIG.minTokens,
    // ...
  }
}
```

A habilitação da funcionalidade é controlada por dois feature flags independentes:

```typescript
// sessionMemoryCompact.ts:333-349
export function shouldUseSessionMemoryCompaction(): boolean {
  if (isEnvTruthy(process.env.ENABLE_CLAUDE_CODE_SM_COMPACT)) return true
  if (isEnvTruthy(process.env.DISABLE_CLAUDE_CODE_SM_COMPACT)) return false
  
  const sessionMemoryFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_session_memory', false)
  const smCompactFlag = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sm_compact', false)
  return sessionMemoryFlag && smCompactFlag
}
```

## 8.8 Context Collapse e Armazenamento de Resultados de Ferramentas

### 8.8.1 Mecanismo collapseReadSearch

`utils/collapseReadSearch.ts` (1.109 linhas) implementa o colapso de mensagens na camada de UI — recolhe operações consecutivas de busca/leitura em um resumo de linha única (por exemplo: "Read 5 files, searched 3 patterns").

A lógica central de classificação (`getToolSearchOrReadInfo`, `collapseReadSearch.ts:142-237`) divide chamadas de ferramentas em:

| Categoria | Colapsável | Comportamento de Colapso |
|-----------|-----------|--------------------------|
| Read (file_path) | Sim | "Read N files" |
| Search (Grep/Glob) | Sim | "Searched N patterns" |
| Shell (Bash) | Sim (modo tela cheia) | "Ran N bash commands" |
| REPL | Sim (absorção silenciosa) | contagem separada interna |
| Memory Write | Sim | marcação especial |
| ToolSearch | Sim (absorção silenciosa) | não incrementa contador |
| Edit/Write (não memory) | Não | interrompe grupo de colapso |

"Absorção silenciosa" (`isAbsorbedSilently`) é um design elegante: REPL e ToolSearch não incrementam o contador mas também não interrompem o grupo de colapso atual. Isso significa que `[Read, ToolSearch, Read]` é colapsado como "Read 2 files", em vez de ser cortado em dois grupos pelo ToolSearch.

O colapso é uma otimização **apenas da camada de UI** — não altera o conteúdo das mensagens enviadas à API, afetando apenas o display no terminal.

### 8.8.2 Estratégia de Armazenamento em Disco de toolResultStorage

`utils/toolResultStorage.ts` (1.040 linhas) é a "camada zero" do gerenciamento de contexto — processa resultados muito grandes antes que entrem no histórico de conversa.

**Análise do limiar de persistência**:

```typescript
// toolResultStorage.ts:54-77
export function getPersistenceThreshold(
  toolName: string,
  declaredMaxResultSizeChars: number,
): number {
  // ferramenta Read é especial: Infinity → não persiste (Read tem limite próprio de maxTokens)
  if (!Number.isFinite(declaredMaxResultSizeChars)) {
    return declaredMaxResultSizeChars
  }
  // sobrescrita via GrowthBook (tengu_satin_quoll)
  const overrides = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, number> | null>(
    PERSIST_THRESHOLD_OVERRIDE_FLAG, {})
  const override = overrides?.[toolName]
  if (typeof override === 'number' && Number.isFinite(override) && override > 0) {
    return override
  }
  // padrão: min(valor declarado pela ferramenta, padrão global de 50K)
  return Math.min(declaredMaxResultSizeChars, DEFAULT_MAX_RESULT_SIZE_CHARS)
}
```

**Otimização de deduplicação**: `tool_use_id` é único; usa `flag: 'wx'` (exclusive write) para evitar escritas duplicadas:

```typescript
// toolResultStorage.ts:160-170
try {
  await writeFile(filepath, contentStr, { encoding: 'utf-8', flag: 'wx' })
} catch (error) {
  if (getErrnoCode(error) !== 'EEXIST') {
    logError(toError(error))
    return { error: getFileSystemErrorMessage(toError(error)) }
  }
  // EEXIST: já persistido em rodada anterior, ignora
}
```

**Tratamento de resultado vazio**:

```typescript
// toolResultStorage.ts:279-294
// Empty tool_result content at the prompt tail causes some models
// (notably capybara) to emit the \n\nHuman: stop sequence
if (isToolResultContentEmpty(content)) {
  return { ...toolResultBlock, content: `(${toolName} completed with no output)` }
}
```

Essa correção resolve um bug de comportamento de modelo: tool_result vazio faz com que certos modelos correspondam ao padrão `\n\nHuman:` como fim de conversa.

**Per-Message Aggregate Budget** (`enforceToolResultBudget`, `toolResultStorage.ts:769-908`):

Esta é a funcionalidade mais complexa de `toolResultStorage.ts` — impõe um orçamento total de tamanho de tool results por user message em nível de API (após o merge de `normalizeMessagesForAPI`).

Pontos de design essenciais:
- **Congelamento de estado** (`ContentReplacementState`): uma vez que um tool_result foi "visto" (enviado ao modelo), sua decisão (substituir/não substituir) fica congelada e nunca muda — isso garante a estabilidade do prompt cache
- **Estratégia de três partições**: `mustReapply` (substituído anteriormente → reaplica o conteúdo de substituição em cache), `frozen` (visto anteriormente mas não substituído → não toca), `fresh` (novo → pode ser substituído)
- **Maior primeiro**: quando é necessário substituir, seleciona os maiores resultados fresh primeiro

## 8.9 O Papel da Compressão na Recuperação de Erros em 5 Camadas

### 8.9.1 Mecanismo Completo de Recuperação de Erros em 5 Camadas

O sistema de compressão desempenha múltiplos papéis no mecanismo de recuperação de erros do Claude Code:

| Camada | Gatilho | Comportamento de Compressão | Origem |
|--------|---------|----------------------------|--------|
| L1 | API retorna prompt_too_long (413) | Reactive Compact: trunca + gera novo resumo | `compactMessages.ts` |
| L2 | API de compressão retorna 413 | PTL Retry: trunca grupos de mensagens mais antigos × 3 vezes | `compact.ts:truncateHeadForPTLRetry` |
| L3 | Ainda acima do limiar após compressão | Re-compaction: comprime automaticamente de novo | `autoCompact.ts:recompactionInfo` |
| L4 | 3 falhas consecutivas de compressão | Circuit Breaker: para de tentar | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` |
| L5 | Fork Agent sem saída de texto | Streaming Fallback: chamada direta de API em streaming | `compact.ts:streamCompactSummary` |

### 8.9.2 Compressão Reativa vs. Compressão Proativa

As trocas entre as duas estratégias:

**Compressão Proativa** (Auto-Compact, padrão atual):
- Acionada proativamente quando os tokens atingem o limiar
- Vantagem: experiência de usuário mais suave, sem erros 413
- Desvantagem: pode comprimir prematuramente, desperdiçando contexto disponível

**Compressão Reativa** (Reactive Compact, experimento `tengu_cobalt_raccoon`):
- Aguarda a API reportar prompt_too_long antes de agir
- Vantagem: maximiza a utilização do contexto
- Desvantagem: experiência de usuário com interrupção visível, requer espera pelo retry

A relação de exclusão mútua entre as duas pode ser vista no código:

```typescript
// autoCompact.ts:157-162
if (feature('REACTIVE_COMPACT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
    return false  // no modo reativo, não comprime proativamente
  }
}
```

## 8.10 Agrupamento de Mensagens e Estimativa de Tokens

### 8.10.1 Algoritmo groupMessagesByApiRound

`grouping.ts` (63 linhas) agrupa mensagens por rodada de API — cada grupo corresponde a uma ida e volta completa de API:

```typescript
// grouping.ts:28-62
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (msg.type === 'assistant' && 
        msg.message.id !== lastAssistantId && 
        current.length > 0) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }
  if (current.length > 0) groups.push(current)
  return groups
}
```

O único critério para o limite de grupo é a mudança de `message.id` — múltiplos blocos em streaming da mesma resposta de API compartilham o mesmo `message.id`, de modo que naturalmente são agrupados juntos.

Esse design substitui o agrupamento anterior baseado em "turnos humanos" (que agrupava apenas nas mensagens reais do usuário), o qual não conseguia lidar com sessões de agente longas de turno único em cenários SDK/CCR/eval.

### 8.10.2 roughTokenCountEstimation e Padding Conservador

A estimativa de tokens adota uma estratégia de dois níveis de conservadorismo:

**Primeiro nível**: estimativa básica

```typescript
// tokenEstimation.ts:203-207
export function roughTokenCountEstimation(
  content: string, bytesPerToken: number = 4
): number {
  return Math.round(content.length / bytesPerToken)
}
```

Padrão: 4 bytes/token; arquivos JSON usam 2 bytes/token (pois JSON tem abundância de tokens de caractere único como `{`, `}`, `:`, `,`).

**Segundo nível**: padding em nível de mensagem

```typescript
// microCompact.ts:192-193
// Pad estimate by 4/3 to be conservative since we're approximating
return Math.ceil(totalTokens * (4 / 3))
```

Efeito combinado: para texto comum, a estimativa efetiva é `text.length / 4 * 4/3 = text.length / 3`.

### 8.10.3 Estratégia Híbrida de Estimativa Precisa vs. Aproximada

O sistema usa diferentes níveis de precisão em cenários distintos:

| Cenário | Precisão | Origem | Latência |
|---------|----------|--------|----------|
| shouldAutoCompact | Híbrida: prioriza valor preciso retornado pela API | `tokenCountWithEstimation` | 0 (já em cache) |
| estimateMessageTokens | Estimativa grosseira (`text.length/3`) | `roughTokenCountEstimation` | 0 |
| calculateMessagesToKeepIndex | Estimativa grosseira | `estimateMessageTokens` | 0 |
| Estatísticas de tokens pós-compact | Precisa | `tokenCountFromLastAPIResponse` | 0 (API já retornou) |

A estratégia híbrida de `tokenCountWithEstimation` é: prioriza o `usage.input_tokens` (valor preciso) retornado pela chamada de API mais recente; se indisponível (por exemplo, antes da primeira requisição), recorre à estimativa.

## 8.11 Análise de Decisões de Design

### 8.11.1 Filosofia de Degradação Gradual

O gerenciamento de contexto do Claude Code adota uma degradação gradual **sem saltos de camada**: cada camada tenta resolver o problema com o mínimo de custo, e somente quando a camada atual falha é que se escala para a seguinte. Isso evita o problema comum de "reação excessiva" — por exemplo, acionar uma compressão completa apenas por causa de um grande resultado de leitura de arquivo.

Comparação com práticas da indústria:
- **ChatGPT**: trunca mensagens antigas (simples, mas bruto)
- **GitHub Copilot Chat**: janela de contexto fixa + últimas N mensagens (sem compressão)
- **Claude Code**: 5 camadas progressivas (prevenção → ajuste fino → resumo → recuperação de emergência)

### 8.11.2 Design Cache-First

O prompt cache é a linha de vida do Claude Code — uma requisição de 200K tokens em que 180K são cache reads ($0,30/M tokens) custa 10 vezes menos do que se fossem todos cache misses ($3/M tokens). Quase todas as decisões de design servem a essa restrição econômica:

1. **Fork Agent compartilha prefixo de cache**: a chamada de API de compressão reutiliza o cache da conversa principal
2. **maxOutputTokens não definido no Fork**: evita incompatibilidade de thinking config que causaria cache miss
3. **Cached MC não modifica mensagens locais**: mantém o prompt prefix inalterado
4. **ContentReplacementState congela IDs já vistos**: garante que a decisão de substituição de um mesmo tool_result não muda durante seu ciclo de vida
5. **sentSkillNames não reinicia**: evita reinjetar ~4K tokens de skill_listing
6. **pinnedCacheEdits reenviado na posição fixa**: garante consistência da posição de cache edit

### 8.11.3 Garantias de Segurança

O sistema mantém três classes de invariantes:

**Pares não podem ser cortados**: `adjustIndexToPreserveAPIInvariants` garante que tool_use e tool_result nunca ficam em lados opostos do corte. Isso é um requisito tanto de correção funcional (a API reportaria erro) quanto de correção semântica (o modelo precisa ver o resultado da ferramenta que chamou anteriormente).

**Proteção contra recursão**: a verificação de `querySource` em `shouldAutoCompact` garante que o agente de session_memory, o agente de compact e o agente de context collapse não acionam compressão automática — esses agentes fazem parte do gerenciamento de contexto em si, e a compressão recursiva causaria deadlock.

**Mecanismo de circuit breaker**: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` foi estabelecido com base em dados reais (loops de falha em 1.279 sessões), convertendo retries infinitos em retries finitos + disparador.

### 8.11.4 Comparação com API-native Context Management

`apiMicrocompact.ts` revela que o Claude Code está explorando a direção de descarregar parte do gerenciamento de contexto para a camada de API:

```typescript
// apiMicrocompact.ts:37-47
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

Essas estratégias `context_management.edits` são declaradas diretamente na requisição de API e executadas pelo lado do servidor. As vantagens são menor latência (sem necessidade de processamento no cliente) e alinhamento preciso com a contagem de tokens do servidor. Atualmente, apenas usuários internos (`USER_TYPE === 'ant'`) têm acesso à estratégia de limpeza de ferramentas; usuários externos usam apenas a limpeza de thinking.

## 8.12 Padrões Transferíveis

### 8.12.1 Padrões de Design Gerais do Sistema de Compressão em Múltiplas Camadas

O gerenciamento de contexto do Claude Code destila os seguintes padrões gerais transferíveis:

**Padrão 1: Eviction em Camadas (Tiered Eviction)**
- Aplica estratégias de eviction diferentes para diferentes tipos de conteúdo
- Conteúdo regenerável (saídas de ferramentas) é evicted primeiro; conteúdo não regenerável (input do usuário) por último
- Implementação: whitelist + ordenação por prioridade

**Padrão 2: Estimativa-Precisão Híbrida (Hybrid Estimation)**
- Decisões rápidas usam estimativas grosseiras (`text.length / 3`); apuração precisa usa valores retornados pela API
- Estimativas são sempre conservadoras (melhor superestimar e comprimir cedo do que subestimar e ter erros de API)

**Padrão 3: Congelar-Replay (Freeze-Replay)**
- Uma vez que o conteúdo foi "visto" pelo modelo, sua decisão de processamento fica congelada
- Rodadas subsequentes apenas "reproduzem" (re-apply substituição em cache) conteúdo congelado, sem novas decisões
- Garante estabilidade bit a bit do prompt prefix → cache hit

**Padrão 4: Truncamento Consciente de Limites (Boundary-Aware Truncation)**
- Nunca trunca no meio de unidades semânticas (pares tool_use/tool_result, grupos de mensagens com mesmo ID)
- Após o truncamento, corrige proativamente (insere mensagens sintéticas, ajusta índices)

**Padrão 5: Proteção por Circuit Breaker (Circuit Breaker Protection)**
- Define contador de falhas para operações que poderiam entrar em loop infinito de retries
- Estabelece limiares com base em dados operacionais reais (não em intuição)

### 8.12.2 Lições Aplicáveis ao Doramagic

No pipeline Soul Extractor do Doramagic, o processo de extração pode gerar grandes volumes de resultados intermediários (trechos de código, documentação de API, discussões da comunidade). Padrões que podem ser aproveitados:

1. **Cache de extração em camadas**: similar ao mecanismo de whitelist do microcompact, classifica respostas intermediárias de API e resultados de análise de código por regenerabilidade, priorizando a eliminação de conteúdo que pode ser recuperado novamente
2. **Resumo incremental**: similar ao Session Memory Compact, mantém resumos incrementais do conhecimento extraído em vez de histórico completo
3. **Decisões congeladas**: uma vez que um bloco de conhecimento é classificado como "valioso" ou "sem valor", a decisão é irreversível — evita reavaliações repetidas em diferentes rodadas de extração

## 8.13 Índice de Código-fonte

| Arquivo | Linhas | Responsabilidade Principal |
|---------|--------|---------------------------|
| `services/compact/compact.ts` | ~1.705 | Lógica principal da compressão tradicional: Fork Agent, PTL retry, reinjeção de anexos, compressão parcial |
| `services/compact/sessionMemoryCompact.ts` | ~630 | Compressão via Session Memory: calculateMessagesToKeepIndex, adjustIndexToPreserveAPIInvariants |
| `services/compact/microCompact.ts` | ~530 | Micro-compressão: gatilho temporal, cache editing, estimativa de tokens |
| `services/compact/prompt.ts` | ~374 | Prompt de compressão: template de 9 seções, NO_TOOLS_PREAMBLE, formatCompactSummary |
| `services/compact/autoCompact.ts` | ~351 | Compressão automática: cálculo de limiar, cadeia de decisão de shouldAutoCompact, circuit breaker |
| `services/compact/apiMicrocompact.ts` | ~153 | Gerenciamento de contexto nativo de API: clear_tool_uses, clear_thinking |
| `services/compact/grouping.ts` | ~63 | Agrupamento de mensagens: groupMessagesByApiRound |
| `services/compact/postCompactCleanup.ts` | ~77 | Limpeza pós-compressão: reinicia cache, estado de módulo, classificadores |
| `services/compact/timeBasedMCConfig.ts` | ~43 | Configuração de gatilho temporal: configuração remota via GrowthBook |
| `services/compact/compactWarningHook.ts` | ~16 | React hook: assinatura de estado de supressão de aviso de compact |
| `services/compact/compactWarningState.ts` | ~18 | Armazenamento de estado: flag de supressão de aviso de compact |
| `services/cost-tracker.ts` | ~323 | Rastreamento de custos: cobrança de tokens, estatísticas de uso de modelos |
| `utils/collapseReadSearch.ts` | ~1.109 | Context collapse: agrupamento e colapso de mensagens na camada de UI |
| `utils/toolResultStorage.ts` | ~1.040 | Armazenamento de resultados de ferramentas: persistência em disco, orçamento por mensagem, ContentReplacementState |
| `services/tokenEstimation.ts` | ~350+ | Estimativa de tokens: roughTokenCountEstimation (text.length/4) |
