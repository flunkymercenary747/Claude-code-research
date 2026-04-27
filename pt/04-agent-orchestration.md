# Capítulo 4: Orquestração de Agentes e Arquitetura Multiagente

## 4.1 Visão Geral e Posicionamento

O sistema multiagente do Claude Code é o subsistema mais complexo de toda a arquitetura do produto, abrangendo cerca de 8.700 linhas de código central em 12 módulos-chave. Este sistema resolve um problema fundamental de engenharia: **como um aplicativo REPL single-threaded pode orquestrar de forma segura e eficiente a execução concorrente de múltiplos LLM Agents**.

O sistema oferece três modos progressivos de colaboração:

| Modo | Acionamento | Concorrência | Mecanismo de comunicação | Nível de isolamento |
|------|---------|--------|---------|---------|
| **Subagent (padrão)** | Chamada ao AgentTool | Síncrono/assíncrono | Valor de retorno de função | Async Generator em processo |
| **Coordinator Mode** | `CLAUDE_CODE_COORDINATOR_MODE=1` | Totalmente assíncrono | XML `<task-notification>` | AbortController independente |
| **Team Mode** | `spawnTeammate()` + TeamFile | Paralelo persistente | Caixa de correio em arquivo + polling | Tmux Pane / InProcess / Remote |

Esses três modos não são implementações independentes — compartilham o mesmo motor central `runAgent()` (`runAgent.ts`), realizando características de comportamento distintas por combinação de parâmetros. Esta é uma das decisões de design mais elegantes de todo o sistema.

**Estatísticas de escala do código-fonte:**

| Arquivo | Linhas | Responsabilidade |
|------|------|------|
| `AgentTool.tsx` | 1.397 | Ponto de entrada unificado, decisão de roteamento, gerenciamento do ciclo de vida |
| `runAgent.ts` | 973 | Motor de execução de Agent, loop query() |
| `loadAgentsDir.ts` | 755 | Análise de definições de Agent (Markdown/JSON/Plugin) |
| `agentToolUtils.ts` | 686 | Filtragem de ferramentas, permissões, serialização de resultados |
| `UI.tsx` | 871 | Renderização de progresso e resultados de Agent |
| `coordinatorMode.ts` | 369 | Prompt de sistema e contexto do Coordinator |
| `SendMessageTool.ts` | 917 | Roteamento de 5 vias de mensagens |
| `spawnMultiAgent.ts` | 1.093 | Criação de Teammates (Tmux/InProcess) |
| `inProcessRunner.ts` | 1.552 | Implementação completa do backend InProcess |
| `teammateMailbox.ts` | 1.183 | Protocolo de caixa de correio em arquivo |
| `worktree.ts` | 1.519 | Isolamento Git Worktree |

## 4.2 Fundamentos Teóricos

### 4.2.1 Modelo de Atores e sua Relação com Orquestração de Agentes

A arquitetura multiagente do Claude Code é uma variante pragmática do modelo de atores no domínio de orquestração de LLM. Os três primitivos centrais do modelo de atores clássico (Hewitt, 1973) — **receber mensagens, criar novos atores, enviar mensagens** — têm correspondências claras no código:

| Primitivo do Ator | Implementação no Claude Code | Localização no código-fonte |
|-----------|-----------------|---------|
| Receber mensagem | Loop de polling `waitForNextPromptOrShutdown()` | `inProcessRunner.ts:689-868` |
| Criar Ator | `AgentTool.call()` / `spawnTeammate()` | `AgentTool.tsx:238-764` |
| Enviar mensagem | `writeToMailbox()` / `SendMessageTool.call()` | `teammateMailbox.ts:133-191` |

Mas há dois desvios-chave em relação ao modelo de atores puro:

1. **Hierarquia assimétrica**: o Leader tem uma visão global (AppState); os Workers têm apenas seu próprio ToolUseContext. Não são atores pares, mas uma árvore de supervisão com hierarquia clara Leader-Worker.
2. **Canal de estado compartilhado**: o backend InProcess de Teammates compartilha o AppState raiz via `setAppStateForTasks` (`runAgent.ts:336-337`), em vez de passagem pura de mensagens. É um compromisso pragmático com o modelo de atores — dentro de um único processo, o estado compartilhado é mais eficiente do que mensagens serializadas.

### 4.2.2 Modelos de Concorrência: Passagem de Mensagens vs. Memória Compartilhada

O sistema usa simultaneamente dois modelos de concorrência, escolhendo com base no nível de isolamento:

**Modelo de passagem de mensagens** (Team Mode - backend Tmux Pane):
```
Leader → writeToMailbox("worker-1", {...}) → sistema de arquivos → readMailbox() → Worker
```
A comunicação é implementada via arquivos JSON + bloqueio de arquivo. As `LOCK_OPTIONS` em `teammateMailbox.ts` configuram retry com backoff exponencial (10 tentativas, 5-100ms) para serializar escritas concorrentes:

```typescript
// teammateMailbox.ts:34-40
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

**Modelo de memória compartilhada** (backend InProcess):
```
Leader → setAppState(prev => {...}) → mesmo AppState store ← getAppState() ← Worker
```
Teammates InProcess leem e escrevem no store raiz diretamente via `toolUseContext.setAppStateForTasks`. Condições de corrida são evitadas via atualizações funcionais `setAppState(prev => {...})` no estilo React (embora o substrato não seja React, adota o mesmo padrão CAS).

### 4.2.3 O Padrão Coordinator em Sistemas Distribuídos

O design do Coordinator Mode mapeia para o padrão Coordinator clássico (também chamado Master-Worker) em sistemas distribuídos, mas adiciona uma restrição única: **o Coordinator em si é um LLM Agent; sua "lógica de coordenação" não é codificada diretamente, mas programada via prompt de sistema**.

A função `getCoordinatorSystemPrompt()` em `coordinatorMode.ts:126-369` retorna um prompt estruturado de cerca de 5.000 caracteres, contendo uma estratégia completa de agendamento de Workers:

```typescript
// coordinatorMode.ts:161-167 — regras de agendamento-chave
// Phase | Who       | Purpose
// Research | Workers (parallel) | Investigate codebase
// Synthesis | **You** (coordinator) | Read findings, craft specs
// Implementation | Workers | Make targeted changes
// Verification | Workers | Test changes work
```

Esse padrão de "programar lógica de coordenação via prompt" significa que o comportamento do Coordinator pode ser ajustado modificando o prompt — o fluxo de trabalho de quatro fases (pesquisa→síntese→implementação→verificação) não é imposto pelo código, mas realizado por meio da capacidade de seguir instruções do LLM. Isso contrasta marcadamente com a lógica de agendamento codificada de um Coordinator distribuído tradicional.

## 4.3 Arquitetura e Estruturas de Dados

### 4.3.1 Diagrama Geral da Arquitetura (Leader-Worker)

```
                    ┌─────────────────────────────────────────┐
                    │           Usuário Humano (Terminal)       │
                    └──────────────┬──────────────────────────┘
                                   │ entrada do usuário
                    ┌──────────────▼──────────────────────────┐
                    │         REPL principal (loop query())    │
                    │    ┌──────────────────────────────┐     │
                    │    │  AgentTool.call() — roteamento│     │
                    │    └──┬─────────┬─────────┬───────┘     │
                    │       │         │         │              │
                    │  ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐     │
                    │  │ Agent  │ │ Agent  │ │Teammate │     │
                    │  │ Sínc.  │ │ Assínc.│ │ Spawn   │     │
                    │  │(bloqueia)│(fire&  │ │         │     │
                    │  │        │ │forget) │ │         │     │
                    │  └───┬────┘ └───┬────┘ └──┬──────┘     │
                    │      │          │         │              │
                    │      └────┬─────┘    ┌────▼──────────┐  │
                    │           │          │  spawnMulti-   │  │
                    │      ┌────▼────┐     │  Agent.ts      │  │
                    │      │runAgent │     └────┬───────────┘  │
                    │      │  .ts    │          │              │
                    │      │         │     ┌────▼──────────┐  │
                    │      │ query() │     │  3 Backends:   │  │
                    │      │  loop   │     │ • Tmux Pane    │  │
                    │      │         │     │ • InProcess    │  │
                    │      └─────────┘     │ • Remote (ant) │  │
                    │                      └───────────────┘  │
                    └─────────────────────────────────────────┘

    Camada de comunicação:
    ┌──────────────────────────────────────────────────────┐
    │                                                      │
    │  Agent Síncrono:   yield message → pai coleta        │
    │  Agent Assíncrono: XML <task-notification> → msg usr │
    │  Teammate:         caixa de correio em arquivo        │
    │                    (.claude/teams/*/inboxes/)         │
    │  InProcess:        AppState compartilhado + mailbox  │
    │  Remote (ant):     teleportToRemote() → sessão CCR   │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 4.3.2 Sistema de Tipos de AgentDefinition

As definições de Agent usam um design de tipo de união em três camadas:

```typescript
// loadAgentsDir.ts — hierarquia de tipos central

// Tipo base: campos compartilhados por todos os Agents
type BaseAgentDefinition = {
  agentType: string              // chave de roteamento (como "Explore", "worker")
  whenToUse: string              // base para o LLM escolher o Agent
  tools?: string[]               // lista branca (undefined = todos)
  disallowedTools?: string[]     // lista negra
  model?: string                 // 'inherit' | nome específico do modelo
  effort?: EffortValue           // nível de esforço de raciocínio
  permissionMode?: PermissionMode // estratégia de herança de permissões
  maxTurns?: number              // número máximo de turnos de conversa
  background?: boolean           // sempre executa em background
  isolation?: 'worktree' | 'remote' // modo de isolamento
  memory?: AgentMemoryScope      // memória persistente
  omitClaudeMd?: boolean         // omite CLAUDE.md (economiza ~5-15 Gtok/semana)
  // ...
}

// Agent integrado: prompt dinâmico, sem systemPrompt estático
type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: (params: { toolUseContext }) => string
}

// Agent personalizado: carregado de Markdown/JSON
type CustomAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: SettingSource  // 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
}

// Agent de Plugin: proveniente do sistema de plugins
type PluginAgentDefinition = BaseAgentDefinition & {
  getSystemPrompt: () => string
  source: 'plugin'
  plugin: string
}

// Tipo de união final
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition | PluginAgentDefinition
```

A elegância deste design está no método `getSystemPrompt`: Agents integrados recebem um parâmetro `toolUseContext` (podendo ajustar o prompt dinamicamente com base no conjunto atual de ferramentas), enquanto Agents personalizados/Plugin usam closures para capturar o conteúdo Markdown analisado no momento do carregamento. Isso significa que:

- **O prompt de Agents integrados é dinâmico**: pode variar a cada chamada
- **O prompt de Agents personalizados é estático**: definido pelo arquivo Markdown, mas se `memory` estiver ativado, conteúdo de memória é anexado em tempo de execução (`loadAgentsDir.ts:335-340`)

A prioridade de carregamento das definições de Agent segue uma cadeia de sobreposição: `builtInAgents → pluginAgents → userAgents → projectAgents → flagAgents → managedAgents`, implementada com um Map via `getActiveAgentsFromList()` onde o último sobrescreve os anteriores (`loadAgentsDir.ts:169-186`).

### 4.3.3 Abstração Unificada dos Três Backends de Execução

Os três backends compartilham a interface AsyncGenerator de `runAgent()`, mas diferem significativamente no modelo de processo e no mecanismo de comunicação:

| Dimensão | Tmux Pane | InProcess | Remote (apenas ant) |
|------|-----------|-----------|-------------------|
| **Modelo de processo** | Processo CLI Claude independente | Isolamento AsyncLocalStorage no mesmo processo | Sessão remota CCR |
| **Inicialização** | `sendCommandToPane(paneId, cmd)` | `startInProcessTeammate(config)` | `teleportToRemote({...})` |
| **Comunicação** | Polling de caixa de correio em arquivo (500ms) | AppState compartilhado + fallback por caixa de correio | HTTP API |
| **Permissões** | Contexto de permissão independente | Ponte via fila de UI do Leader | Remoto independente |
| **Custo de recursos** | Alto (processo completo) | Baixo (heap V8 compartilhado) | Muito alto (instância remota) |
| **Tempo de vida** | Independente do Leader | Vinculado ao processo do Leader | Independente |
| **Lógica de detecção** | `isTmuxAvailable()` | `isInProcessEnabled()` | `process.env.USER_TYPE === 'ant'` |

A detecção de backend e degradação em `spawnMultiAgent.ts:339-375` implementa uma cadeia de degradação elegante:

```
iTerm2 (backend it2) → Tmux → fallback InProcess
```

Se o iTerm2 for detectado, mas o CLI it2 não estiver instalado, o sistema exibe um prompt de configuração interativo (`It2SetupPrompt`), permitindo ao usuário escolher entre instalar o it2 ou degradar para Tmux.

### 4.3.4 Estruturas de Dados do Protocolo de Comunicação

**Formato de mensagem da caixa de correio em arquivo** (`teammateMailbox.ts:42-49`):

```typescript
type TeammateMessage = {
  from: string       // nome do remetente
  text: string       // conteúdo da mensagem (pode ser texto simples ou mensagem JSON estruturada)
  timestamp: string  // timestamp ISO
  read: boolean      // marcador de lido
  color?: string     // identificador de cor do remetente
  summary?: string   // resumo de prévia para UI (5-10 palavras)
}
```

O caminho da caixa de correio segue o formato fixo: `~/.claude/teams/{team_name}/inboxes/{agent_name}.json`

**Tipos de mensagens estruturadas** (transmitidas via codificação JSON no campo `text`):

| Tipo de mensagem | Direção | Finalidade |
|---------|------|------|
| `shutdown_request` | Leader → Worker | Solicitar encerramento |
| `shutdown_approved` / `shutdown_rejected` | Worker → Leader | Resposta de encerramento |
| `idle_notification` | Worker → Leader | Notificação de inatividade |
| `permission_request` | Worker → Leader | Solicitação de permissão |
| `permission_response` | Leader → Worker | Resposta de permissão |
| `plan_approval_request` | Worker → Leader | Solicitação de aprovação do Plan Mode |
| `plan_approval_response` | Leader → Worker | Resposta de aprovação |
| `sandbox_permission_request` / `_response` | Bidirecional | Permissão de sandbox de rede |
| `task_assignment` | Leader → Worker | Atribuição de tarefa |
| `team_permission_update` | Leader → Workers | Broadcast de permissões |

## 4.4 Algoritmos Centrais e Fluxos

### 4.4.1 Árvore de Decisão Completa de Roteamento do AgentTool

`AgentTool.call()` é o ponto de entrada unificado do sistema; sua lógica de roteamento é implementada em `AgentTool.tsx:238-764`. A árvore de decisão completa é:

```
AgentTool.call(input) — ponto de entrada
│
├─ [1] Os parâmetros team name E name estão presentes?
│   ├─ SIM: É uma tentativa de Teammate de criar agentes aninhados?
│   │   ├─ SIM: throw "roster is flat" (AgentTool.tsx:272)
│   │   └─ NÃO: → spawnTeammate() (retorna teammate_spawned)
│   └─ NÃO: continua
│
├─ [2] Resolve effectiveType (subagent_type)
│   ├─ Especificado explicitamente → usa valor especificado
│   ├─ Não especificado + Fork Gate ON → undefined (caminho Fork)
│   └─ Não especificado + Fork Gate OFF → GENERAL_PURPOSE_AGENT
│
├─ [3] effectiveType === undefined? (caminho Fork)
│   ├─ SIM: verificação recursiva de Fork
│   │   ├─ Já em subprocesso Fork → throw
│   │   └─ Passa → selectedAgent = FORK_AGENT
│   └─ NÃO: busca em activeAgents
│       ├─ Encontrado → selectedAgent = found
│       ├─ Negado por permissão → throw (com info de regra de negação)
│       └─ Inexistente → throw (lista agents disponíveis)
│
├─ [4] Resolve effectiveIsolation
│   ├─ 'remote' (apenas ant) → teleportToRemote() → retorna remote_launched
│   └─ 'worktree' → createAgentWorktree(slug) → etapas posteriores usam worktreePath
│
├─ [5] Constrói prompt de sistema e mensagens de prompt
│   ├─ Caminho Fork: herda prompt pai + buildForkedMessages()
│   └─ Normal: getSystemPrompt() + enhanceSystemPromptWithEnvDetails()
│
├─ [6] Determina shouldRunAsync
│   │   = run_in_background
│   │   || selectedAgent.background
│   │   || isCoordinator
│   │   || forceAsync (Fork Gate)
│   │   || assistantForceAsync (KAIROS)
│   │   || proactiveActive
│   │   — MAS NÃO isBackgroundTasksDisabled
│   │
│   ├─ ASYNC: registerAsyncAgent() → void runAsyncAgentLifecycle()
│   │   → retorna { status: 'async_launched', agentId, outputFile }
│   │
│   └─ SYNC: registerAgentForeground() → entra no loop while(true)
│       ├─ Race: nextMessage vs backgroundSignal
│       │   ├─ background ganha → muda para execução assíncrona (wasBackgrounded=true)
│       │   └─ message ganha → yield message, rastreia progresso
│       └─ Loop termina → finalizeAgentTool() → retorna AgentToolResult
```

### 4.4.2 Fluxo de Execução do AsyncGenerator runAgent()

`runAgent()` é o motor central de todo o sistema multiagente (`runAgent.ts:247-860`), sendo um `AsyncGenerator<Message, void>` — a cada Message produzida via yield, o chamador pode processá-la (registrar, exibir ou enfileirar em background).

**Fases-chave do fluxo de execução:**

1. **Resolução de ferramentas**: `resolveAgentTools()` resolve a lista branca `tools` da definição de Agent para objetos Tool reais, aplicando simultaneamente a lista negra `disallowedTools` (`runAgent.ts:500-502`)

2. **Construção do System Prompt**: baseado em `override?.systemPrompt` ou `getAgentSystemPrompt()`; Agents Explore/Plan ignoram `claudeMd` e `gitStatus`, economizando ~5-15 Gtok/semana por toda a frota (`runAgent.ts:389-409`)

3. **Estratégia de AbortController** (`runAgent.ts:524-528`):
   ```typescript
   const agentAbortController = override?.abortController
     ? override.abortController     // controle externo (caminho async)
     : isAsync
       ? new AbortController()      // assíncrono: controller independente
       : toolUseContext.abortController  // síncrono: compartilha controller pai
   ```

4. **Sobreposição de permissões** (`runAgent.ts:414-497`): o `permissionMode` do Agent sobrescreve o modo do pai, mas os modos `bypassPermissions`, `acceptEdits` e `auto` do pai sempre têm prioridade — garantindo que políticas de segurança do administrador não sejam rebaixadas por subagentes.

5. **Loop central** — chama `query()` diretamente e faz yield (`runAgent.ts:748-806`):
   ```typescript
   for await (const message of query({
     messages: initialMessages,
     systemPrompt: agentSystemPrompt,
     userContext: resolvedUserContext,
     systemContext: resolvedSystemContext,
     canUseTool,
     toolUseContext: agentToolUseContext,
     querySource,
     maxTurns: maxTurns ?? agentDefinition.maxTurns,
   })) {
     // ... trata stream_event, attachment, mensagens registráveis
     if (isRecordableMessage(message)) {
       await recordSidechainTranscript([message], agentId, lastRecordedUuid)
       yield message
     }
   }
   ```

6. **Bloco finally de limpeza** (`runAgent.ts:816-858`): limpeza MCP, limpeza de session hooks, rastreamento de Prompt Cache, liberação de cache de estado de arquivo, kill de tarefas bash em background, desregistro do Perfetto, limpeza de todos do AppState, limpeza do subdiretório de transcrição do Agent — 9 operações de limpeza no total, garantindo que não haja vazamento de recursos.

### 4.4.3 Ciclo de Vida do Agent Assíncrono (fire-and-forget)

O ciclo de vida completo do Agent assíncrono é acionado por `runAsyncAgentLifecycle()` (`agentToolUtils.ts:322-497`):

```
registerAsyncAgent() → registra tarefa no AppState
   │
   ▼
void runAsyncAgentLifecycle() — fire-and-forget
   │
   ├─ for await (message of makeStream()) — coleta todas as mensagens
   │   ├─ agentMessages.push(message)
   │   ├─ se task.retain → anexa a AppState.tasks[taskId].messages
   │   ├─ updateProgressFromMessage(tracker, ...)
   │   └─ emitTaskProgress() — eventos de progresso SDK
   │
   ├─ finalizeAgentTool() — extrai resultado final
   │
   ├─ completeAsyncAgent() — marca como concluído (PRIMEIRO, antes de qualquer operação lenta)
   │   │                      ↑ design crítico: correção gh-20236
   │   │                        classifyHandoff e worktree cleanup podem travar
   │   │                        não podem bloquear a transição de estado
   │
   ├─ classifyHandoffIfNeeded() — verificação do classificador de segurança (opcional)
   │
   ├─ getWorktreeResult() — limpeza do worktree
   │
   └─ enqueueAgentNotification() — notifica pai via XML <task-notification>
```

**A correção gh-20236** é uma decisão de design que vale registrar: `completeAsyncAgent()` é chamado antes de `classifyHandoffIfNeeded()` e `getWorktreeResult()`. O comentário explica claramente o motivo:

> Mark task completed FIRST so TaskOutput(block=true) unblocks immediately. classifyHandoffIfNeeded (API call) and getWorktreeResult (git exec) are notification embellishments that can hang — they must not gate the status transition (gh-20236).

### 4.4.4 Filtragem de Ferramentas e Herança de Permissões

A filtragem de ferramentas é uma cadeia de filtros em três camadas (`agentToolUtils.ts:66-115`):

```
Camada 1: ALL_AGENT_DISALLOWED_TOOLS — ferramentas proibidas para todos os Agents
Camada 2: CUSTOM_AGENT_DISALLOWED_TOOLS — ferramentas adicionalmente proibidas apenas para Agents personalizados
Camada 3: ASYNC_AGENT_ALLOWED_TOOLS — lista branca para Agents assíncronos (lógica invertida)
```

Exceções especiais:
- Ferramentas MCP (prefixo `mcp__`) são sempre permitidas
- `ExitPlanMode` é sempre permitido no Plan Mode
- Teammates InProcess no modo Agent Swarms podem usar `AgentTool` (para criar subagentes síncronos) e ferramentas Task (para coordenação via lista de tarefas compartilhada)

A resolução de ferramentas também suporta curingas (`'*'` ou `undefined` = todas as ferramentas) e restrições com escopo de Agent (sintaxe `AgentTool(worker, researcher)`, `agentToolUtils.ts:165-172`).

### 4.4.5 Fluxo de Trabalho de Quatro Fases do Coordinator Mode

A lógica central do Coordinator Mode é definida via prompt em `getCoordinatorSystemPrompt()` em `coordinatorMode.ts:126-369`. Ele divide todas as tarefas em quatro fases:

**Fase 1: Pesquisa** (Workers em paralelo)
- Múltiplos Workers exploram simultaneamente o codebase
- Instrução de prompt-chave: *"Parallelism is your superpower. Launch independent workers concurrently whenever possible."*

**Fase 2: Síntese** (o próprio Coordinator)
- Esta é a fase mais crítica — o Coordinator deve ler e compreender pessoalmente os resultados da pesquisa
- Anti-padrão explicitamente proibido: *"Never write 'based on your findings'"*
- Requer produzir uma spec sintetizada, com caminhos específicos de arquivo, números de linha e conteúdo de modificações

**Fase 3: Implementação** (Workers executam)
- O Coordinator decide se continua (`SendMessageTool`) ou cria novo (`AgentTool`)
- A decisão é baseada no grau de sobreposição de contexto (o prompt contém uma tabela de decisão completa)

**Fase 4: Verificação** (Worker independente)
- Verificação independente explicitamente solicitada: *"Verifier should see the code with fresh eyes, not carry implementation assumptions"*
- Critério de verificação: *"proving the code works, not confirming it exists"*

### 4.4.6 Colaboração Persistente no Team Mode

O Team Mode implementa estado de equipe persistente baseado em TeamFile (`.claude/teams/{team_name}/team.json`). Diferentemente dos Workers fire-and-forget do Coordinator Mode, Teammates são **processos de longa duração**:

1. **Criação**: `spawnTeammate()` cria um pane Tmux ou tarefa InProcess
2. **Execução**: Teammate executa prompt → conclui → envia `idle_notification` → aguarda próximo prompt
3. **Comunicação**: todas as mensagens via caixa de correio em arquivo (qualquer backend pode usar comunicação pelo sistema de arquivos)
4. **Encerramento**: Leader envia `shutdown_request` → o LLM do Teammate decide aprovar ou rejeitar

O loop principal do InProcess Runner (`inProcessRunner.ts:883-1464`) implementa a semântica completa de persistência:

```typescript
while (!abortController.signal.aborted && !shouldExit) {
  // 1. Executa prompt atual (chama runAgent())
  // 2. Marca como inativo
  // 3. Envia idle_notification ao Leader
  // 4. waitForNextPromptOrShutdown() — polling da caixa de correio
  //    ├─ shutdown_request → passa ao LLM para decidir
  //    ├─ new_message → define como próximo prompt
  //    └─ aborted → shouldExit = true
}
```

Vale notar a estratégia de prioridade de mensagens (`inProcessRunner.ts:760-804`):
1. Prioridade máxima: `shutdown_request` (instrução de encerramento do Leader não é enterrada)
2. Em seguida: mensagens de `team-lead` (o Leader representa a intenção do usuário)
3. Por último: mensagens peer na fila FIFO

### 4.4.7 Protocolo de Comunicação da Caixa de Correio em Arquivo

A caixa de correio em arquivo é a base de comunicação de todos os backends. Seu design escolheu **simplicidade** em vez de desempenho:

**Protocolo de escrita** (`teammateMailbox.ts:133-191`):
```
1. ensureInboxDir() — garante que o diretório existe
2. writeFile(inboxPath, '[]', { flag: 'wx' }) — criação atômica (se não existir)
3. lockfile.lock(inboxPath, LOCK_OPTIONS) — obtém bloqueio de arquivo
4. readMailbox() — releitura dentro do bloqueio (evita leitura suja)
5. messages.push(newMessage)
6. writeFile(inboxPath, JSON.stringify(messages)) — escreve de volta
7. release() — libera o bloqueio
```

**Protocolo de leitura** (`teammateMailbox.ts:83-107`):
```
1. readFile(inboxPath, 'utf-8')
2. JSON.parse(content)
3. retorna TeammateMessage[]
```

Note que a leitura é **sem bloqueio** — design intencional. O lado de leitura precisa apenas de consistência eventual; o lado de escrita usa `lockfile` para garantir atomicidade.

### 4.4.8 Roteamento de 5 Vias do SendMessage

`SendMessageTool.call()` implementa 5 caminhos independentes de roteamento de mensagens (`SendMessageTool.ts`):

```
valor de input.to
│
├─ [Rota 1] parseAddress(to).scheme === 'bridge'
│   → postInterClaudeMessage() — Controle Remoto entre máquinas
│   (requer verificação de segurança: mensagens entre máquinas precisam de consentimento explícito do usuário)
│
├─ [Rota 2] parseAddress(to).scheme === 'uds'
│   → sendToUdsSocket() — Unix Domain Socket local
│
├─ [Rota 3] agentNameRegistry ou toAgentId corresponde
│   ├─ task.status === 'running' → queuePendingMessage()
│   └─ task stopped/evicted → resumeAgentBackground()
│       (retoma automaticamente Agent parado a partir da transcrição em disco)
│
├─ [Rota 4] to === '*'
│   → handleBroadcast() — percorre TeamFile.members e escreve na caixa de correio de cada um
│
└─ [Rota 5] outros
    ├─ texto simples → handleMessage() — escreve na caixa de correio
    └─ mensagem estruturada → despacha para handler correspondente:
        ├─ shutdown_request → handleShutdownRequest()
        ├─ shutdown_response (approve) → handleShutdownApproval()
        ├─ shutdown_response (reject) → handleShutdownRejection()
        ├─ plan_approval_response (approve) → handlePlanApproval()
        └─ plan_approval_response (reject) → handlePlanRejection()
```

O mecanismo de **retomada automática** na Rota 3 é particularmente elegante: quando uma mensagem é enviada a um Agent já parado, o sistema o retoma automaticamente a partir da transcrição em disco e o executa em background. Isso significa que o Coordinator pode continuar um Worker anteriormente concluído via `SendMessage` sem precisar se preocupar se ele ainda está em execução.

### 4.4.9 Fluxo Completo de Delegação de Permissões

O tratamento de permissões do Teammate InProcess é uma das partes mais complexas do sistema (`inProcessRunner.ts:127-449`). O desafio central: **como um Agent em background solicita autorização humana?**

A solução é um fallback em dois níveis:

**Caminho principal: ponte via fila de UI do Leader**
```
Worker aciona ferramenta que requer permissão
  → createInProcessCanUseTool() é chamado
  → hasPermissionsToUseTool() retorna { behavior: 'ask' }
  → verifica aprovação automática do classificador Bash (se disponível)
  → getLeaderToolUseConfirmQueue() — obtém fila de confirmação de UI do Leader
  → setToolUseConfirmQueue(queue => [...queue, { tool, input, workerBadge, ... }])
     │                                           ↑ identificador de identidade do Worker
     └→ terminal do Leader exibe caixa de diálogo de permissão com badge do Worker
        ├─ onAllow → persistPermissionUpdates() + resolve({ behavior: 'allow' })
        └─ onReject → resolve({ behavior: 'ask', message: REJECT_MESSAGE })
```

**Caminho de fallback: solicitação de permissão via caixa de correio**
```
Worker aciona ferramenta que requer permissão
  → fila de UI do Leader indisponível
  → createPermissionRequest({...})
  → sendPermissionRequestViaMailbox(request)
  → polling da própria caixa de correio (intervalo de 500ms)
  → aguarda Leader escrever permission_response de volta
  → processMailboxPermissionResponse()
```

A propagação de atualizações de permissão também é importante: quando o Leader aprova uma permissão e escolhe "Always allow", `persistPermissionUpdates()` escreve no disco; simultaneamente, `getLeaderSetToolPermissionContext()` escreve a atualização de volta para o contexto compartilhado do Leader — mas com `preserveMode: true`, impedindo que o modo `acceptEdits` do Worker vaze de volta para o Coordinator (`inProcessRunner.ts:275-277`).

### 4.4.10 Ciclo de Vida Completo do Worker

```
Nascimento
  │
  ├─ Caminho Agent Síncrono:
  │   AgentTool.call() → createAgentId() → registerAgentForeground()
  │   → runAgent() { for await yield message }
  │   → finalizeAgentTool() → retorna AgentToolResult
  │   → unregisterAgentForeground()
  │
  ├─ Caminho Agent Assíncrono:
  │   AgentTool.call() → createAgentId() → registerAsyncAgent()
  │   → void runAsyncAgentLifecycle() (fire-and-forget)
  │   → runAgent() → finalizeAgentTool()
  │   → completeAsyncAgent() → enqueueAgentNotification()
  │
  └─ Caminho InProcess Teammate:
      spawnTeammate() → spawnInProcessTeammate() → startInProcessTeammate()
      → runInProcessTeammate() — loop persistente:
          while (!aborted && !shouldExit) {
            runAgent(currentPrompt) → idle_notification
            → waitForNextPromptOrShutdown()
            → nova mensagem/shutdown/abort → decide se continua
          }

Em Execução
  │
  ├─ loop query() → chamada de API → tool_use → verificação canUseTool
  │   ├─ allow → executa ferramenta
  │   ├─ deny → ferramenta negada
  │   └─ ask → caixa de diálogo de permissão (síncrono) ou permissão via caixa de correio (assíncrono/teammate)
  │
  ├─ Rastreamento de progresso:
  │   updateProgressFromMessage() → updateAsyncAgentProgress()
  │   → emitTaskProgress() (evento SDK)
  │
  └─ Movimentação automática para background (apenas Agent Síncrono):
      race de backgroundPromise → se usuário pressiona Ctrl+Z
      → wasBackgrounded = true → continua executando em background

Comunicação
  │
  ├─ Agent Síncrono: yield message → pai coleta diretamente
  ├─ Agent Assíncrono: <task-notification> injetado nas mensagens de usuário do pai
  └─ Teammate: writeToMailbox() → Leader lê via polling

Encerramento
  │
  ├─ Conclusão normal: finalizeAgentTool() → extrai texto final → marca como completed
  ├─ Kill pelo usuário: AbortError → killAsyncAgent() → extrai partialResult → notifica
  ├─ Erro: catch → failAsyncAgent() → notifica erro
  └─ Limpeza: finally {
       mcpCleanup(), clearSessionHooks(), cleanupAgentTracking(),
       readFileState.clear(), killShellTasksForAgent(),
       unregisterPerfettoAgent(), clearAgentTranscriptSubdir()
     }
```

### 4.4.11 Criação e Limpeza de Isolamento Worktree

O Git Worktree fornece isolamento a nível de sistema de arquivos para Agents (`worktree.ts`). Fluxo principal:

**Criação** (`worktree.ts:234-374`):
```
1. validateWorktreeSlug(slug) — previne ataques de path traversal
2. verificação de retomada rápida: readWorktreeHeadSha() — se worktree já existe, pula o fetch
3. Se não existe:
   a. tenta ler ref local origin/<default> (evita ~6-8s de custo de `git fetch`)
   b. se não existe localmente → git fetch origin <branch>
   c. git worktree add -B <branch> <path> <base>
   d. opcional: sparse-checkout (checkout apenas de caminhos especificados)
4. performPostCreationSetup():
   - copia settings.local.json
   - configura git hooks (lida com problema core.hooksPath do husky)
   - symlinks de diretórios grandes como node_modules
   - copia arquivos gitignored especificados em .worktreeinclude
```

**Decisão de limpeza** (`AgentTool.tsx:644-685`):
```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (hookBased) return { worktreePath }; // baseado em Hook: sempre mantém
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {}; // sem mudanças: remove worktree
    }
  }
  return { worktreePath, worktreeBranch }; // com mudanças: mantém
};
```

Medidas de segurança-chave:
- `validateWorktreeSlug()` valida que cada segmento separado por `/` corresponde a `[a-zA-Z0-9._-]+`, prevenindo path traversal `../../../`
- `flattenSlug()` achata slugs aninhados (`user/feature` → `user+feature`), evitando conflitos git ref D/F e aninhamento de diretórios
- `GIT_NO_PROMPT_ENV` desabilita todos os prompts de credencial git, prevenindo travamento do CLI

## 4.5 Análise das Decisões de Design

### 4.5.1 Por que Escolher Caixa de Correio em Arquivo em vez de IPC

A caixa de correio em arquivo pode parecer uma escolha "primitiva" — por que não usar Unix Domain Socket, Named Pipe ou gRPC?

**Motivo central: independência de backend**. O sistema de arquivos é o maior denominador comum dos três backends (Tmux, InProcess, Remote):
- Tmux Pane é um processo independente sem memória compartilhada
- InProcess está no mesmo processo mas usa isolamento AsyncLocalStorage
- Remote está em redes cruzadas, mas pode compartilhar sistema de arquivos de rede

Vantagens adicionais da caixa de correio em arquivo:
1. **Observabilidade**: basta `cat ~/.claude/teams/*/inboxes/*.json` para depurar
2. **Persistência**: mensagens não se perdem após crash do processo
3. **Simplicidade**: sem gerenciamento complexo de conexão, heartbeat, reconexão após queda
4. **Segurança de concorrência**: o bloqueio de arquivo fornecido por `proper-lockfile` é suficientemente confiável

O custo é a **latência**: o intervalo de polling de 500ms significa que no pior caso há 500ms de atraso na entrega de mensagens. Mas em cenários de LLM Agent, cada chamada de ferramenta em si leva segundos; 500ms é negligenciável.

### 4.5.2 Tradeoff entre Backends InProcess vs. Pane

| Dimensão | InProcess | Tmux Pane |
|------|-----------|-----------|
| **Memória** | Heap V8 compartilhado (baixo) | Heap de processo independente (alto) |
| **Latência de inicialização** | ~0ms | ~2-3s (inicialização do CLI) |
| **Isolamento** | AsyncLocalStorage (fraco) | Processo do SO (forte) |
| **Permissões** | Ponte via UI do Leader (em tempo real) | Polling via caixa de correio (com atraso) |
| **Depuração** | Logs compartilhados (complexo) | Terminal independente (intuitivo) |
| **Tempo de vida** | Vinculado ao Leader | Independente |

A maior vantagem do backend InProcess é a **ponte de permissões** — via `getLeaderToolUseConfirmQueue()`, a caixa de diálogo de permissão do Worker é exibida diretamente no terminal do Leader com um badge identificador do Worker. Isso significa que o usuário não precisa mudar para o terminal do Worker para aprovar permissões.

Mas o InProcess tem uma limitação fundamental: **Workers não podem criar Agents em background** (`AgentTool.tsx:277-278`), pois seu ciclo de vida está vinculado ao processo do Leader; Agents em background precisam de AbortController independente.

### 4.5.3 Filosofia de Design de que o Controle de Permissões Está Sempre nas Mãos dos Humanos

Toda a arquitetura de permissões do sistema multiagente segue um princípio inegociável: **os humanos são sempre os concessores finais de permissões**.

Manifestações desse princípio no código:
1. **Subagentes não podem elevar permissões**: `runAgent.ts:419` — os modos `bypassPermissions`, `acceptEdits` e `auto` do pai sempre têm prioridade sobre o `permissionMode` do subagente
2. **Permissões do Leader não vazam para Workers**: `runAgent.ts:467-477` — quando `allowedTools` é especificado, regras de allow a nível de sessão são limpas, mantendo apenas regras a nível de argumentos do CLI
3. **Mensagens entre máquinas requerem consentimento explícito**: `SendMessageTool.ts:checkPermissions` — enviar para endereço `bridge:` requer `safetyCheck` e `classifierApprovable: false` (o classificador de segurança não pode aprovar automaticamente)
4. **Aprovação do Plan Mode**: Teammates podem ser configurados como `plan_mode_required`, exigindo que um plano seja submetido ao Leader para aprovação antes da execução

### 4.5.4 Design Recursivo de Reutilização do Loop query()

O núcleo de `runAgent()` é chamar a função `query()` — a mesma função usada pelo loop REPL principal. Isso significa que **subagentes e o agente principal usam exatamente o mesmo pipeline de chamada de API e execução de ferramentas**.

```typescript
// runAgent.ts:748-757 — chamada query() do Agent
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns,
})) { ... }
```

Implicações profundas desse design:
- **Consistência de ferramentas**: as ferramentas usadas pelos Agents são exatamente as mesmas que o usuário usa (apenas filtradas)
- **Capacidade recursiva**: se `AgentTool` estiver no pool de ferramentas de um Agent, este pode criar subagentes (Teammates InProcess têm permissão para criar subagentes síncronos)
- **Reutilização do Prompt Cache**: o caminho Fork usa `useExactTools` para garantir que os prefixos de requisição de API do subagente são byte a byte idênticos aos do agente pai, maximizando a taxa de acerto do Prompt Cache

Mas a recursão também traz riscos — fork recursivo infinito. A solução é uma verificação dupla (`AgentTool.tsx:331-333`):
1. `querySource === 'agent:builtin:fork'` — duradouro em tempo de compilação (context.options não é afetado por autocompact)
2. `isInForkChild(messages)` — fallback por varredura de mensagens

### 4.5.5 Comparação com LangGraph / AutoGen / CrewAI

| Dimensão | Claude Code | LangGraph | AutoGen | CrewAI |
|------|------------|-----------|---------|--------|
| **Modelo de orquestração** | Leader-Worker (programado por prompt) | DAG/StateGraph | Agent Chat | Sequential/Hierarchical |
| **Comunicação** | Caixa de correio em arquivo + AppState compartilhado | Canais de estado | Chamadas de função Python | Memória compartilhada |
| **Isolamento** | 3 níveis (InProcess/Pane/Remote) | Nenhum | Nenhum | Nenhum |
| **Permissões** | Human-in-the-loop, sempre | Opcional | Opcional | Nenhum |
| **Persistência** | Transcrição em disco + TeamFile | Checkpointing opcional | Nenhum | Nenhum |
| **Compartilhamento de ferramentas** | Pool de Tools unificado + filtragem | Vinculação independente por nó | Independente por Agent | Independente por Agent |
| **Heterogeneidade de modelos** | Parâmetro `model` por Agent | Suportado | Suportado | Suportado |

As maiores diferenciações do Claude Code são duas:

1. **A lógica do Coordinator é programada por prompt** — a lógica de orquestração de outros frameworks é um DAG ou máquina de estados codificado. O Coordinator do Claude Code é programado por prompt em linguagem natural, o que significa que a estratégia de coordenação pode ser ajustada modificando o prompt, sem alterar o código.
2. **Sistema de arquivos como base de comunicação** — isso pode parecer primitivo, mas oferece capacidade de comunicação unificada entre processos e entre máquinas, além de total observabilidade. Outros frameworks dependem de chamadas de função Python em processo; cenários multimáquina requerem uma camada RPC adicional.

## 4.6 Padrões Transferíveis

### 4.6.1 Padrões Gerais de Orquestração de Agentes

Da implementação do Claude Code podem ser extraídos 5 padrões gerais de orquestração de Agentes:

**Padrão 1: AsyncGenerator como Interface de Agent**
```typescript
async function* runAgent(params): AsyncGenerator<Message, void> {
  for await (const msg of queryLLM(params)) {
    yield msg;
  }
}
```
AsyncGenerator fornece semântica de fluxo de mensagens pull-based (orientada ao consumidor) — o chamador decide quando consumir a próxima mensagem, suportando naturalmente a troca para background (inserindo race no ponto de yield) e rastreamento de progresso.

**Padrão 2: Transição Seamless de Foreground → Background**

O Agent Síncrono do Claude Code pode ser movido para background durante a execução — via `Promise.race([nextMessage, backgroundSignal])`. Esse padrão se aplica a qualquer cenário que precise de "tarefas longas podem ser movidas para background no meio do caminho". A chave é ter um taskId estável para transferir entre foreground e background.

**Padrão 3: Sistema de Arquivos como "Mínimo Múltiplo Comum" para Comunicação entre Agents**

Quando múltiplos backends (em processo / entre processos / entre máquinas) precisam de comunicação unificada, o sistema de arquivos é a escolha mais simples. Arquivos JSON + bloqueio de arquivo fornecem garantias de consistência suficientes.

**Padrão 4: Coordenação Programada por Prompt**

Escrever a lógica de orquestração no prompt de sistema em vez do código torna a estratégia de coordenação "configuração" em vez de "implementação". Isso é especialmente valioso na fase de iteração rápida da orquestração de Agents — o custo de alterar prompts é muito menor do que alterar código.

**Padrão 5: Transição de Estado Segura tem Prioridade sobre Decorações de Notificação**

O padrão de correção gh-20236: em fluxos assíncronos, primeiro completar a transição de estado central (`completeAsyncAgent`), depois executar operações decorativas que podem travar (verificação do classificador, limpeza de worktree). Qualquer operação que possa bloquear não deve fazer gate de mudanças críticas de estado.

### 4.6.2 O que o FlowController do Doramagic Pode Aprender

A arquitetura de Agents do Claude Code e o FlowController do Doramagic (sistema de lease + isolamento staging/delivery + máquina de 12 estados) têm vários pontos que valem comparação:

1. **Máquina de estados vs. programação por prompt**: o Doramagic usa uma máquina de 12 estados para controle de fluxo codificado; o Claude Code usa programação por prompt para o Coordinator. Ambos têm cenários de aplicação — use máquina de estados para fluxos determinísticos, programação por prompt para fluxos que precisam de julgamento flexível.

2. **Aplicabilidade direta da caixa de correio em arquivo**: o isolamento de diretórios staging/delivery do Doramagic e a estrutura `.claude/teams/*/inboxes/` do Claude Code são análogas. O FlowController do Doramagic pode adotar diretamente o padrão de caixa de correio em arquivo para comunicação fracamente acoplada entre skills.

3. **Aprendizado com o modelo de permissões**: o princípio "subagentes não podem elevar permissões" do Claude Code pode ser mapeado para as permissões de skills do Doramagic — uma skill chamada não deve obter acesso ao sistema superior ao do chamador.

4. **Ideia de isolamento com Worktree**: para execução paralela de skills do Doramagic (como múltiplos soul extractors extraindo projetos diferentes em paralelo), o padrão de isolamento de sistema de arquivos do Worktree pode ser aproveitado, criando diretórios de trabalho independentes para cada execução paralela.

## 4.7 Índice do Código-Fonte

| Arquivo | Caminho | Exportações-chave |
|------|------|---------|
| AgentTool.tsx | `tools/AgentTool/AgentTool.tsx` | `AgentTool` (definição buildTool), `inputSchema`, `outputSchema` |
| runAgent.ts | `tools/AgentTool/runAgent.ts` | AsyncGenerator `runAgent()`, `filterIncompleteToolCalls()` |
| loadAgentsDir.ts | `tools/AgentTool/loadAgentsDir.ts` | União de tipos `AgentDefinition`, `getAgentDefinitionsWithOverrides()`, `parseAgentFromMarkdown/Json()` |
| agentToolUtils.ts | `tools/AgentTool/agentToolUtils.ts` | `filterToolsForAgent()`, `resolveAgentTools()`, `finalizeAgentTool()`, `runAsyncAgentLifecycle()`, `classifyHandoffIfNeeded()` |
| UI.tsx | `tools/AgentTool/UI.tsx` | `renderToolUseMessage()`, `renderToolResultMessage()`, `renderGroupedAgentToolUse()` |
| coordinatorMode.ts | `coordinator/coordinatorMode.ts` | `isCoordinatorMode()`, `getCoordinatorSystemPrompt()`, `getCoordinatorUserContext()` |
| SendMessageTool.ts | `tools/SendMessageTool/SendMessageTool.ts` | `SendMessageTool` (roteamento de 5 vias), `handleMessage/Broadcast/ShutdownRequest/Approval/Rejection()` |
| spawnMultiAgent.ts | `tools/shared/spawnMultiAgent.ts` | `spawnTeammate()`, `handleSpawnSplitPane()`, `resolveTeammateModel()`, `buildInheritedCliFlags()` |
| inProcessRunner.ts | `utils/swarm/inProcessRunner.ts` | `runInProcessTeammate()`, `createInProcessCanUseTool()`, `waitForNextPromptOrShutdown()` |
| teammateMailbox.ts | `utils/teammateMailbox.ts` | `readMailbox()`, `writeToMailbox()`, `markMessageAsReadByIndex()`, todos os tipos de mensagem estruturada |
| worktree.ts | `utils/worktree.ts` | `createWorktreeForSession()`, `createAgentWorktree()`, `removeAgentWorktree()`, `validateWorktreeSlug()` |
| tasks/types.ts | `tasks/types.ts` | União `TaskState` (7 tipos de task), `isBackgroundTask()` |

**União do tipo TaskState** (`tasks/types.ts`):
```typescript
type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

---

*Este capítulo foi concluído com base no snapshot do código-fonte TypeScript do Claude Code (2026-03-31, ~512K LOC). Todas as referências ao código incluem nome de arquivo e intervalo de linhas específicos.*
