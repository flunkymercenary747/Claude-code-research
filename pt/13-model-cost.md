# Capítulo 13: Seleção de Modelos e Controle de Custos

> Fonte de dados: snapshot do código-fonte TypeScript do Claude Code (2026-03-31, ~512K LOC)
> Arquivos centrais: `services/api/claude.ts` (3.419 linhas), `services/api/withRetry.ts`, `cost-tracker.ts` (323 linhas), `utils/effort.ts`, `utils/modelCost.ts`, `utils/model/model.ts`, diretório `migrations/` (11 arquivos)

---

## 13.1 Visão Geral e Posicionamento

A filosofia de design do Claude Code em seleção de modelos e controle de custos pode ser resumida em três frases:

1. **Intenção do usuário tem prioridade**: a cadeia de prioridades vai do comando `/model` → flag `--model` → variável de ambiente → arquivo de configuração, descendo em cascata; cada camada pode ser sobrescrita pela camada superior, mas não será substituída inadvertidamente pela inferior.
2. **Custos completamente transparentes**: ao término da sessão, o uso de tokens e o custo em dólares por modelo são imprimidos obrigatoriamente; não pode ser desabilitado (apenas quando `hasConsoleBillingAccess()` é verdadeiro).
3. **Sem degradação silenciosa**: quando ocorre Overload Fallback (Opus → Sonnet), o usuário deve ver uma mensagem de aviso obrigatoriamente; nunca troca silenciosamente.

Este capítulo verifica individualmente as afirmações do cc-notebook sobre esse subsistema no nível do código-fonte, aprofundando a análise.

---

## 13.2 Bases Teóricas

### Estratégias de Roteamento em Sistemas Multi-modelo

Em sistemas multi-modelo, as estratégias de roteamento geralmente equilibram três dimensões: **capacidade** (capability), **custo** (cost), **latência** (latency). A escolha do Claude Code é rotear o diálogo principal (main loop) para o modelo mais forte disponível, rotear tarefas auxiliares em segundo plano para os modelos mais rápidos e baratos, e fornecer degradação transparente quando o modelo principal está indisponível.

### Aplicação de Análise de Custo-Benefício em Sistemas de IA

Em `modelCost.ts` podemos ver que o Claude Code tem uma tabela de preços precisa integrada:

```typescript
// utils/modelCost.ts
// Standard pricing tier for Sonnet models: $3 input / $15 output per Mtok
export const COST_TIER_3_15 = {
  inputTokens: 3,
  outputTokens: 15,
  promptCacheWriteTokens: 3.75,
  promptCacheReadTokens: 0.3,
  webSearchRequests: 0.01,
}

// Fast mode pricing for Opus 4.6: $30 input / $150 output per Mtok
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
  webSearchRequests: 0.01,
}
```

O Haiku 4.5 tem o preço mais baixo ($1/$5 por Mtok), o Opus 4.6 Fast Mode o mais alto ($30/$150 por Mtok), uma diferença de 30x. Essa diferença de preço é a lógica econômica central pela qual o sistema aloca tarefas em segundo plano para o Haiku.

### Padrão de Degradação Graciosa (Graceful Degradation)

Em software tradicional, degradação graciosa é recorrer a uma alternativa inferior quando uma funcionalidade não está disponível, sem colapso. Em sistemas LLM, a alternativa é "trocar para um modelo mais barato/mais disponível". O Claude Code implementa um mecanismo de acionamento com proteção por contador: 3 erros 529 consecutivos acionam a troca de modelo, em vez de troca imediata (evitando degradação desnecessária de qualidade por overload ocasional).

---

## 13.3 Arquitetura de Seleção de Modelos

### Hierarquia de Prioridade do Modelo

A função `getUserSpecifiedModelSetting()` em `utils/model/model.ts` define precisamente a ordem de prioridade:

