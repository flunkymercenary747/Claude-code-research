# Capítulo 3: Sistema de Ferramentas

## 3.1 Visão Geral e Posicionamento

O sistema de ferramentas do Claude Code é a camada de execução de todo o produto. O LLM é responsável por raciocínio e tomada de decisão, mas os efeitos colaterais reais — ler arquivos, executar comandos, pesquisar código, acessar a rede — são todos realizados pelo sistema de ferramentas. O sistema de ferramentas é o único canal entre a intenção do LLM e o mundo real.

Em termos de escala, este é um subsistema bastante robusto:
- O diretório de ferramentas no snapshot do código-fonte conta com **40+ subdiretórios**, abrangendo categorias como operações de arquivo, execução de código, coordenação de Agents e integração MCP
- O arquivo de abstração central `Tool.ts` tem 792 linhas; o arquivo de registro `tools.ts` tem 389 linhas; e o motor de execução de ferramentas `services/tools/toolExecution.ts` tem 1.745 linhas
- O módulo de armazenamento de resultados de ferramentas `utils/toolResultStorage.ts` tem 1.040 linhas, tratando independentemente questões de orçamento de tokens

Essa escala revela um fato: **o sistema de ferramentas não é um acessório do Claude Code, mas seu ativo central de engenharia**. A confiabilidade, segurança e extensibilidade de todo o produto são amplamente determinadas pela qualidade do design do sistema de ferramentas.

A análise competitiva (cc-notebook) não tem um capítulo independente sobre o sistema de ferramentas — uma lacuna de análise evidente que este capítulo preenche.

---

## 3.2 Fundamentos Teóricos

### Padrão Self-Describing Tools (Ferramentas Auto-Descritivas)

Nas chamadas de API tradicionais, o chamador precisa conhecer antecipadamente as especificações da interface. O sistema de ferramentas do Claude Code adota uma filosofia de design diferente: **cada ferramenta se auto-descreve em termos de capacidades, formato de entrada e restrições de uso**.

Isso se reflete em vários campos centrais do tipo `Tool`:

```typescript
// Tool.ts:300-310 (simplificado)
export type Tool<Input, Output, P> = {
  name: string
  searchHint?: string          // resumo de capacidade de 3-10 palavras para correspondência de palavras-chave pelo ToolSearch
  description(input, options): Promise<string>   // geração dinâmica de descrição
  prompt(options): Promise<string>               // prompt de sistema completo da ferramenta
  inputSchema: Input           // Zod schema; serve tanto como documentação quanto como validador
  outputSchema?: z.ZodType
  // ...
}
```

`description()` e `prompt()` são métodos assíncronos, o que significa que a auto-descrição da ferramenta pode ser **gerada dinamicamente** — ajustando o conteúdo do prompt com base no contexto de permissão atual, nas ferramentas instaladas e no estado do ambiente. Não é documentação estática, mas descrições geradas em tempo de execução com consciência de contexto.

### Arquitetura de Plugins e Injeção de Dependência

O sistema de ferramentas é essencialmente uma arquitetura de plugins. Cada ferramenta é construída pela função fábrica `buildTool()`, implementando a interface `Tool` unificada, mas completamente desacoplada entre si. Adicionar uma nova ferramenta requer apenas:

1. Criar um diretório de ferramenta (como `tools/MyTool/`)
2. Implementar a interface `ToolDef`
3. Registrar em `getAllBaseTools()` em `tools.ts`

As ferramentas em si não dependem umas das outras (dependências circulares são quebradas via lazy require), mas todas dependem do `ToolUseContext` — um objeto de contexto que permeia toda a cadeia de execução, contendo estado de permissões, histórico de mensagens, estado da aplicação etc.

```typescript
// Tool.ts:167-172 (simplificado)
export type ToolUseContext = {
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    // ...
  }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  contentReplacementState?: ContentReplacementState
  // ...
}
```

O design do `ToolUseContext` é injeção de dependência típica: todas as dependências externas necessárias para a execução de ferramentas são passadas via context; as ferramentas em si são componentes funcionais sem estado. Isso torna testes, isolamento e execução de subagentes viáveis.

### O Papel do Function Calling em LLMs

O Claude Code segue o protocolo de Function Calling da API Anthropic. O LLM pode gerar blocos `tool_use` durante o raciocínio, especificando o nome da ferramenta e os parâmetros; os resultados da execução são retornados ao LLM como blocos `tool_result`, servindo como entrada para a próxima rodada de raciocínio.

A restrição-chave desse ciclo é: **as definições de ferramentas (nome + schema de entrada) devem ser enviadas ao LLM no prompt de sistema**, consumindo tokens preciosos do contexto. Quando o número de ferramentas aumenta para 40+, mais ferramentas MCP de terceiros, esse custo torna-se significativo — o que diretamente impulsiona o mecanismo de carregamento lazy ToolSearch descrito na seção 3.6.

