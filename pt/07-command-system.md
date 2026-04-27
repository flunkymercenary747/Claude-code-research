# Capítulo 7: Sistema de Comandos

## 7.1 Visão Geral e Posicionamento

O sistema de comandos do Claude Code é a principal entrada de interação do usuário com o REPL. Toda vez que o usuário digita `/` na caixa de input, este sistema é acionado. Ele desempenha três papéis:

1. **Camada de controle da UI**: Manipula diretamente o estado da interface do terminal, sem passar pelo LLM (como `/clear`, `/theme`, `/vim`)
2. **Camada de gerenciamento de sessão**: Gerencia histórico de conversa, compressão e restauração de contexto (como `/compact`, `/resume`, `/branch`)
3. **Camada de extensão de capacidades**: Delega tarefas complexas ao modelo para execução, via mecanismo de expansão de Prompt (como `/review`, `/skills`)

O design de fronteiras do sistema de comandos reflete uma clara separação de responsabilidades: comandos são responsáveis pelo "acionamento", ferramentas (Tools) pelo "execução", e o LLM pela "decisão". Um comando `/review` não chama git diretamente, mas injeta o prompt de revisão no fluxo de conversa, deixando o modelo conduzir a cadeia de chamadas de ferramentas subsequentes.

---

## 7.2 Fundamentos Teóricos

### Padrão Command (Command Pattern)

O design do sistema está altamente alinhado com o padrão clássico GoF Command:

- **Interface Command**: O tipo union `Command` (`PromptCommand | LocalCommand | LocalJSXCommand`) encapsula uniformemente requisições
- **ConcreteCommand**: Cada arquivo `commands/<name>/index.ts` é uma implementação concreta de comando
- **Invoker**: O `processSlashCommand` do REPL é responsável pelo despacho e execução
- **Receiver**: `ToolUseContext` (estado da conversa) e `AppState` (estado da aplicação) são os objetos sendo manipulados

Porém, o Claude Code fez duas extensões-chave no padrão clássico:

**Carregamento tardio (Lazy Loading)**: Comandos são carregados com atraso via `load(): Promise<Module>` em vez de instanciados imediatamente no registro. Isso distribui o overhead de inicialização para a primeira chamada, o que é significativo para comandos com dependências pesadas (como o módulo de renderização HTML de 113KB do `/insights`).

**Valores de retorno tipados**: Comandos não são ações sem valor de retorno (void), mas retornam resultados estruturados (`LocalCommandResult`), deixando o REPL de nível superior decidir como renderizar, alcançando desacoplamento entre execução e apresentação.

### Padrões de Design no Processamento de Comandos REPL

O processamento de comandos REPL do Claude Code segue dois princípios centrais:

**Immediate vs Queued**: O campo `immediate?: boolean` no objeto de comando determina se o comando é executado imediatamente, bypassando a fila de mensagens. Operações de interface como `/clear` e `/exit` precisam de resposta imediata, enquanto operações envolvendo chamadas de API como `/compact` entram na fila para processamento ordenado.

**Disponibilidade Auth-gated**: Diferente dos feature flags em tempo de execução (`isEnabled()`), o campo `availability` entra em vigor na fase de filtragem da lista de comandos, garantindo que usuários não autorizados nem vejam a existência de determinados comandos (como comandos exclusivos para assinantes do claude.ai).

---

## 7.3 Mecanismo de Registro de Comandos

### Fluxo de Registro no commands.ts

A lógica central de registro de comandos está concentrada em `commands.ts` (754 linhas), dividida em quatro camadas:

**Primeira camada: Comandos embutidos estáticos**

```typescript
// commands.ts:240-310 (fragmento central)
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome, clear, color,
  compact, config, copy, desktop, context, contextNonInteractive,
  cost, diff, doctor, effort, exit, fast, files, heapDump,
  help, ide, init, keybindings, installGitHubApp, installSlackApp,
  mcp, memory, mobile, model, outputStyle, remoteEnv, plugin,
  // ... aproximadamente 60 comandos embutidos
])
```

A função `COMMANDS` é envolta em `memoize` em vez de uma constante de nível de módulo, porque alguns comandos precisam ler arquivos de configuração no momento do registro, e a configuração ainda não está disponível durante a inicialização do módulo.