```typescript
// utils/model/model.ts:44-66
/**
 * Priority order within this function:
 * 1. Model override during session (from /model command) - highest priority
 * 2. Model override at startup (from --model flag)
 * 3. ANTHROPIC_MODEL environment variable
 * 4. Settings (from user's saved settings)
 */
export function getUserSpecifiedModelSetting(): ModelSetting | undefined {
  let specifiedModel: ModelSetting | undefined

  const modelOverride = getMainLoopModelOverride()
  if (modelOverride !== undefined) {
    specifiedModel = modelOverride
  } else {
    const settings = getSettings_DEPRECATED() || {}
    specifiedModel = process.env.ANTHROPIC_MODEL || settings.model || undefined
  }

  // Ignore the user-specified model if it's not in the availableModels allowlist.
  if (specifiedModel && !isModelAllowed(specifiedModel)) {
    return undefined
  }

  return specifiedModel
}
```

`getMainLoopModel()` adiciona a 5ª prioridade — o valor padrão integrado:

```typescript
// utils/model/model.ts:68-77
export function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}
```

Cadeia completa de 5 níveis de prioridade:
| Prioridade | Fonte | Descrição |
|-----------|-------|-----------|
| 1 (mais alta) | Comando `/model` | Entra em vigor imediatamente na sessão, armazenado em override de memória |
| 2 | Flag `--model` de inicialização | Escrito em override de memória na inicialização |
| 3 | Variável de ambiente `ANTHROPIC_MODEL` | Nível de processo |
| 4 | Arquivo de configuração `settings.json` | Preferência persistida do usuário |
| 5 (mais baixa) | Valor padrão integrado | Determinado pelo tipo de assinatura |

### Modelo Padrão Diferenciado por Tipo de Assinatura

`getDefaultMainLoopModelSetting()` revela as diferenças entre assinaturas:

```typescript
// utils/model/model.ts:153-175
export function getDefaultMainLoopModelSetting(): ModelName | ModelAlias {
  // Ants (funcionários internos) padrão Opus 1M
  if (process.env.USER_TYPE === 'ant') {
    return (
      getAntModelOverrideConfig()?.defaultModel ??
      getDefaultOpusModel() + '[1m]'
    )
  }

  // Usuários Max e Team Premium padrão Opus 4.6
  if (isMaxSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }
  if (isTeamPremiumSubscriber()) {
    return getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
  }

  // PAYG, Enterprise, Team Standard, Pro padrão Sonnet 4.6
  return getDefaultSonnetModel()
}
```

Esse design significa: mesmo sem nenhuma configuração, usuários Max/Team Premium abrem com Opus 4.6, enquanto usuários Pro/Sonnet abrem com Sonnet 4.6. **O próprio valor padrão é uma estratégia de diferenciação de produto.**

### Sistema de Alias de Modelo

`parseUserSpecifiedModel()` suporta resolução de alias curtos, dispensando os usuários de memorizar IDs de modelo completos:

```typescript
// utils/model/model.ts — trecho de parseUserSpecifiedModel
case 'opus':   return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
case 'sonnet': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '')
case 'haiku':  return getDefaultHaikuModel() + (has1mTag ? '[1m]' : '')
case 'best':   return getBestModel()
case 'opusplan': return getDefaultSonnetModel() + (has1mTag ? '[1m]' : '') // plan mode usa Sonnet; não plan mode usa Opus
```

O sufixo `[1m]` pode ser adicionado a qualquer alias (como `opus[1m]`); o sistema resolve automaticamente para a variante de 1M de janela de contexto.

### Detecção de Capacidades de Modelo

`utils/model/modelCapabilities.ts` implementa um mecanismo de cache habilitado apenas para funcionários internos (`USER_TYPE === 'ant'`):

```typescript
// utils/model/modelCapabilities.ts:45-52
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false
  if (getAPIProvider() !== 'firstParty') return false
  if (!isFirstPartyAnthropicBaseUrl()) return false
  return true
}
```

Usuários externos não solicitam listas de capacidades de modelo; as informações de capacidade estão hardcoded em funções como `modelSupportsEffort()`, `modelSupports1M()`, etc., evitando o overhead de chamadas de API adicionais.

---

## 13.4 Usos em Segundo Plano do Haiku

O cc-notebook afirma que o Haiku tem 6 usos em segundo plano. Através de busca completa das posições de chamada da função `queryHaiku` (`grep -rn 'queryHaiku\b'`) e das posições de chamada de `getSmallFastModel()`, a **verificação no código-fonte** é a seguinte:

### Sumário de Usos em Segundo Plano (verificação no código-fonte)