---

## 3.3 Arquitetura e Estruturas de Dados

### Abstração Unificada buildTool()

`buildTool()` é a função fábrica central do sistema de ferramentas, definida em `Tool.ts:756-769`:

```typescript
// Tool.ts:756-769
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

Ela faz uma coisa: mescla o `ToolDef` fornecido pelo usuário (que pode omitir campos opcionais) com `TOOL_DEFAULTS` (valores padrão seguros) e retorna um `Tool` completo.

Os valores padrão (`Tool.ts:729-742`) refletem a filosofia de design **fail-closed** (falha segura):

```typescript
// Tool.ts:729-742
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?) => false,    // padrão: não seguro para concorrência
  isReadOnly: (_input?) => false,            // padrão: assume que haverá escrita
  isDestructive: (_input?) => false,
  checkPermissions: (input, _ctx?) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // padrão: permite; o sistema de permissões genérico cuida disso
  toAutoClassifierInput: (_input?) => '',    // padrão: ignora o classificador de segurança
  userFacingName: (_input?) => '',
}
```

Vale notar que `isConcurrencySafe` tem padrão `false` — o sistema prefere executar duas ferramentas sequencialmente a arriscar executá-las em paralelo com possíveis efeitos colaterais. Apenas ferramentas que declaram explicitamente `isConcurrencySafe: () => true` (como GrepTool, GlobTool e outras ferramentas somente leitura) são escalonadas em paralelo.

### Definição do Tipo Central de Ferramentas

Os métodos da interface `Tool` podem ser divididos em vários domínios funcionais (`Tool.ts:297-580`):

**Domínio de Execução**
- `call(args, context, canUseTool, parentMessage, onProgress)` — método de execução central, retorna `Promise<ToolResult<Output>>`
- `validateInput(input, context)` — validação pré-execução, retorna `ValidationResult`
- `checkPermissions(input, context)` — verificação de permissão, independente do sistema de permissões genérico

**Domínio de Descrição** (capacidade de auto-descrição de ferramentas)
- `description(input, options)` — descrição curta, para a lista de tools da API
- `prompt(options)` — prompt de sistema completo, diz ao modelo como usar esta ferramenta
- `searchHint` — resumo de capacidade de 3-10 palavras, exclusivamente para correspondência de palavras-chave pelo ToolSearch

**Domínio de Renderização** (componentes React, apenas no modo REPL)
- `renderToolUseMessage(input, options)` — UI quando a chamada de ferramenta começa
- `renderToolResultMessage(content, progressMessages, options)` — UI do resultado da ferramenta
- `renderToolUseProgressMessage(progressMessages, options)` — UI de progresso durante a execução
- `renderToolUseRejectedMessage(input, options)` — UI quando rejeitada

**Domínio de Metadados**
- `isConcurrencySafe(input)` — declara se pode ser executada de forma concorrente
- `isReadOnly(input)` — declara se é somente leitura (afeta julgamentos de permissão)
- `isDestructive(input)` — declara se é irreversível (exclusão, sobrescrita, envio)
- `shouldDefer` — se deve ser carregada de forma lazy (carregada sob demanda pelo ToolSearch)
- `alwaysLoad` — sempre carregada no prompt (sem lazy loading)
- `maxResultSizeChars` — limiar para persistência em disco do resultado da ferramenta

A estrutura `ToolResult<T>` (`Tool.ts:289-298`) também merece atenção:

```typescript
// Tool.ts:289-298
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

`contextModifier` permite que ferramentas modifiquem o contexto após a execução (mas os comentários deixam claro: **apenas ferramentas que não são seguras para concorrência executam contextModifier** — esta é uma restrição importante de segurança de concorrência).

### Mecanismo de Registro e Descoberta de Ferramentas

`getAllBaseTools()` em `tools.ts` é a única fonte de verdade para o registro de ferramentas (`tools.ts:108-186`). Esta função retorna todas as ferramentas integradas disponíveis no ambiente atual, com disponibilidade controlada por múltiplas condições em camadas:

**Condições de ambiente** (process.env):
- `USER_TYPE === 'ant'` — ferramentas internas da Anthropic (ConfigTool, TungstenTool, REPLTool)
- `NODE_ENV === 'test'` — ferramentas de teste (TestingPermissionTool)
- `ENABLE_LSP_TOOL` — ferramenta de integração LSP
- `CLAUDE_CODE_VERIFY_PLAN` — ferramenta de verificação de planos