**Segunda camada: Comandos condicionais por Feature Flag**

```typescript
// commands.ts:68-112 (fragmento de importação condicional)
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default
  : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Esses comandos usam a função `feature()` do `bun:bundle` para Dead Code Elimination, cortando diretamente os comandos não habilitados em tempo de build, não em tempo de execução.

**Terceira camada: Comandos somente para uso interno**

```typescript
// commands.ts:197-222 (INTERNAL_ONLY_COMMANDS)
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr,
  ctx_viz, goodClaude, issue, initVerifiers, mockLimits, bridgeKick,
  version, ultraplan, subscribePr, resetLimits, resetLimitsNonInteractive,
  onboarding, share, summary, teleport, antTrace, perfIssue,
  env, oauthRefresh, debugToolCall, agentsPlatform, autofixPr,
].filter(Boolean)
```

Esses comandos são registrados somente quando `USER_TYPE === 'ant'` (usuários internos da Anthropic) e em modo não-demo — é o mecanismo de isolamento de ferramentas internas e comandos de debug.

**Quarta camada: Comandos carregados dinamicamente**

```typescript
// commands.ts:360-395 (loadAllCommands)
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])

  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

Skills, comandos de plugin e comandos de workflow são carregados de forma assíncrona em paralelo, ordenados por prioridade: bundled skills têm maior prioridade, comandos embutidos têm a menor. Isso garante que comandos personalizados pelo usuário possam sobrepor (shadow) comandos embutidos de mesmo nome.

### Definição de Tipos de Comandos

`types/command.ts` define três tipos mutuamente exclusivos de comandos, formando o tipo union `Command`:

```typescript
// types/command.ts (tipo union central)
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

| Tipo | Descrição | Comandos Típicos |
|------|-----------|-----------------|
| `PromptCommand` | Expande para prompt e injeta no fluxo de conversa, executado pelo modelo | `/review`, `/skills`, todas as Skills |
| `LocalCommand` | Execução local síncrona pura, retorna resultado em texto | `/compact`, `/context` |
| `LocalJSXCommand` | Renderiza componentes Ink React UI | `/model`, `/resume`, `/config` |

`CommandBase` é o conjunto de campos compartilhados pelos três tipos:

```typescript
// types/command.ts (campos centrais do CommandBase)
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  availability?: CommandAvailability[]    // 'claude-ai' | 'console'
  isEnabled?: () => boolean               // verificação de feature flag em tempo de execução
  isHidden?: boolean                      // oculto no typeahead
  argumentHint?: string                   // texto de dica de parâmetro
  whenToUse?: string                      // descrição de cenário de chamada pelo modelo
  loadedFrom?: 'skills' | 'plugin' | 'bundled' | 'mcp' | ...
  immediate?: boolean                     // bypass de fila para execução imediata
  isSensitive?: boolean                   // parâmetros mascarados no histórico
}
```

### Categorização de Comandos (Embutidos vs. Plugin vs. Personalizados)

```
Hierarquia de fontes de comandos (prioridade do maior para o menor)
├── bundledSkills        # Skills embutidas empacotadas com o Claude Code
├── builtinPluginSkills  # Skills fornecidas por plugins embutidos habilitados
├── skillDirCommands     # Skills no diretório .claude/skills/ do usuário
├── workflowCommands     # Comandos de scripts de workflow (feature: WORKFLOW_SCRIPTS)
├── pluginCommands       # Comandos registrados por plugins de terceiros
├── pluginSkills         # Skills fornecidas por plugins de terceiros
└── COMMANDS()           # Comandos embutidos hardcoded (menor prioridade)
```

---

## 7.4 Lista Completa de Categorias de Comandos

A seguir, organizado com base na saída `ls` de `commands.ts` e na lista de registro.

### Gerenciamento de Sessão

| Comando | Descrição |
|---------|-----------|
| `/compact [instructions]` | Comprime histórico de conversa, liberando janela de contexto |
| `/resume` | Seleciona da lista de sessões históricas e restaura a conversa |
| `/branch [title]` | Faz fork de uma nova sessão a partir da conversa atual |
| `/rewind` | Retrocede para um nó histórico da conversa |
| `/clear` | Limpa os registros da conversa atual |
| `/session` | Exibe informações da sessão atual |
| `/rename` | Renomeia a sessão atual |
| `/summary` | Gera resumo da conversa atual (comando interno) |
| `/export` | Exporta conteúdo da conversa |
| `/copy` | Copia a última mensagem para a área de transferência |

### Ferramentas de Desenvolvimento

| Comando | Descrição |
|---------|-----------|
| `/review [PR#]` | Revisão de código local (chama `gh pr diff`) |
| `/ultrareview [PR#]` | Revisão de código profunda na nuvem (10-20 minutos, orientada por bughunter) |
| `/commit` | Faz commit das alterações de código (comando interno) |
| `/commit-push-pr` | Commit + Push + criação de PR (comando interno) |
| `/diff` | Visualiza o git diff atual |
| `/init` | Inicializa projeto (gera CLAUDE.md) |
| `/add-dir` | Adiciona diretório de trabalho extra |
| `/hooks` | Gerencia configurações de hooks de eventos |
| `/files` | Lista arquivos rastreados na sessão |
| `/pr_comments` | Visualiza comentários de PR |
| `/issue` | Cria/visualiza GitHub Issue (comando interno) |
| `/autofix-pr` | Corrige automaticamente problemas em PR (comando interno) |

### Configuração

| Comando | Descrição |
|---------|-----------|
| `/model [name]` | Alterna modelo da conversa (com seletor interativo) |
| `/config` | Visualiza/modifica itens de configuração |
| `/theme` | Alterna tema do terminal |
| `/vim` | Alterna modo de input vim |
| `/keybindings` | Gerencia ligações de atalhos |
| `/permissions` | Visualiza/modifica permissões de ferramentas |
| `/privacy-settings` | Gerencia configurações de privacidade |
| `/output-style` | Define preferências de formato de saída |
| `/effort` | Define nível de esforço de resposta |
| `/fast` | Alterna modo rápido |
| `/plan` | Alterna modo Plan (apenas planejar, não executar) |
| `/sandbox-toggle` | Alterna modo sandbox |

### Debug e Diagnóstico

| Comando | Descrição |
|---------|-----------|
| `/doctor` | Diagnostica problemas de configuração e ambiente |
| `/cost` | Exibe consumo de tokens e custos da sessão atual |
| `/context` | Exibe detalhes de uso da janela de contexto (tabela por categoria) |
| `/stats` | Exibe estatísticas de uso |
| `/usage` | Exibe informações de uso da API |
| `/insights` | Gera relatório de análise de uso histórico de sessões (carregamento tardio de módulo de 113KB) |
| `/heapdump` | Gera snapshot de heap de memória (para debug) |
| `/debug-tool-call` | Depura chamadas de ferramentas (comando interno) |
| `/perf-issue` | Registra problemas de desempenho (comando interno) |
| `/ant-trace` | Rastreamento interno da Anthropic (comando interno) |

### Identidade e Serviços

| Comando | Descrição |
|---------|-----------|
| `/login` | Login na conta Claude.ai |
| `/logout` | Sair do login |
| `/upgrade` | Upgrade para plano superior |
| `/install-github-app` | Instala GitHub App |
| `/install-slack-app` | Instala Slack App |
| `/ide` | Gerenciamento de integração com IDE |
| `/terminalSetup` | Configuração de integração com terminal |
| `/mobile` | Exibe QR code de conexão mobile |
| `/chrome` | Gerenciamento de extensão Chrome |
| `/desktop` | Gerenciamento de aplicação desktop |

### Funcionalidades Avançadas

| Comando | Descrição |
|---------|-----------|
| `/mcp` | Gerenciamento de servidores MCP (listar/iniciar/reiniciar) |
| `/skills` | Gerenciamento de Skills (listar/instalar/atualizar) |
| `/tasks` | Gerenciamento de tarefas em background |
| `/agents` | Gerenciamento de sub-agents |
| `/memory` | Gerenciamento de arquivos de memória de projeto (CLAUDE.md) |
| `/plan` | Entra no modo de planejamento |
| `/thinkback` | Retrocede o processo de raciocínio do modelo |
| `/thinkback-play` | Reproduz animação de retrocesso de raciocínio |
| `/advisor` | Modo de consultor IA |
| `/plugin` | Gerenciamento de plugins |
| `/reload-plugins` | Recarrega plugins |
| `/passes` | Gerenciamento de passes de revisão em múltiplas rodadas |
| `/feedback` | Envia feedback para a Anthropic |
| `/btw` | Adiciona mensagem de anotação |
| `/tag` | Etiqueta a conversa |
| `/stickers` | Exibe adesivos (Easter egg) |

Comandos condicionais por feature flag (invisíveis por padrão):

| Comando | Feature Flag | Descrição |
|---------|-------------|-----------|
| `/ultraplan` | `ULTRAPLAN` | Planejamento super avançado na nuvem (assíncrono de longa duração) |
| `/voice` | `VOICE_MODE` | Modo de input por voz |
| `/bridge` | `BRIDGE_MODE` | Modo de ponte de controle remoto |
| `/workflows` | `WORKFLOW_SCRIPTS` | Comandos de workflow por scripts |
| `/peers` | `UDS_INBOX` | Comunicação entre sessões pares |
| `/fork` | `FORK_SUBAGENT` | Criação explícita de sub-agent |
| `/buddy` | `BUDDY` | Modo de colaboração Buddy |

---

## 7.5 Fluxo de Execução de Comandos

### Caminho Completo da Entrada "/" pelo Usuário até a Execução do Comando

```
Usuário digita "/compact some instructions"
        │
        ▼
    Processador de input do REPL
    Detecta prefixo "/"
        │
        ▼
    getCommands(cwd)                    ← Agrega lista de comandos de todas as fontes
    findCommand("compact", commands)     ← Busca por name / aliases
        │
        ▼
    meetsAvailabilityRequirement(cmd)   ← Verifica controle de tipo de autenticação
    isCommandEnabled(cmd)               ← Verifica feature flag / isEnabled()
        │
        ├── Verifica cmd.immediate      ← true: bypass de fila para execução imediata
        │
        ▼
    processSlashCommand(cmd, "some instructions", context)
        │
        ├── type === 'local'     → cmd.load() → module.call(args, ctx)
        │                                        retorna LocalCommandResult
        │
        ├── type === 'local-jsx' → cmd.load() → Ink render(module.call(...))
        │                                        renderiza componente React no terminal
        │
        └── type === 'prompt'   → cmd.getPromptForCommand(args, ctx)
                                   retorna ContentBlockParam[]
                                   injeta no fluxo de conversa → dispara inferência do modelo
```

### Análise de Argumentos de Comandos

O sistema de comandos não possui um framework unificado de análise de argumentos — esta é uma escolha de design intencional. Cada comando processa seu parâmetro `args: string` de forma independente, mantendo grande flexibilidade:

- `/compact` usa diretamente `args.trim()` como instrução de compressão personalizada
- `/review` usa `/^\d+$/.test(prNumber)` para verificar se é um número de PR
- `/model` vai direto para `SetModelAndClose` quando há args, renderiza o `ModelPickerWrapper` interativo quando não há
- `/resume` suporta session ID (UUID), título personalizado, ou abre o seletor de lista sem parâmetros

Este design evita a complexidade de uma camada de análise unificada, ao custo de cada comando precisar lidar com seus próprios casos extremos.

### Renderização de Saída de Comandos

Os três tipos de `LocalCommandResult` correspondem a diferentes caminhos de renderização:

```typescript
// types/command.ts
export type LocalCommandResult =
  | { type: 'text'; value: string }       // renderiza como mensagem de texto
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
                                           // dispara lógica de substituição de contexto
  | { type: 'skip' }                      // não renderiza nada
```

`LocalJSXCommand` passa resultados para o REPL via callback `onDone()`:

```typescript
// types/command.ts (LocalJSXCommandOnDone)
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'   // modo de exibição da mensagem
    shouldQuery?: boolean                   // se dispara consulta ao modelo imediatamente
    metaMessages?: string[]                 // mensagens visíveis ao modelo mas não ao usuário
    nextInput?: string                      // preenche automaticamente o próximo input
    submitNextInput?: boolean               // se submete automaticamente
  },
) => void
```

`display: 'system'` exibe com estilo de mensagem de sistema (itálico cinza), `display: 'user'` exibe como mensagem de usuário comum, `display: 'skip'` não exibe nada.

---

## 7.6 Análise Aprofundada de Comandos Representativos

### Detalhes de Implementação do Comando /compact

`/compact` é um dos comandos com lógica mais complexa no sistema de comandos, assumindo a responsabilidade central de compressão do histórico de conversas.

**Árvore de decisão de execução** (`commands/compact/compact.ts`):

```
/compact [instructions]
    │
    ├── Há instruções personalizadas?
    │   └── Sem instrução → trySessionMemoryCompaction()   ← Tenta compressão por Session Memory primeiro
    │                  Sucesso retorna diretamente, caminho mais rápido
    │
    ├── isReactiveOnlyMode() ?
    │   └── Sim → compactViaReactive()               ← Caminho de compressão reativa (nova arquitetura)
    │               Execução paralela: executePreCompactHooks + getCacheSharingParams
    │               Chama: reactiveCompactOnPromptTooLong()
    │
    └── Não → Caminho de compressão tradicional
              microcompactMessages()                  ← Micro-comprime primeiro para reduzir tokens
              compactConversation()                   ← Compressão principal (geração de resumo)
              setLastSummarizedMessageId(undefined)   ← Reseta ponteiro de rastreamento
```

Ponto de design-chave: antes de comprimir, é obrigatório chamar `getMessagesAfterCompactBoundary(messages)` para filtrar mensagens já cortadas que o REPL mantém para scrollback da UI — essas mensagens não devem aparecer no resumo.

A sequência de limpeza após compressão bem-sucedida é fixa:
1. `setLastSummarizedMessageId(undefined)` — reseta ponteiro de mensagem
2. `suppressCompactWarning()` — suprime avisos de "contexto prestes a esgotar"
3. `getUserContext.cache.clear?.()` — limpa cache de contexto do usuário
4. `runPostCompactCleanup()` — dispara hooks pós-compressão

**O caminho Reactive Compact** aproveita otimização paralela:

```typescript
// compact.ts:compactViaReactive (segmento paralelo central)
const [hookResult, cacheSafeParams] = await Promise.all([
  executePreCompactHooks(...),      // executa hooks pré-compressão (pode iniciar subprocessos)
  getCacheSharingParams(context, messages),  // constrói system prompt (percorre todas as ferramentas)
])
```

Os dois são independentes entre si; a execução paralela reduz significativamente o tempo de espera.

### Lógica de Troca de Modelo do Comando /model

`/model` é do tipo `local-jsx`, renderizando um seletor interativo via componente React.

**Dois caminhos de execução**:

- **Com argumentos** (`/model claude-sonnet-4-6`): Renderiza o componente `SetModelAndClose`, executa validação de modelo assincronamente no `useEffect`, completa imediatamente via `onDone()`
- **Sem argumentos** (`/model`): Renderiza o componente `ModelPickerWrapper`, exibindo a interface interativa completa do `ModelPicker`

**Atualização de estado na troca de modelo**:

```typescript
// model.tsx:handleSelect (atualização central de estado)
setAppState(prev => ({
  ...prev,
  mainLoopModel: model,
  mainLoopModelForSession: null    // limpa override temporário de nível de sessão
}))
```

**Hierarquia de validação de modelo** (da mais rápida à mais lenta):
1. Verifica `isModelAllowed(model)` — whitelist de restrições organizacionais
2. Verifica `isOpus1mUnavailable(model)` — verificação de privilégio para contexto de 1M
3. Verifica `isKnownAlias(model)` — alias conhecido passa diretamente (pula validação via API)
4. `validateModel(model)` — chama a API para validar nome de modelo personalizado

Fast Mode e a troca de modelo têm interação: se o novo modelo não suporta Fast Mode, é desligado automaticamente; se suporta e já está habilitado, exibe "Fast mode ON" na mensagem de confirmação.

### Fluxo de Revisão de Código do Comando /review

`/review` demonstra o uso típico do tipo `PromptCommand` — um template de prompt conciso conduzindo um fluxo completo de revisão:

```typescript
// review.ts:LOCAL_REVIEW_PROMPT (template de prompt completo)
const LOCAL_REVIEW_PROMPT = (args: string) => `
  You are an expert code reviewer. Follow these steps:
  1. If no PR number is provided, run \`gh pr list\` to show open PRs
  2. If a PR number is provided, run \`gh pr view <number>\` to get PR details
  3. Run \`gh pr diff <number>\` to get the diff
  4. Analyze the changes and provide a thorough code review...
  PR number: ${args}
`
```

O próprio comando tem apenas 4 linhas de código-chave, o restante é tudo feito pelo modelo — esta é exatamente a filosofia de design do `PromptCommand`: **o comando define O QUÊ, o modelo decide COMO**.

Em contraste, `/ultrareview` (tipo `local-jsx`) executa um caminho completamente diferente:

```
/ultrareview [PR#]
    │
    ├── checkOverageGate()             ← Verifica cota grátis / saldo Extra Usage
    │   ├── Team/Enterprise → passa diretamente
    │   ├── Tem cota grátis → passa, com aviso
    │   └── Cota esgotada → exibe diálogo de confirmação de excedente
    │
    └── launchRemoteReview()
        ├── Modo PR → teleportToRemote(branchName: "refs/pull/N/head")
        └── Modo branch → git merge-base → verificação git diff → teleportToRemote(useBundle: true)
                        → registerRemoteAgentTask()
                        → retorna URL da tarefa, modelo notifica o usuário
```

`/ultrareview` "teletransporta" a tarefa de revisão de código para rodar na nuvem, registra `RemoteAgentTask` localmente e retorna imediatamente, recebendo resultados via mecanismo de polling — este é um padrão de delegação assíncrona de tarefas, completamente diferente do modelo de execução síncrona de comandos locais.

---

## 7.7 Fronteiras entre Comandos e Skills

### Semelhanças e Diferenças entre os Dois

| Dimensão | Comando (Command) | Skill |
|----------|-------------------|-------|
| Forma de definição | Código TypeScript, lógica hardcoded | Arquivo Markdown, frontmatter + conteúdo de prompt |
| Momento de carregamento | Registro estático na inicialização (embutido) ou carregamento assíncrono (plugin) | Escaneamento do sistema de arquivos em tempo de execução |
| Tipo de execução | `local` / `local-jsx` / `prompt` | Somente `prompt` (expande para prompt) |
| Chamável pelo modelo | A maioria dos comandos embutidos proíbe chamada pelo modelo (`source: 'builtin'`) | Projetado para suportar chamada pelo modelo via SkillTool |
| Visibilidade para o usuário | Todos os comandos aparecem no typeahead `/` | Depende de `userInvocable` e `hasUserSpecifiedDescription` |
| Consciência de contexto | Acessa estado completo da aplicação via `ToolUseContext` | Só pode usar conteúdo de prompt, sem acesso direto ao estado |
| Identificador de fonte | `source: 'builtin'` | `loadedFrom: 'skills' \| 'bundled' \| 'plugin'` |

### Considerações por Trás das Escolhas de Design

**Por Que Comandos Embutidos Não Usam Markdown Skill?**

Comandos embutidos precisam acessar o estado da aplicação (`AppState`), chamar APIs Node.js (sistema de arquivos, criptografia) e renderizar componentes React — capacidades muito além do que templates de prompt podem expressar. `/compact` precisa chamar 4 estratégias diferentes de compressão; `/model` precisa renderizar UI interativa; `/resume` precisa ler e escrever arquivos de sessão. Todos esses devem ser código.

**A lógica de filtragem do SkillTool** revela a delimitação precisa das fronteiras:

```typescript
// commands.ts:getSkillToolCommands
export const getSkillToolCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const allCommands = await getCommands(cwd)
    return allCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        !cmd.disableModelInvocation &&
        cmd.source !== 'builtin' &&    // ← comandos embutidos são excluídos
        (cmd.loadedFrom === 'bundled' ||
          cmd.loadedFrom === 'skills' ||
          cmd.loadedFrom === 'commands_DEPRECATED' ||
          cmd.hasUserSpecifiedDescription ||
          cmd.whenToUse),
    )
  },
)
```

**`source !== 'builtin'`** é a regra central: comandos embutidos são explicitamente excluídos da lista chamável pelo modelo. Isso previne que o modelo bypasse verificações de permissão para manipular diretamente o estado da sessão via SkillTool.

**O conjunto de comandos seguros para remoto (REMOTE_SAFE_COMMANDS)** refina ainda mais esta fronteira:

```typescript
// commands.ts:REMOTE_SAFE_COMMANDS
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim,
  cost, usage, copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])