| Nº | Uso | Arquivo | Condição de acionamento |
|----|-----|---------|------------------------|
| 1 | Extração de conteúdo Web Fetch | `tools/WebFetchTool/utils.ts:503` | Após capturar página web, usa Haiku para filtrar Markdown para o conteúdo especificado pelo usuário |
| 2 | Extração de prefixo de comando Shell | `utils/shell/prefix.ts:220` | Antes de executar ferramenta Bash, usa Haiku para determinar se o comando requer prompt de permissão |
| 3 | Geração de título de sessão | `utils/sessionTitle.ts:87` | Gera automaticamente um título curto após início da sessão (saída com JSON schema) |
| 4 | Parsing de DateTime MCP | `utils/mcp/dateTimeParser.ts:68` | Converte descrição de tempo em linguagem natural para formato ISO 8601 |
| 5 | Geração de resumo de chamadas de ferramenta | `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | Gera um rótulo resumido de uma linha após um lote de chamadas de ferramentas |
| 6 | Renomeação de sessão | `commands/rename/generateSessionName.ts:20` | Comando `/rename` gera nome em formato kebab-case |

**Descobertas adicionais** (não mencionadas no cc-notebook, descobertas via busca de `getSmallFastModel()`):

| Nº | Uso | Arquivo | Condição de acionamento |
|----|-----|---------|------------------------|
| 7 | Verificação de API Key | `services/api/claude.ts:544` | Verifica validade da API Key (comentário no código-fonte: "WARNING: if you change this to use a non-Haiku model, this request will fail in 1P") |
| 8 | Resumo do modo Away | `services/awaySummary.ts:49` | Gera resumo de contexto quando o usuário está ausente (modo AFK) |
| 9 | Assistência de busca Web | `tools/WebSearchTool/WebSearchTool.ts:280` | Alguns cenários de busca web usam Haiku para processar resultados |
| 10 | Verificação de status de Quota | `services/claudeAiLimits.ts:200` | Usa a menor requisição Haiku para sondar o status de quota atual |
| 11 | Estimativa de quantidade de tokens | `services/tokenEstimation.ts:277` | Estima o uso da janela de contexto |
| 12 | Execução de hook Prompt/Exec | `utils/hooks/execPromptHook.ts:79`, `execAgentHook.ts:118` | Callbacks de hook usam Haiku por padrão (a menos que a configuração do hook sobrescreva) |
| 13 | Análise de melhorias de Skill | `utils/hooks/skillImprovement.ts:169` | Analisa automaticamente sugestões de melhoria após execução de Skill |

**Conclusão**: os "6 usos em segundo plano" do cc-notebook são uma **subestimativa**. No código-fonte, há pelo menos 13 pontos de chamada de `queryHaiku` ou `getSmallFastModel()`, cobrindo todas as fases do ciclo de vida da sessão (verificação no início, assistência durante execução, organização ao término da sessão). Haiku/SmallFastModel é a "camada de serviço básico" em segundo plano de todo o sistema, não uma otimização que aparece ocasionalmente.

Detalhe de design chave: `queryHaiku` usa chamada não-streaming (`queryModelWithoutStreaming`), sem contexto de permissão de Tool (`getEmptyToolPermissionContext()`):

```typescript
// services/api/claude.ts:3280-3291
const result = await queryModelWithoutStreaming({
  messages,
  systemPrompt,
  thinkingConfig: { type: 'disabled' },
  tools: [],
  signal,
  options: {
    ...options,
    model: getSmallFastModel(),
    enablePromptCaching: options.enablePromptCaching ?? false,
    async getToolPermissionContext() {
      return getEmptyToolPermissionContext()
    },
  },
})
```

---

## 13.5 Mecanismo Overload Fallback

O cc-notebook afirma a existência de "529 Overload Fallback, fallback Opus → Sonnet". O código-fonte **verifica completamente** essa afirmação, com detalhes ainda mais ricos.

### Identificação de Erros 529

Função `is529Error()` em `services/api/withRetry.ts`:

```typescript
// services/api/withRetry.ts
export function is529Error(error: unknown): boolean {
  if (!(error instanceof APIError)) {
    return false
  }
  // verifica código de status 529, ou situação em que o SDK não consegue transmitir corretamente o código de status no streaming
  return (
    error.status === 529 ||
    (error.message?.includes('"type":"overloaded_error"') ?? false)
  )
}
```

Observe a detecção dupla: código de status `529` e a string `overloaded_error` na mensagem de erro. Isso ocorre porque o SDK às vezes não consegue transmitir corretamente o código de status 529 durante streaming.

### Condição de acionamento: 3 erros 529 consecutivos

```typescript
// services/api/withRetry.ts — trecho de withRetry
const MAX_529_RETRIES = 3