**Condições de Feature Flag** (`feature()` do `bun:bundle`):
- `PROACTIVE` / `KAIROS` — SleepTool (comportamento proativo)
- `AGENT_TRIGGERS` — ScheduleCronTool e outras ferramentas de agendamento
- `COORDINATOR_MODE` — ferramentas relacionadas ao modo coordenador
- `WEB_BROWSER_TOOL` — ferramenta de navegador
- `WORKFLOW_SCRIPTS` — ferramentas de fluxo de trabalho

**Condições de tempo de execução**:
- `isToolSearchEnabledOptimistic()` — se deve incluir ToolSearchTool
- `isTodoV2Enabled()` — se deve incluir o conjunto de ferramentas de gerenciamento de tarefas
- `isAgentSwarmsEnabled()` — se deve incluir ferramentas de colaboração em equipe
- `hasEmbeddedSearchTools()` — se bfs/ugrep estiverem integrados, não inclui GlobTool/GrepTool

**Deduplicação e ordenação** de ferramentas (`assembleToolPool()`, `tools.ts:218-248`) usa uma estratégia cuidadosamente projetada: ferramentas integradas e ferramentas MCP são classificadas separadamente e concatenadas, com ferramentas integradas como prefixo e ferramentas MCP anexadas ao final. Isso é para manter a estabilidade do prompt de sistema (prompt cache stability) — o servidor Anthropic define breakpoints de cache em posições fixas; se ferramentas integradas e MCP forem misturadas, qualquer nova ferramenta MCP adicionada quebraria o cache.

---

## 3.4 Lista Completa de Ferramentas por Categoria

Com base na estrutura do diretório `tools/` e na lógica de registro em `tools.ts`, pode-se organizar a lista completa de ferramentas:

### Operações de Arquivo (File Operations)

| Ferramenta | Diretório | Funcionalidade | Segurança para concorrência |
|------|------|------|---------|
| FileReadTool | `FileReadTool/` | Lê arquivos; suporta PDF/imagens/Notebooks, leitura paginada | Sim |
| FileEditTool | `FileEditTool/` | Substituição precisa de strings; suporta replace_all | Não |
| FileWriteTool | `FileWriteTool/` | Escreve/cria arquivos | Não |
| GlobTool | `GlobTool/` | Localiza arquivos por padrão glob | Sim |
| GrepTool | `GrepTool/` | Pesquisa de conteúdo regex com ripgrep | Sim |
| NotebookEditTool | `NotebookEditTool/` | Edição de células de Jupyter Notebook | Não |

### Execução de Código (Execution)

| Ferramenta | Diretório | Funcionalidade | Observações |
|------|------|------|------|
| BashTool | `BashTool/` | Execução de comandos shell; suporta tarefas em background e sandbox | Ferramenta central |
| PowerShellTool | `PowerShellTool/` | Execução de Windows PowerShell | Habilitada condicionalmente |
| REPLTool | `REPLTool/` | Execução REPL em ambiente VM isolado | Interno Ant |

### Coordenação de Agentes (Agent Orchestration)

| Ferramenta | Diretório | Funcionalidade |
|------|------|------|
| AgentTool | `AgentTool/` | Inicia subagentes; suporta execução paralela |
| SendMessageTool | `SendMessageTool/` | Envia mensagens a outros Agents |
| TeamCreateTool | `TeamCreateTool/` | Cria equipes de Agents |
| TeamDeleteTool | `TeamDeleteTool/` | Remove equipes de Agents |
| TaskCreateTool | `TaskCreateTool/` | Cria tarefas em background |
| TaskGetTool | `TaskGetTool/` | Obtém status de tarefas |
| TaskUpdateTool | `TaskUpdateTool/` | Atualiza status de tarefas |
| TaskListTool | `TaskListTool/` | Lista todas as tarefas |
| TaskStopTool | `TaskStopTool/` | Para tarefas |
| TaskOutputTool | `TaskOutputTool/` | Obtém saída de tarefas |

### Contexto e Descoberta (Context & Discovery)

| Ferramenta | Diretório | Funcionalidade |
|------|------|------|
| SkillTool | `SkillTool/` | Carrega e executa Skills (~/.claude/skills/) |
| ToolSearchTool | `ToolSearchTool/` | Pesquisa ferramentas com carregamento lazy |
| MCPTool (gerada dinamicamente) | `MCPTool/` | Ferramentas de servidor MCP (registradas dinamicamente em tempo de execução) |
| ListMcpResourcesTool | `ListMcpResourcesTool/` | Lista recursos MCP |
| ReadMcpResourceTool | `ReadMcpResourceTool/` | Lê recursos MCP |
| LSPTool | `LSPTool/` | Integração com servidor de linguagem LSP |

### Planejamento e Estado (Planning & State)