```

Apenas 20 comandos estão disponíveis no modo `--remote` — esses comandos não dependem do sistema de arquivos local, git ou IDE; são operações de estado TUI puras que podem ser executadas com segurança em sessões de bridge remoto.

---

## 7.8 Análise de Decisões de Design

**Decisão 1: Três Tipos de Comandos em Vez de Interface Unificada**

A divisão em três partes `local` / `local-jsx` / `prompt` pode parecer adicionar complexidade, mas cada tipo resolve um problema central diferente:
- `local` lida com operações com efeitos colaterais mas sem UI (precisa retornar dados estruturados)
- `local-jsx` lida com operações que precisam de interface interativa (dependendo da árvore de renderização Ink)
- `prompt` lida com operações que podem ser delegadas ao modelo (menor acoplamento)

Se forçado a unificar em uma única interface, ou todos os comandos precisariam lidar com renderização React (dependência desnecessária), ou a segurança de tipos seria perdida.

**Decisão 2: memoize por cwd em Vez de Singleton Global**

`loadAllCommands = memoize(async (cwd: string) => ...)` usa o diretório de trabalho como chave de cache, significando que instâncias do Claude Code em diferentes diretórios têm caches de comandos independentes. Isso suporta a necessidade de monorepos e cenários multi-projeto onde cada diretório tem seu próprio conjunto de Skills.

**Decisão 3: Não Usar Análise de Argumentos Unificada**

Esta é uma "design flexível" intencional. Um framework de análise unificado (como commander.js) forçaria cada comando a declarar um schema de argumentos completo, o que seria sem sentido para comandos como `/compact` que usam "instruções em texto livre". Manter a string bruta e deixar cada comando decidir como analisar troca consistência por flexibilidade.

**Decisão 4: Dois Níveis de Controle de Disponibilidade vs. isEnabled**

Os dois níveis de controle resolvem problemas de visibilidade em ciclos de vida diferentes:
- `availability` filtra durante a construção da lista de comandos, resultado em cache, adequado para verificações de tipo de autenticação estáticas
- `isEnabled()` é reavaliado a cada chamada de `getCommands()` (sem cache), adequado para verificações dinâmicas de feature flags

O comentário no código observa especificamente que `isEnabled()` não é memoizado: após a execução de `/login`, o estado de autenticação muda e deve ser refletido imediatamente na lista de comandos.

**Decisão 5: Comandos Internos Não Gerenciados em Pacote Separado**

`INTERNAL_ONLY_COMMANDS` controla visibilidade diretamente via variável de ambiente `USER_TYPE === 'ant'`, em vez de através de um pacote npm separado. Isso simplifica a complexidade de build, ao custo de que builds externos precisam eliminar esta parte de código via Dead Code Elimination (`filter(Boolean)` também é eficaz para comandos condicionais `null`).

---

## 7.9 Padrões Transferíveis

### Padrão 1: Divisão em Três Tipos de Comandos

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**Cenários de aplicação**: Qualquer sistema de comandos que precisa suportar simultaneamente "lógica pura", "interação de UI" e "delegação para LLM". Os limites entre os três são muito claros e podem ser portados diretamente para outros frameworks REPL/CLI.

**Valor central**: O sistema de tipos reforça a separação de responsabilidades, sem necessidade de verificações `instanceof` em tempo de execução.

### Padrão 2: Carregamento Tardio + memoize por cwd

```typescript
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })
```

**Cenários de aplicação**: Ferramentas CLI com grande número de comandos (>50) ou onde alguns comandos dependem de módulos pesados.

**Pontos de implementação**: A chave de memoize deve incluir todos os fatores que afetam o conjunto de comandos (aqui é cwd), e o momento de invalidação do cache deve corresponder a mudanças reais de estado (aqui é `clearCommandsCache()`).

### Padrão 3: Agregação Multi-Fonte de Comandos + Ordenação por Prioridade

```typescript
return [
  ...bundledSkills,       // maior prioridade (pode sobrepor comandos embutidos de mesmo nome)
  ...pluginCommands,
  ...COMMANDS(),          // menor prioridade (pode ser sobreposto)
]
```

**Cenários de aplicação**: Ferramentas CLI que suportam ecossistema de plugins, onde extensões de terceiros precisam poder sobrepor (override) comportamentos embutidos.

**Observação**: `findCommand` retorna o primeiro item correspondente na lista, portanto a ordem do array é a ordem de prioridade — isso precisa ser documentado claramente no design.

### Padrão 4: Visibilidade de Comandos Auth-gated

```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai': if (isClaudeAISubscriber()) return true; break
      case 'console':   if (!isUsing3PServices() && ...) return true; break
    }
  }
  return false
}
```

**Cenários de aplicação**: Produtos SaaS que precisam exibir diferentes conjuntos de funcionalidades para usuários de diferentes camadas de assinatura.

**Design-chave**: Interceptar na fase de filtragem da lista de comandos, não reportar erro na fase de execução — usuários não veem comandos que não podem usar, reduzindo a carga cognitiva.

### Padrão 5: Whitelist de Bridge Safe / Remote Safe

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([session, exit, clear, ...])
```