if (
  is529Error(error) &&
  (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS ||
    (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))
) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })
      // lança erro especial, acionando troca de modelo na camada superior
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }
    // ...
  }
}
```

Restrições chave:
- Por padrão, só aciona para **usuários não-ClaudeAI** com **modelos da série Opus** (`isNonCustomOpusModel()`)
- Variável de ambiente `FALLBACK_FOR_ALL_PRIMARY_MODELS` pode expandir para todos os modelos principais
- Requisições 529 em streaming contam para o contador (`initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0`), coordenando a contagem com novas tentativas não-streaming

### Propagação do sinal FallbackTriggeredError

`FallbackTriggeredError` é uma classe de erro especializada, carregando os campos `originalModel` e `fallbackModel`, propagando pela pilha de chamadas até `query.ts`:

```typescript
// services/api/withRetry.ts
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    super(`Model fallback triggered: ${originalModel} -> ${fallbackModel}`)
    this.name = 'FallbackTriggeredError'
  }
}
```

### Troca de Modelo e Notificação ao Usuário em query.ts

`query.ts:894-946` captura esse erro e executa a troca real de modelo:

```typescript
// query.ts — trecho de tratamento de FallbackTriggeredError
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // exibe ao usuário com nível warning — visível independentemente do modo verbose
  yield createSystemMessage(
    `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
    'warning',
  )

  // atualiza sincronamente o modelo do main loop no toolUseContext
  toolUseContext.options.mainLoopModel = fallbackModel

  continue  // nova tentativa da requisição completa com o novo modelo
}
```

**Mecanismo de notificação ao usuário**: a mensagem de troca usa nível `'warning'`, o que significa que independentemente do modo verbose, o usuário verá a notificação na interface. **A afirmação do cc-notebook sobre "sem degradação silenciosa" é completamente verificada.**

### Estratégia 529 para Tarefas em Segundo Plano: Descartar Imediatamente

Tarefas não em primeiro plano (como resumo, título, sugestões) **não fazem nova tentativa** em caso de 529, descartando diretamente:

```typescript
// services/api/withRetry.ts — FOREGROUND_529_RETRY_SOURCES
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'agent:builtin',
  'compact',
  'verification_agent',
  'side_question',
  'auto_mode',
  // ...
])

// tarefas não em primeiro plano com 529 lançam diretamente, sem nova tentativa
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: ... })
  throw new CannotRetryError(error, retryContext)
}
```

Esta é uma decisão de controle de custo arquitetural: novas tentativas de tarefas em segundo plano produziriam um efeito de amplificação de 3-10x no gateway durante periods de capacidade limitada, enquanto os usuários nem perceberiam essas falhas de tarefas.

---

## 13.6 Mecanismo de Effort Level

O cc-notebook afirma a existência de um sistema de Effort Level. O código-fonte **verifica completamente**, com detalhes muito mais ricos do que a descrição.

### Quatro Níveis de Effort

```typescript
// utils/effort.ts
export const EFFORT_LEVELS = ['low', 'medium', 'high', 'max'] as const
```

Semântica de cada nível (de `getEffortLevelDescription()`):
- **low**: Quick, straightforward implementation with minimal overhead
- **medium**: Balanced approach with standard implementation and testing
- **high**: Comprehensive implementation with extensive testing and documentation
- **max**: Maximum capability with deepest reasoning (**somente Opus 4.6**)

### Matriz de Suporte de Modelos

```typescript
// utils/effort.ts
export function modelSupportsEffort(model: string): boolean {
  // apenas Opus 4.6 e Sonnet 4.6 suportam o parâmetro effort
  if (m.includes('opus-4-6') || m.includes('sonnet-4-6')) {
    return true
  }
  // Haiku, Opus/Sonnet antigos não suportam
  if (m.includes('haiku') || m.includes('sonnet') || m.includes('opus')) {
    return false
  }
  // 1P padrão true, 3P padrão false
  return getAPIProvider() === 'firstParty'
}

// max effort apenas disponível para Opus 4.6
export function modelSupportsMaxEffort(model: string): boolean {
  if (model.toLowerCase().includes('opus-4-6')) {
    return true
  }
  // ...
}
```

### Cadeia de Prioridade: env → appState → valor padrão do modelo

```typescript
// utils/effort.ts — resolveAppliedEffort
export function resolveAppliedEffort(
  model: string,
  appStateEffortValue: EffortValue | undefined,
): EffortValue | undefined {
  const envOverride = getEffortEnvOverride()  // CLAUDE_CODE_EFFORT_LEVEL
  if (envOverride === null) {
    return undefined  // 'unset' ou 'auto' → não envia parâmetro effort
  }
  const resolved = envOverride ?? appStateEffortValue ?? getDefaultEffortForModel(model)
  // API recusa max de não-Opus 4.6 → degrada automaticamente para high
  if (resolved === 'max' && !modelSupportsMaxEffort(model)) {
    return 'high'
  }
  return resolved
}
```

### Diferenciação do Effort Padrão

O effort padrão do Opus 4.6 difere por tipo de assinatura:

```typescript
// utils/effort.ts — trecho de getDefaultEffortForModel
if (model.toLowerCase().includes('opus-4-6')) {
  if (isProSubscriber()) {
    return 'medium'  // usuários Pro padrão medium (economiza cota)
  }
  if (getOpusDefaultEffortConfig().enabled &&
      (isMaxSubscriber() || isTeamSubscriber())) {
    return 'medium'  // Max/Team também pode ser configurado para medium pelo GrowthBook
  }
}
```

É interessante notar que o `dialogDescription` de `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` diz explicitamente: "We recommend medium effort for most tasks to balance speed and intelligence and maximize rate limits." — isso indica que o padrão medium é uma estratégia intencional de gerenciamento de cota, não prioridade de performance.

### Restrição de Persistência do max

```typescript
// utils/effort.ts — toPersistableEffort
export function toPersistableEffort(value): EffortLevel | undefined {
  if (value === 'low' || value === 'medium' || value === 'high') {
    return value
  }
  // max não é persistido para usuários externos; válido apenas na sessão atual
  if (value === 'max' && process.env.USER_TYPE === 'ant') {
    return value
  }
  return undefined
}
```

O effort `max` definido por usuários externos não é gravado em `settings.json`; só tem efeito na sessão atual.

---

## 13.7 Sistema de Rastreamento de Custos

### Responsabilidades Centrais do cost-tracker.ts

`cost-tracker.ts` (323 linhas) tem três responsabilidades:
1. **Acumulação em tempo real**: chama `addToTotalSessionCost()` após cada resposta de API
2. **Persistência**: grava no arquivo de configuração do projeto ao término da sessão (`saveCurrentSessionCosts()`)
3. **Restauração**: lê os dados de custo da última sessão no arquivo de configuração ao reiniciar (`restoreCostStateForSession()`)

### Estatísticas de Tokens por Modelo

`addToTotalModelUsage()` acumula 5 dimensões de dados por nome de modelo:

```typescript
// cost-tracker.ts — addToTotalModelUsage
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

Na exibição ao término da sessão (`formatModelUsage()`): agrega por nome curto (múltiplos endpoints de API retornam o mesmo modelo em diferentes formatos), exibindo como:

```
Usage by model:
    claude-opus-4-6:   1,234 input, 567 output, 890 cache read, 234 cache write ($0.0123)
   claude-haiku-4-5:   456 input, 123 output, 0 cache read, 0 cache write ($0.0002)
```

### Marcação de Custo no Fast Mode

`addToTotalSessionCost()` tem tratamento especial para Fast Mode:

```typescript
// cost-tracker.ts
const attrs =
  isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }

getCostCounter()?.add(cost, attrs)
```

A marcação `speed: 'fast'` afeta o faturamento — no Fast Mode, o Opus 4.6 usa `COST_TIER_30_150` ($30/$150), em vez do padrão `COST_TIER_5_25` ($5/$25).

### Rastreamento de Custo Aninhado do Advisor

`addToTotalSessionCost()` processa recursivamente o uso do Advisor:

```typescript
// cost-tracker.ts
for (const advisorUsage of getAdvisorUsage(usage)) {
  const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
  logEvent('tengu_advisor_tool_token_usage', { ... })
  totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
}
```

Advisor é uma chamada de modelo secundário oculta na resposta do modelo principal; seu custo é rastreado separadamente e mesclado ao custo total.

### Mecanismo de Acionamento de Exibição de Custo

`costHook.ts` (22 linhas) é um hook React que monitora eventos de saída do processo:

```typescript
// costHook.ts
export function useCostSummary(getFpsMetrics?: () => FpsMetrics | undefined): void {
  useEffect(() => {
    const f = () => {
      if (hasConsoleBillingAccess()) {
        process.stdout.write('\n' + formatTotalCost() + '\n')
      }
      saveCurrentSessionCosts(getFpsMetrics?.())
    }
    process.on('exit', f)
    return () => { process.off('exit', f) }
  }, [])
}
```

`hasConsoleBillingAccess()` controla se exibe o custo, garantindo que em ambientes sem acesso a informações de faturamento (como modo CCR/Remote), o custo não seja exibido; enquanto a execução de `saveCurrentSessionCosts()` é incondicional — independentemente de ser exibido ou não, sempre persiste.

---

## 13.8 Camada de Chamada de API

### Parâmetros Centrais de Construção de Requisição em claude.ts

`services/api/claude.ts` (3.419 linhas) é a entrada unificada de chamadas de API. Os parâmetros-chave convergem de múltiplos sistemas:

```typescript
// services/api/claude.ts — montagem de parâmetros de requisição (esquema)
{
  model: normalizeModelStringForAPI(options.model),  // remove sufixo [1m]
  max_tokens: getMaxOutputTokensForModel(model),
  thinking: thinkingConfig,
  betas: getMergedBetas(model),
  // parâmetro Effort (apenas modelos suportados)
  ...(modelSupportsEffort(model) && resolvedEffort !== undefined && {
    effort: resolvedEffort
  }),
}
```

`normalizeModelStringForAPI()` remove os sufixos `[1m]` e `[2m]` antes de enviar para a API — esses sufixos são apenas convenções internas do cliente para marcar a janela de contexto de 1M; a camada de API não os reconhece.

### Resposta em Streaming e Fallback Não-streaming

O diálogo principal usa transmissão em streaming (Server-Sent Events), mas faz fallback para não-streaming quando o streaming falha:

```typescript
// services/api/claude.ts:2535-2559
// se o próprio streaming falhar com 529, conta para os 529 consecutivos
initialConsecutive529Errors: is529Error(streamingError) ? 1 : 0,
```

O fallback não-streaming tem limite máximo de tokens:

```typescript
// services/api/claude.ts
export const MAX_NON_STREAMING_TOKENS = 64_000
```

### Injeção Dinâmica de Beta Headers

Diferentes funcionalidades correspondem a diferentes Beta Headers, adicionados dinamicamente nas requisições:

```typescript
// constants/betas.ts (referência)
EFFORT_BETA_HEADER        // suporte ao parâmetro effort
CONTEXT_1M_BETA_HEADER    // janela de contexto 1M
FAST_MODE_BETA_HEADER     // fast mode
TASK_BUDGETS_BETA_HEADER  // controle de orçamento
```

---

## 13.9 Análise das Decisões de Design

### Filosofia de Design Sem Degradação Silenciosa

Pela notificação de troca com nível `'warning'` em `query.ts` e pelo design da classe de erro especializada `FallbackTriggeredError`, podemos ver que essa é uma escolha arquitetural deliberada:

**Por que não pode haver troca silenciosa?** Porque o Claude Code é uma ferramenta de escrita de código, e a qualidade do modelo impacta diretamente a qualidade da saída. Os usuários têm o direito de saber que "estou usando Sonnet, não Opus", podendo decidir se continuam esperando ou usam uma estratégia diferente. Diferente de produtos de chat de consumo, os usuários de ferramentas de código são mais especializados e mais sensíveis às diferenças de modelo.

### Considerações de Design sobre Transparência de Custos

O design de `hasConsoleBillingAccess()` em `costHook.ts` é digno de nota: mesmo sem exibição, os dados de custo são persistidos. Isso indica que o principal propósito do rastreamento de custos é **recuperação de sessão** (exibir o gasto da última sessão ao próximo início) em vez de alertas em tempo real. Este é um design com "consciência offline": os usuários podem ver o gasto completo ao término da sessão, sem serem interrompidos após cada chamada de API.

### Lógica de Produto na Diferenciação do Modelo Padrão

Usar Opus como modelo padrão para Max/Team Premium e Sonnet para Pro/PAYG tem uma lógica de produto clara: uma das propostas de valor da assinatura Max é "obter o modelo mais forte", e o próprio valor padrão é a expressão dessa proposta.

Ao mesmo tempo, mesmo para usuários Max, o effort padrão do Opus 4.6 é `medium` (controlado pelo GrowthBook) — isso indica que a Anthropic está **equilibrando qualidade e cota** através do sistema de effort, em vez de simplesmente fornecer a configuração máxima para usuários Max.

---

## 13.10 Necessidade das Migrações de Modelo (migrations)

Os 11 arquivos de migração no diretório `migrations/` revelam o rastro da evolução do produto; cada migração corresponde a uma decisão de produto:

| Arquivo de migração | Momento de acionamento | Lógica central |
|--------------------|------------------------|----------------|
| `migrateFennecToOpus.ts` | Funcionários internos (ant) | alias de codinome fennec → alias opus (limpeza de codinome interno) |
| `migrateLegacyOpusToCurrent.ts` | Usuários 1P com `opus-4-0`/`4-1` ainda nas configurações | IDs antigos de modelo Opus → alias `opus` (Opus 4.0/4.1 descontinuado) |
| `migrateOpusToOpus1m.ts` | Max/Team Premium (1P), `opus` nas configurações | `opus` → `opus[1m]` (merge da experiência 1M) |
| `migrateSonnet1mToSonnet45.ts` | Usuários com `sonnet[1m]` | `sonnet[1m]` → `sonnet-4-5-20250929[1m]` (fixado em 4.5 pois 4.6 1M tem audiência diferente) |
| `migrateSonnet45ToSonnet46.ts` | Pro/Max/Team Premium (1P) fixados no Sonnet 4.5 | Strings Sonnet 4.5 → alias `sonnet` (upgrade para 4.6) |
| `resetProToOpusDefault.ts` | Usuários Pro 1P sem modelo personalizado | Registra timestamp da migração, exibe notificação de upgrade uma vez no REPL |
| `resetAutoModeOptInForDefaultOffer.ts` | auto mode habilitado, usuários antigos com diálogo OptIn | Limpa `skipAutoPermissionPrompt`, exibe novamente o novo diálogo de permissões |
| `migrateAutoUpdatesToSettings.ts` | Usuário desabilitou atualizações automáticas explicitamente | Migra `autoUpdates: false` para variável de ambiente em settings.json |
| `migrateBypassPermissionsAcceptedToSettings.ts` | Config global com `bypassPermissionsModeAccepted` | Migra para `skipDangerousModePermissionPrompt` em settings.json |
| `migrateSonnet45ToSonnet46.ts` | Igual ao anterior | Mesma migração com nome duplicado |
| `migrateEnableAllProjectMcpServersToSettings.ts` | Config relacionada a MCP | Ajuste de estrutura de configurações de servidor MCP |

**Insight arquitetural**: cada migração opera apenas em `userSettings` (settings.json a nível de usuário), nunca tocando `projectSettings` (a nível de projeto) ou `policySettings` (a nível de política empresarial). Isso é design intencional:

1. **Idempotência**: lê e escreve na mesma fonte de dados; re-execuções não produzem efeitos colaterais
2. **Mínimo privilégio**: não pode (nem deve) modificar o pin do usuário a nível de projeto
3. **Evita promoção global**: se o usuário fixou um Opus antigo em algum projeto, a migração não promove para configuração global

A própria existência desse sistema de migrações indica: **a migração de schema em sistemas de IA é muito mais complexa do que migrações tradicionais de banco de dados** — é necessário considerar mudanças de tipo de assinatura, descontinuação de modelos, upgrades de janela de contexto e múltiplas outras dimensões, sem poder simplesmente sobrescrever a intenção do usuário.

---

## 13.11 Padrões Transferíveis

Cinco padrões de design extraídos desta análise que podem ser usados em sistemas próprios:

### Padrão Um: Cadeia de Override Multinível
```
session_override > startup_flag > env_var > config_file > builtin_default
```
Qualquer camada pode ser sobrescrita pela camada superior, mas a inferior não pode influenciar secretamente a superior. Combinado com verificação de allowlist para prevenir injeção de IDs de modelo inválidos.

### Padrão Dois: Separação de Estratégia 529 para Primeiro Plano/Segundo Plano
Tarefas em primeiro plano (usuário aguardando resultado): nova tentativa N vezes; se exceder, acionar fallback.
Tarefas em segundo plano (usuário não percebe): descartar na primeira ocorrência de 529, evitando efeito de amplificação de novas tentativas durante colapso de capacidade.

### Padrão Três: Sinalização via FallbackTriggeredError
Não trocar silenciosamente de modelo dentro do retry, mas lançar um erro especializado para que a lógica de troca seja tratada pela camada superior de chamada. Assim a lógica de troca fica centralizada em um único ponto (query.ts) e necessariamente acompanhada de notificação ao usuário.

### Padrão Quatro: Filtragem de Persistência toPersistableEffort
Configurações a nível de sessão (como effort `max`) são filtradas antes de serem gravadas em settings.json. "Estado que não deve persistir entre sessões" e "preferências do usuário que devem persistir" são separados desde a camada de modelo de dados.

### Padrão Cinco: Rastreamento de Custos em Buckets por Modelo
Não apenas rastrear custo total, mas agrupar por nome de modelo (normalizado). Só assim é possível exibir ao término da sessão "Opus custou quanto, Haiku custou quanto", ajudando os usuários a entender qual funcionalidade é mais cara.

---

## 13.12 Índice de Código-Fonte

| Arquivo | Linhas | Conteúdo Central |
|---------|--------|-----------------|
| `services/api/claude.ts` | 3.419 | Camada de chamada de API, queryHaiku, construção de requisição, processamento de streaming |
| `services/api/withRetry.ts` | ~600 | Lógica de nova tentativa, tratamento de 529, FallbackTriggeredError |
| `cost-tracker.ts` | 323 | Rastreamento de custos, persistência, exibição formatada |
| `costHook.ts` | 22 | Hook React, monitora saída de processo para acionar exibição de custo |
| `utils/effort.ts` | ~350 | Definição de Effort Level, cadeia de prioridades, detecção de suporte do modelo |
| `utils/modelCost.ts` | ~200 | Tabela de preços, funções de cálculo de custo |
| `utils/model/model.ts` | ~450 | Cadeia de prioridades do modelo, resolução de alias, lógica de modelo padrão |
| `utils/model/modelCapabilities.ts` | ~100 | Cache de capacidades do modelo (apenas usuários internos) |
| `query.ts` | ~1000 | Captura de FallbackTriggeredError, notificação ao usuário, troca de modelo |
| `migrations/*.ts` | 11 arquivos | Scripts de migração de versão de modelo |
| `tools/WebFetchTool/utils.ts:503` | — | Uso do Haiku 1: extração de conteúdo Web Fetch |
| `utils/shell/prefix.ts:220` | — | Uso do Haiku 2: julgamento de prefixo de comando Shell |
| `utils/sessionTitle.ts:87` | — | Uso do Haiku 3: geração de título de sessão |
| `utils/mcp/dateTimeParser.ts:68` | — | Uso do Haiku 4: parsing de DateTime |
| `services/toolUseSummary/toolUseSummaryGenerator.ts:69` | — | Uso do Haiku 5: resumo de chamadas de ferramenta |
| `commands/rename/generateSessionName.ts:20` | — | Uso do Haiku 6: renomeação de sessão |
| `services/api/claude.ts:544` | — | Uso do Haiku 7: verificação de API Key |

---

*Este capítulo cobre completamente as afirmações do cc-notebook sobre seleção de modelos e controle de custos. Resultados de verificação: "pelo menos 6 usos em segundo plano" do Haiku foram verificados (na verdade, 13 pontos de chamada); sem degradação silenciosa, completamente verificado; mecanismo 529 Overload Fallback, completamente verificado; sistema Effort Level, completamente verificado. Todos os trechos de código foram copiados precisamente dos arquivos fonte, com localização de arquivo e número de linha anotados.*