| Ferramenta | Diretório | Funcionalidade |
|------|------|------|
| EnterPlanModeTool | `EnterPlanModeTool/` | Entra no modo de planejamento (somente leitura, sem execução) |
| ExitPlanModeTool | `ExitPlanModeTool/` | Sai do modo de planejamento |
| EnterWorktreeTool | `EnterWorktreeTool/` | Entra em ambiente git worktree isolado |
| ExitWorktreeTool | `ExitWorktreeTool/` | Sai do ambiente worktree |
| TodoWriteTool | `TodoWriteTool/` | Escreve lista de Todo (exibida na barra lateral) |
| BriefTool | `BriefTool/` | Gera resumo da sessão |

### Acesso à Rede (Network)

| Ferramenta | Diretório | Funcionalidade |
|------|------|------|
| WebFetchTool | `WebFetchTool/` | Busca HTTP, conversão HTML→Markdown, verificação de segurança de domínio |
| WebSearchTool | `WebSearchTool/` | Pesquisa na web |

### Sistema e Agendamento (System & Scheduling)

| Ferramenta | Diretório | Funcionalidade | Condição |
|------|------|------|------|
| ConfigTool | `ConfigTool/` | Lê/escreve configurações | Interno Ant |
| SleepTool | `SleepTool/` | Aguarda (modo proativo) | PROACTIVE/KAIROS |
| SyntheticOutputTool | `SyntheticOutputTool/` | Saída sintética (uso especial) | — |
| ScheduleCronTool | `ScheduleCronTool/` | Cria/remove/lista tarefas agendadas | AGENT_TRIGGERS |
| RemoteTriggerTool | `RemoteTriggerTool/` | Gatilho remoto | AGENT_TRIGGERS_REMOTE |
| AskUserQuestionTool | `AskUserQuestionTool/` | Pergunta ao usuário (interativo) | — |

---

## 3.5 Fluxo de Execução de Ferramentas

### Fluxo Completo de LLM tool_use até a Execução de Ferramentas

O ponto de entrada da execução de ferramentas é a função `runToolUse()` em `services/tools/toolExecution.ts` (`toolExecution.ts:298-428`), que é um async generator:

```
LLM gera bloco tool_use
    ↓
runToolUse(toolUse, assistantMessage, canUseTool, context)
    ↓
findToolByName() — encontra a ferramenta; suporta aliases (compatibilidade retroativa com ferramentas renomeadas)
    ↓
abortController.signal.aborted? → retorna CANCEL_MESSAGE
    ↓
streamedCheckPermissionsAndCallTool() [retorna AsyncIterable]
    ↓
checkPermissionsAndCallTool()
  1. tool.inputSchema.safeParse(input)   — validação de tipos Zod
  2. tool.validateInput(input, context)  — validação personalizada da ferramenta
  3. runPreToolUseHooks()                — executa hooks PreToolUse
  4. canUseTool()                        — verificação de permissão (pode exibir UI de confirmação)
  5. tool.call(input, context, canUseTool, parentMessage, onProgress)
  6. processToolResultBlock()            — persiste resultados grandes
  7. runPostToolUseHooks()               — executa hooks PostToolUse
    ↓
yield MessageUpdateLazy (contendo tool_result)
    ↓
próxima rodada de raciocínio do LLM
```

Um design importante de compatibilidade retroativa (`toolExecution.ts:350-360`): quando uma ferramenta é renomeada, o nome antigo é mantido como `aliases`. Quando a ferramenta não é encontrada em `options.tools`, o sistema ainda pesquisa `getAllBaseTools()` por correspondências de aliases — garantindo que nomes de ferramentas antigas em transcrições históricas ainda possam ser executados.

### Execução de Ferramentas em Streaming (Streaming Tool Execution)

A execução de ferramentas é transmitida via `Stream<MessageUpdateLazy>` (`toolExecution.ts:500-535`):

```typescript
// toolExecution.ts:500-535 (simplificado)
function streamedCheckPermissionsAndCallTool(...): AsyncIterable<MessageUpdateLazy> {
  const stream = new Stream<MessageUpdateLazy>()
  checkPermissionsAndCallTool(
    ...,
    progress => {
      stream.enqueue({ message: createProgressMessage({...}) })  // mensagens de progresso
    },
  )
    .then(results => {
      for (const result of results) stream.enqueue(result)       // resultado final
    })
    .catch(error => stream.error(error))
    .finally(() => stream.done())
  return stream
}
```

O significado do design em streaming: a UI pode exibir progresso em tempo real enquanto a ferramenta ainda está executando (por exemplo, saída em tempo real do BashTool, progresso de subagentes no AgentTool). Mensagens de progresso e resultado final são passados pelo mesmo pipe `Stream`, simplificando o código do consumidor.

### Execução Concorrente de Ferramentas