**Cenários de aplicação**: Sistemas de comandos que precisam rodar em ambientes restritos (sessões remotas, sandbox, bridge mobile).

**Lógica de implementação**: Mais seguro que blacklist — novos comandos adicionados são indisponíveis em ambientes restritos por padrão, precisando ser explicitamente adicionados à whitelist. Isso evita que comandos sensíveis sejam expostos no ambiente errado por descuido.

---

## 7.10 Índice de Código-Fonte

| Caminho do Arquivo | Linhas | Conteúdo |
|-------------------|--------|----------|
| `src/commands.ts` | 754 | Toda a lógica de entrada de registro, agregação, filtragem e busca de comandos |
| `src/types/command.ts` | ~250 | Definição do tipo union Command, CommandBase, declarações detalhadas de cada subtipo |
| `src/commands/compact/compact.ts` | 287 | Implementação de três caminhos do /compact (session memory / reactive / traditional) |
| `src/commands/model/model.tsx` | 296 | Dois caminhos do /model: seletor interativo + configuração direta, saída compilada pelo React Compiler |
| `src/commands/review.ts` | ~50 | Entradas de /review (tipo prompt) e /ultrareview (tipo local-jsx) |
| `src/commands/review/reviewRemote.ts` | 316 | Lógica de inicialização remota do /ultrareview: teleport, overage gate, registro de tarefas |
| `src/commands/resume/resume.tsx` | 274 | UI do seletor de lista de sessões do /resume |
| `src/commands/branch/branch.ts` | 296 | Fork de conversa do /branch: cópia JSONL, reescrita de sessionId, tratamento de conflitos |
| `src/commands/context/context-noninteractive.ts` | 325 | Caminho não-interativo do /context: estatísticas de tokens por categoria, renderização de tabela Markdown |
| `src/skills/loadSkillsDir.ts` | — | Lógica de escaneamento e carregamento dinâmico de diretório de Skills |
| `src/skills/bundledSkills.ts` | — | Registro de Skills embutidas empacotadas com o produto |
| `src/plugins/builtinPlugins.ts` | — | Extração de comandos Skill de plugins embutidos |
| `src/utils/plugins/loadPluginCommands.ts` | — | Carregamento e cache de comandos de plugins de terceiros |

**Índice de Funções-Chave**:

| Função | Arquivo | Uso |
|--------|---------|-----|
| `getCommands(cwd)` | commands.ts | Retorna todos os comandos disponíveis para o usuário atual (entrada principal) |
| `findCommand(name, commands)` | commands.ts | Busca comando por nome/alias |
| `meetsAvailabilityRequirement(cmd)` | commands.ts | Verificação de controle de tipo de autenticação |
| `getSkillToolCommands(cwd)` | commands.ts | Retorna conjunto de comandos Skill chamáveis pelo modelo |
| `getSlashCommandToolSkills(cwd)` | commands.ts | Retorna conjunto de Skills que o usuário pode acionar via / |
| `isBridgeSafeCommand(cmd)` | commands.ts | Determina se o comando pode ser executado no modo bridge |
| `formatDescriptionWithSource(cmd)` | commands.ts | Formatação de descrição com anotação de fonte na interface do usuário |
| `clearCommandsCache()` | commands.ts | Limpa todo o cache de comandos (incluindo Skills e plugins) |