O Claude Code suporta o LLM gerando múltiplos blocos `tool_use` em uma única resposta para execução paralela. O pré-requisito para concorrência é: **todas as ferramentas devem declarar `isConcurrencySafe: () => true`**.

Durante execução paralela, `contextModifier` não é executado (como diz o comentário em `ToolResult`: "contextModifier is only honored for tools that aren't concurrency safe"). Esta é uma restrição importante de segurança: operações que modificam o contexto global não podem ser realizadas em ambientes concorrentes.

Ferramentas tipicamente seguras para concorrência: GrepTool, GlobTool, FileReadTool (todas declarando `isConcurrencySafe: () => true`).

---

## 3.6 ToolSearch — Mecanismo de Carregamento Lazy

### Por que o ToolSearch é Necessário (Problema de Inflação do Prompt)

A definição de cada ferramenta (nome + JSON Schema + descrição) ocupa tokens quando enviada ao LLM. Quando o número de ferramentas excede um certo limiar (experimentos mostram cerca de 40-60 ferramentas), os problemas são:

1. **Aumento do custo de tokens**: cada chamada de API carrega grandes definições de ferramentas
2. **Diluição da atenção**: o LLM enfrentando dezenas de ferramentas pode reduzir a atenção dedicada a cada uma
3. **Risco de invalidação do Prompt Cache**: mudanças na lista de ferramentas (como a adição dinâmica de ferramentas MCP) invalidam o cache

A solução do ToolSearch é **carregamento sob demanda**: a maioria das ferramentas é marcada com `shouldDefer: true`, não enviando o schema completo no prompt inicial; apenas quando descobertas via pesquisa são carregadas.

### Registro e Descoberta de Ferramentas Adiadas

Ferramentas declaram se devem ser carregadas de forma lazy via o campo `shouldDefer` (`Tool.ts:456-462`):

```typescript
// Tool.ts:456-462
readonly shouldDefer?: boolean

/**
 * When true, this tool is never deferred — its full schema appears in the
 * initial prompt even when ToolSearch is enabled. For MCP tools, set via
 * `_meta['anthropic/alwaysLoad']`. Use for tools the model must see on
 * turn 1 without a ToolSearch round-trip.
 */
readonly alwaysLoad?: boolean
```

A função `isDeferredTool()` (definida em `tools/ToolSearchTool/prompt.ts`) determina se uma ferramenta deve ser adiada: ferramentas com `shouldDefer: true` mas sem `alwaysLoad: true` são marcadas como adiadas.

O ToolSearchTool em si **nunca é adiado** — deve estar disponível desde a primeira rodada; caso contrário, nenhuma outra ferramenta pode ser descoberta.

### Implementação do Carregamento sob Demanda

O método `call()` do ToolSearchTool (`ToolSearchTool.ts:221-302`) suporta dois modos de consulta:

**Modo de seleção direta** (prefixo `select:`):
```
query: "select:NotebookEdit"     → retorna diretamente NotebookEditTool
query: "select:Read,Edit,Grep"   → seleciona múltiplas ferramentas em lote
```

**Modo de pesquisa por palavras-chave**:
```
query: "jupyter notebook"        → correspondência de palavras-chave, retorna NotebookEditTool etc.
query: "mcp__github"             → correspondência de prefixo de servidor MCP
```

Algoritmo de pontuação de pesquisa (`ToolSearchTool.ts:155-198`):

```
Correspondência exata de parte do nome da ferramenta (MCP): +12 pontos
Correspondência exata de parte do nome da ferramenta (normal): +10 pontos
Nome da ferramenta contém parcialmente palavra-chave (MCP): +6 pontos
Nome da ferramenta contém parcialmente palavra-chave (normal): +5 pontos
Nome da ferramenta com correspondência completa (fallback): +3 pontos
Correspondência de palavra no searchHint: +4 pontos (resumo de capacidade cuidadosamente elaborado, sinal forte)
Correspondência de palavra no texto de descrição: +2 pontos
```

O peso do campo `searchHint` (+4 pontos) é maior que o do texto de descrição (+2 pontos), incentivando desenvolvedores de ferramentas a fornecer resumos precisos de capacidades. Por exemplo, `searchHint` do GrepTool: `'search file contents with regex (ripgrep)'`; `searchHint` do FileEditTool: `'modify file contents in place'`.

Os resultados da pesquisa são retornados ao LLM via blocos `tool_reference` (`ToolSearchTool.ts:330-352`), que é uma extensão especial da API Anthropic, informando ao servidor "injete os schemas completos dessas ferramentas na lista de ferramentas da conversa atual".

---

## 3.7 Armazenamento de Resultados de Ferramentas

### Estratégia de Armazenamento em Disco

Os resultados da execução de ferramentas podem ser muito grandes (como ler um arquivo de log de 10MB ou executar um comando que gera grande volume de saída). Colocar resultados grandes diretamente no histórico de mensagens desperdiça tokens e infla o contexto das requisições subsequentes.

`utils/toolResultStorage.ts` implementa a estratégia de **persistência sob demanda**:

1. Calcula o tamanho do resultado (`contentSize()`)
2. Compara com o limiar `maxResultSizeChars` da ferramenta (resolvido via `getPersistenceThreshold()`)
3. Resultados que excedem o limiar são gravados em `~/.claude/projects/<project>/<session>/tool-results/<tool_use_id>.txt`
4. Substituídos por uma mensagem contendo caminho do arquivo + prévia

```typescript
// toolResultStorage.ts:168-177
export function buildLargeToolResultMessage(result: PersistedToolResult): string {
  let message = `${PERSISTED_OUTPUT_TAG}\n`
  message += `Output too large (${formatFileSize(result.originalSize)}). Full output saved to: ${result.filepath}\n\n`
  message += `Preview (first ${formatFileSize(PREVIEW_SIZE_BYTES)}):\n`
  message += result.preview
  message += result.hasMore ? '\n...\n' : '\n'
  message += PERSISTED_OUTPUT_CLOSING_TAG
  return message
}
```

`PREVIEW_SIZE_BYTES = 2000` (cerca de 2KB); a prévia é cortada na última quebra de linha, evitando truncar o meio de uma linha.

Um design de **idempotência** importante (`toolResultStorage.ts:145-158`): ao escrever, usa `{ flag: 'wx' }` (criação exclusiva); se o arquivo já existir, o erro de escrita é ignorado e o arquivo existente é usado para gerar a prévia. Isso garante que a reprodução de mensagens históricas durante microcompactação não grava novamente nem gera erros EEXIST.

O FileReadTool tem um tratamento especial: `maxResultSizeChars: Infinity` — os resultados da leitura nunca são persistidos em disco. O motivo está nos comentários: "persisting creates a circular Read→file→Read loop and the tool already self-bounds via its own limits" (persistir criaria um loop circular: Read lê arquivo, resultado muito grande é persistido como arquivo, modelo usa Read para ler aquele arquivo...).

### Gerenciamento de Orçamento de Tokens

`toolResultStorage.ts` também implementa um **orçamento de resultados de ferramentas por mensagem** mais abrangente. É acionado pelo mecanismo `ContentReplacementState` (`toolResultStorage.ts:395-440`):

```typescript
// toolResultStorage.ts:395-413
export type ContentReplacementState = {
  seenIds: Set<string>        // tool_use_ids que já passaram pela verificação de orçamento (resultados congelados)
  replacements: Map<string, string>  // IDs substituídos → conteúdo da string substituída
}
```

Restrição central: **uma vez que um resultado é julgado (substituído ou não), nunca muda** (garantido pelo conjunto `seenIds`). Isso é para estabilidade do Prompt Cache — o tratamento do mesmo `tool_use_id` deve permanecer consistente em toda a sessão; caso contrário, o cache seria invalidado por mudanças de conteúdo.

O limite do orçamento é controlado dinamicamente pelo feature flag do GrowthBook `tengu_hawthorn_window`; quando o volume total de resultados de ferramentas em uma mensagem excede o limite, o sistema substitui os maiores resultados de ferramentas pela versão persistida em disco, até que o volume total fique dentro do orçamento.

---

## 3.8 Análise das Decisões de Design

### Tradeoff entre Auto-Descrição vs. Registro Externo

O Claude Code escolheu o modelo de **auto-descrição** (cada ferramenta carrega seu próprio schema, descrição, prompt e lógica de renderização), em vez de centralizar essas informações em um registro central.

Vantagens:
- **Ferramentas completamente auto-contidas**: adicionar uma nova ferramenta requer apenas um diretório; não é necessário modificar a lógica do registro central
- **Descrições podem ser geradas dinamicamente**: `description()` e `prompt()` são funções assíncronas; podem ajustar o conteúdo dinamicamente com base no ambiente, permissões e estado de instalação
- **Lógica de renderização coexiste com a ferramenta**: componentes de renderização React ficam diretamente nos arquivos da ferramenta; alterar o comportamento da ferramenta e alterar a UI é o mesmo PR

Desvantagens:
- **Interface da ferramenta inflada**: o tipo `Tool` tem 40+ métodos/campos; autores de novas ferramentas precisam conhecer muitos detalhes da interface
- **Código duplicado**: cada ferramenta tem métodos de renderização como `renderToolUseMessage`, `renderToolResultMessage` etc., com padrões altamente similares
- **`buildTool()` não elimina completamente**: fornece valores padrão, mas muitos métodos ainda precisam ser implementados por cada ferramenta

Na prática, o Claude Code mitiga a duplicação de código com **componentes de UI compartilhados** (como `tools/shared/`) e **extração de padrões** (como `lazySchema()`), mas a complexidade fundamental da interface permanece.

### Por que Algumas Ferramentas São Carregadas de Forma Lazy

A decisão de carregamento lazy do ToolSearch segue um princípio: **ferramentas que provavelmente não são necessárias na primeira rodada de conversa devem ser adiadas**.

Ferramentas com `alwaysLoad` (nunca adiadas) devem satisfazer: o modelo precisa saber de sua existência na primeira rodada. Exemplos típicos: AgentTool, BashTool, FileReadTool — ferramentas fundamentais para qualquer tarefa de programação.

Ferramentas com `shouldDefer` (carregamento lazy) normalmente são: ferramentas necessárias apenas em cenários específicos (NotebookEditTool só é necessário para tarefas Jupyter); grandes volumes de ferramentas MCP (o usuário instalou dezenas de servidores MCP, mas usa apenas alguns em cada conversa).

Ferramentas MCP por padrão acionam o mecanismo ToolSearch com base no número de ferramentas, mas podem ser forçadas a não serem adiadas definindo `_meta['anthropic/alwaysLoad']` nos metadados da ferramenta.

### Design de Permissões em Três Camadas das Ferramentas

As permissões de ferramentas usam um design de **três camadas de defesa**:

1. **Validação de tipos Zod** (primeiro passo de `checkPermissionsAndCallTool`): o `inputSchema` da ferramenta valida rigorosamente os tipos de parâmetros; parâmetros de tipo errado gerados pelo LLM são rejeitados com mensagem de erro
2. **Validação personalizada da ferramenta** (`validateInput()`): a ferramenta implementa sua própria validação de lógica de negócio; por exemplo, o FileEditTool verifica se old_string e new_string são diferentes, verifica se o tamanho do arquivo excede 1GiB
3. **Sistema de permissões genérico** (`canUseTool()` + `checkPermissions()`): faz o julgamento final com base em regras allow/deny configuradas pelo usuário, se a ferramenta é somente leitura, se é uma operação destrutiva etc.; pode exibir confirmação interativa

Essas três camadas são executadas sequencialmente; qualquer falha em uma camada causa um curto-circuito, sem entrar na próxima.

---

## 3.9 Padrões Transferíveis

### Design Geral do Padrão Self-Describing Tools

O padrão de maior valor de migração extraído do sistema de ferramentas do Claude Code é: **ferramentas como plugins auto-contidos**.

Princípios centrais:
1. **Schema como documentação como validador**: use Zod schema para definir entradas, gerando automaticamente JSON Schema para o LLM, enquanto valida a saída do LLM em tempo de execução
2. **Função fábrica + valores padrão seguros**: `buildTool()` fornece comportamento padrão seguro (padrão não seguro para concorrência, padrão somente leitura falso); desenvolvedores de ferramentas só precisam declarar suas exceções
3. **searchHint de resumo conciso**: descrição de capacidade de 3-10 palavras, otimizada especificamente para pesquisa por palavras-chave, separada da descrição completa
4. **Declaração de capacidade em vez de julgamento em tempo de execução**: `isReadOnly()`, `isConcurrencySafe()`, `isDestructive()` permitem que o escalonador tome decisões sem executar a ferramenta

### O Que o Sistema de Ferramentas (Sistema de Bricks) do Doramagic Pode Aprender

O sistema de Bricks do Doramagic (278+ blocos) tem semelhanças profundas com o sistema de ferramentas do Claude Code, mas também tem diferenças essenciais:

**Semelhanças**:
- Ambas são arquiteturas de "plugin": cada Brick/Tool é uma unidade funcional auto-contida
- Ambas precisam de mecanismos de descrição: informar ao LLM quando usar qual ferramenta/bloco
- Ambas têm sistemas de classificação: organizadas por domínio funcional

**Padrões específicos que podem ser aprendidos**:

1. **`searchHint` análogo aos tags dos Bricks**: o Claude Code fornece descrições de capacidade concisas de 3-10 palavras para cada ferramenta, especificamente para correspondência de pesquisa. Os Bricks do Doramagic atualmente usam tags e categorias; pode-se adicionar um campo `hint`, otimizando especificamente a eficiência de descoberta de blocos pelo modelo.

2. **Carregamento lazy → ativação sob demanda de blocos**: o mecanismo de ferramentas adiadas do Claude Code mostra que encher o prompt de sistema com todos os blocos não é uma boa ideia. O Doramagic pode referenciar o design `shouldDefer`, tornando blocos pouco usados (blocos de domínio especializado) lazy, ativando-os apenas quando o modelo os necessita explicitamente.

3. **`maxResultSizeChars` → orçamento de saída dos blocos**: cada ferramenta declara seu próprio orçamento máximo de tokens para resultados; quando excedido, é comprimido. As saídas de blocos do Doramagic (JSON de conhecimento extraído) também podem ser grandes; referencie este mecanismo para implementar a estratégia "resumo primeiro, detalhes sob demanda".

4. **`isConcurrencySafe` → declaração paralela dos blocos**: no pipeline de extração de conhecimento do Doramagic, múltiplos blocos podem atuar simultaneamente no mesmo codebase. Declarar explicitamente a segurança de concorrência dos blocos permite que o escalonador decida automaticamente quais blocos podem ser executados em paralelo e quais precisam ser serializados.

5. **Defesa em três camadas de permissões → segurança de execução de blocos**: para cenários onde o Doramagic é executado como uma OpenClaw Skill, a validação de legitimidade de execução de Bricks pode referenciar este design de três camadas: validação de schema → validação de negócio → verificação de permissão da plataforma.

**Diferença essencial**: as ferramentas do Claude Code são principalmente voltadas a **operações determinísticas** (ler arquivos, executar comandos); a saída é precisamente definível. Os Bricks do Doramagic são voltados a **extração de conhecimento**; a saída é semântica. Isso significa que o Doramagic não pode usar Zod schema para validar estritamente a saída de Bricks como o Claude Code — este é exatamente o significado do princípio arquitetural "código diz fatos, IA conta histórias" do Doramagic: a estrutura determinística (extração de facts) pode ser restringida por schema; a interpretação não determinística (geração de stories) não precisa.

---

## 3.10 Índice do Código-Fonte

| Arquivo | Linhas | Conteúdo-chave |
|------|------|---------|
| `src/Tool.ts` | 792 | Definição do tipo `Tool`, `buildTool()`, `ToolUseContext`, `ToolResult`, `TOOL_DEFAULTS` |
| `src/tools.ts` | 389 | `getAllBaseTools()`, `getTools()`, `assembleToolPool()`, `getMergedTools()`, `filterToolsByDenyRules()` |
| `src/services/tools/toolExecution.ts` | 1.745 | `runToolUse()`, `checkPermissionsAndCallTool()`, `streamedCheckPermissionsAndCallTool()`, `buildSchemaNotSentHint()` |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 471 | `searchToolsWithKeywords()`, `parseToolName()`, algoritmo de pontuação de palavras-chave, seleção direta com prefixo `select:` |
| `src/utils/toolResultStorage.ts` | 1.040 | `persistToolResult()`, `buildLargeToolResultMessage()`, `ContentReplacementState`, `enforceToolResultBudget()` |
| `src/tools/BashTool/BashTool.tsx` | ~1.800+ | `isSearchOrReadBashCommand()`, sandbox, tarefas em background, exibição de progresso |
| `src/tools/FileEditTool/FileEditTool.ts` | ~500+ | Substituição de strings, proteção de arquivos grandes (limite de 1GiB), detecção de segredos |
| `src/tools/FileReadTool/FileReadTool.ts` | ~600+ | Suporte a múltiplos formatos (PDF/imagens/Notebooks), contagem de tokens, `maxResultSizeChars: Infinity` |
| `src/tools/GrepTool/GrepTool.ts` | ~400+ | Integração ripgrep, paginação com head_limit/offset, `DEFAULT_HEAD_LIMIT = 250` |
| `src/tools/WebFetchTool/utils.ts` | ~450+ | Verificação de lista negra de domínios, cache LRU (50MB/15min), conversão HTML→Markdown |
| `src/tools/MCPTool/classifyForCollapse.ts` | ~350 | Classificação search/read de ferramentas MCP (regras pré-definidas para 20+ provedores como Slack/GitHub/Linear/Jira) |

**Constantes importantes** (espalhadas em múltiplos arquivos):
- `PREVIEW_SIZE_BYTES = 2000` (toolResultStorage.ts) — tamanho da prévia para resultados grandes
- `DEFAULT_HEAD_LIMIT = 250` (GrepTool.ts) — limite padrão de resultados grep
- `MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024` (WebFetchTool/utils.ts) — limite de 10MB para busca na rede
- `FETCH_TIMEOUT_MS = 60_000` (WebFetchTool/utils.ts) — timeout de 60 segundos para requisições HTTP
- `CACHE_TTL_MS = 15 * 60 * 1000` (WebFetchTool/utils.ts) — cache de URL por 15 minutos
- `PROGRESS_THRESHOLD_MS = 2000` (BashTool.tsx) — exibe progresso quando excede 2 segundos
- `MAX_EDIT_FILE_SIZE = 1024 * 1024 * 1024` (FileEditTool.ts) — limite de 1GiB para edição de arquivos
