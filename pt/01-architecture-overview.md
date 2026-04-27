# Capítulo 1: Visão Geral da Arquitetura e Fluxo de Inicialização

> Fonte de dados: Snapshot do código-fonte TypeScript do Claude Code (2026-03-31)
> Caminho do código-fonte (mini): `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`

---

## 1.1 Visão Geral e Posicionamento

**O que é o Claude Code:** O Claude Code é um assistente de programação de IA executado no terminal, que renderiza uma TUI (Terminal User Interface) interativa com React/Ink, acionando a Claude API em um loop REPL para realizar tarefas de desenvolvimento como edição de código, execução de comandos e operações em arquivos.

### Visão Geral da Stack Tecnológica

| Camada | Tecnologia | Finalidade |
|------|------|------|
| Runtime | Bun (principal) / Node.js 18+ (compatibilidade) | Ambiente de execução JavaScript |
| Linguagem | TypeScript | Tipagem estrita em todo o projeto |
| Framework UI | React + Ink | Renderização de TUI no terminal |
| Framework CLI | Commander.js (`@commander-js/extra-typings`) | Análise de argumentos de linha de comando |
| Cliente API | `@anthropic-ai/sdk` | Chamadas à Claude API |
| Integração MCP | `@modelcontextprotocol/sdk` | Protocolo de servidor MCP |
| Feature Flags | GrowthBook + `bun:bundle` feature flags | Testes A/B e DCE |
| Telemetria | OpenTelemetry (carregamento lazy ~400KB) | Métricas/logs/rastreamento |
| Validação | Zod v4 | Validação de schema em tempo de execução |

### Estatísticas de Tamanho do Código

- **Total de linhas**: 512.664 (arquivos `.ts` + `.tsx`)
- **Número de arquivos**: 1.884 arquivos TypeScript
- **Número de diretórios de nível superior**: 35

Proporção de LOC por principais diretórios:

```
utils/       180.472 linhas  (35,2%)  — funções utilitárias, permissões, autenticação, configurações etc.
components/   81.546 linhas  (15,9%)  — componentes React de UI
services/     53.680 linhas  (10,5%)  — serviços de API, MCP, analytics, memória etc.
tools/        50.828 linhas  (9,9%)   — 30 implementações de ferramentas (Bash/File/Agent etc.)
commands/     26.428 linhas  (5,2%)   — implementações de comandos slash
screens/       5.977 linhas  (1,2%)   — telas de nível superior como REPL
bootstrap/     ~5.000 linhas  (1,0%)  — estado global (state.ts 1.758 linhas)
entrypoints/   ~3.000 linhas  (0,6%)  — pontos de entrada CLI/SDK/MCP
main.tsx       4.683 linhas  (0,9%)   — coordenador do ponto de entrada principal
setup.ts         477 linhas  (0,1%)   — configuração de inicialização
```

---

## 1.2 Fundamentos Teóricos

### Padrões de Arquitetura para Aplicações de Linha de Comando

O Claude Code combina dois padrões clássicos de arquitetura CLI:

**REPL Loop (Read-Eval-Print Loop)**
O REPL tradicional lê entradas, avalia e imprime saídas em um loop síncrono. O Claude Code o eleva a um REPL assíncrono orientado a eventos: a entrada é capturada por componentes React, a "avaliação" é uma round-trip da Claude API (incluindo múltiplas chamadas de ferramentas) e a saída é renderizada no terminal via React/Ink reconciler.

**Event-Driven Architecture**
A inicialização não bloqueia esperando que toda a inicialização seja concluída — leitura de MDM, pré-busca do Keychain, conexões MCP e carregamento de plugin hooks são todos acionados em paralelo no estilo fire-and-forget (veja a seção 1.4). Isso minimiza o TTFR (Time To First Render), consistente com a filosofia de otimização do Critical Rendering Path em aplicações web.

### Filosofia de Design do Framework de UI de Terminal: React no Terminal

O Ink transporta o modelo de componentes do React, o estado declarativo e o mecanismo de reconciliation para o terminal. A ideia central:

- **DOM virtual → buffer virtual de terminal**: cada mudança de estado aciona um diff, redesenhando apenas as linhas de caracteres modificadas, evitando flickering
- **Flexbox → layout de terminal**: usa o mecanismo CSS Yoga para calcular larguras de colunas e quebras de linha, permitindo que a UI de terminal seja descrita de forma declarativa com JSX
- **Reutilização de componentes**: loading spinner, caixas de diálogo de confirmação, exibição de Diff e outras lógicas de UI são encapsuladas como componentes React testáveis

Isso permite que o código de UI do Claude Code compartilhe o mesmo quadro cognitivo com código de frontend web, e as 81.546 linhas de código no diretório `components/` podem ser compreendidas com padrões React familiares.

### Fundamentos Teóricos da Arquitetura de Plugins

O sistema de plugins do Claude Code é baseado no padrão Capability Registration:

- Ferramentas (Tools), Comandos (Commands) e Hooks são todos registrados em um registro global na inicialização
- Plugins estendem listas de ferramentas/comandos por convenção do sistema de arquivos (`~/.claude/plugins/`)
- A função `feature()` do `bun:bundle` realiza Dead Code Elimination (DCE) em tempo de compilação; funcionalidades experimentais não aparecem em produtos de build externos

---

## 1.3 Diagrama Geral da Arquitetura

### Arquitetura em Camadas (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                    Camada de Entrada (Entry Layer)        │
│  main.tsx  │  entrypoints/cli.tsx  │  entrypoints/mcp.ts │
│  (inter. CLI)  (roteamento Commander.js)  (modo MCP server)│
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Camada de Bootstrap (Bootstrap Layer)    │
│    setup.ts      │    entrypoints/init.ts                 │
│  (inicializ. de   │    bootstrap/state.ts                 │
│   sessão)         │    (singleton de estado global)        │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Camada de UI (Ink/React TUI)              │
│  screens/REPL.tsx  │  components/App.tsx                  │
│  (interface princ.) │  components/ (81K LOC)               │
│  replLauncher.tsx  │  (entrada/saída/diálogos/animações)  │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Camada de Motor (Engine Layer)            │
│  QueryEngine.ts  │  query.ts  │  state/AppStateStore.ts   │
│  (gestão do ciclo│  (chamadas  │  (árvore de estado React) │
│   de vida da     │   de API)   │                          │
│   sessão)        │             │                          │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Camada de Ferramentas (Tool Layer)        │
│  tools/ (30 ferramentas, 50K LOC)                         │
│  BashTool │ FileReadTool │ FileEditTool │ AgentTool        │
│  GlobTool │ GrepTool     │ SkillTool    │ MCPTool           │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  Camada de Serviços (Service Layer)        │
│  services/ (53K LOC)                                      │
│  api/         │ mcp/          │ analytics/                 │
│  (Claude API)   (cliente MCP)   (GrowthBook/OTel)          │
│  lsp/         │ SessionMemory │ remoteManagedSettings      │
│  (servidor de   (memória de     (config. gerenciada         │
│   linguagem)    sessão)         empresarial)                │
└─────────────────────────────────────────────────────────┘
```

### Visão Geral das Dependências entre Módulos

```
main.tsx
  ├── entrypoints/init.ts       (memoized, inicializado apenas uma vez)
  ├── entrypoints/cli.tsx       (roteamento de subcomandos Commander)
  ├── bootstrap/state.ts        (estado global, proibido importar ciclicamente)
  ├── setup.ts                  (chamado em cada sessão)
  ├── QueryEngine.ts            (caminho headless/SDK)
  ├── replLauncher.tsx          (caminho interativo)
  │     └── screens/REPL.tsx
  │           └── components/App.tsx
  └── services/mcp/client.ts   (carregamento de ferramentas/recursos MCP)
```

**O papel especial do `bootstrap/state.ts`**: O código contém o comentário explícito `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`, e há uma regra ESLint `custom-rules/bootstrap-isolation` impedindo que esse arquivo seja importado por módulos não-folha, prevenindo dependências circulares.

### Comparação dos Três Pontos de Entrada

| Entrada | Arquivo | Acionamento | Características |
|------|------|----------|------|
| CLI interativa | `entrypoints/cli.tsx` | Comando `claude` | REPL completo + React TUI |
| SDK headless | `QueryEngine.ts` | Flag `-p` / API do SDK | Sem UI, saída única ou em streaming |
| Servidor MCP | `entrypoints/mcp.ts` | `claude --mcp` | Expõe conjunto de ferramentas como MCP server |

---

## 1.4 Detalhamento do Fluxo de Inicialização

### Sequência Completa de Inicialização do main.tsx

As 4.683 linhas do `main.tsx` não são executadas sequencialmente — os efeitos colaterais dos imports no topo do arquivo formam uma sequência cuidadosamente orquestrada de pré-aquecimento paralelo.

**Fase 0: Período de carregamento de módulos (efeitos colaterais de import, ~135ms)**

```typescript
// main.tsx:9-12
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 1. Ponto de referência de desempenho

// main.tsx:13-16
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();                    // 2. Paralelo: subprocesso MDM (plutil/reg query)

// main.tsx:17-20
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();              // 3. Paralelo: pré-leitura macOS Keychain (OAuth + chave de API)

// main.tsx:209
profileCheckpoint('main_tsx_imports_loaded');  // Todos os imports concluídos
```

Os comentários explicam precisamente o benefício dessas três operações paralelas: a leitura de MDM economiza ~135ms de tempo de avaliação de módulos; a pré-leitura do Keychain economiza ~65ms de spawn síncrono sequencial. Este é o truque central de otimização de inicialização do Claude Code: **aproveitar a análise estática de módulos ES para executar operações intensivas de I/O durante a avaliação do grafo de módulos**.

**Fase 1: Roteamento Commander (síncrono)**

O Commander.js em `entrypoints/cli.tsx` analisa argv e, com base no subcomando (`chat`, `api`, `mcp`, `resume` etc.) ou flag, despacha para diferentes caminhos de execução:

```typescript
// entrypoints/cli.tsx (estrutura simplificada)
async function main(): Promise<void> {
  // Caminho rápido: --version com zero imports
  // Caminho normal: await init() → setup() → despacho para ramificação
}
```

**Fase 2: Inicialização com init() (memoized, executa apenas uma vez)**

A função `init` em `entrypoints/init.ts` é envolvida com `memoize`, garantindo que múltiplas chamadas resultem em apenas uma inicialização:

```typescript
// entrypoints/init.ts
export const init = memoize(async (): Promise<void> => {
  enableConfigs()                    // ativação do sistema de configuração
  applySafeConfigEnvironmentVariables()  // env vars seguras antes de conversas de confiança
  applyExtraCACertsFromConfig()     // configura certificados CA antes de conexões TLS
  setupGracefulShutdown()           // registra hooks de limpeza na saída
  // carregamento lazy: OpenTelemetry (~400KB) + gRPC (~700KB)
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(...)
  populateOAuthAccountInfoIfNeeded() // fire-and-forget
  initJetBrainsDetection()           // cache assíncrono
  detectCurrentRepository()          // detecção de repositório GitHub
  preconnectAnthropicApi()           // pré-conexão TCP+TLS (~100-200ms de sobreposição)
  configureGlobalMTLS()
  configureGlobalAgents()            // configuração de proxy
})
```

**Fase 3: Inicialização de sessão com setup() (chamado a cada sessão)**

```typescript
// setup.ts — sequência de passos-chave
export async function setup(cwd, permissionMode, ...): Promise<void> {
  // 1. Servidor de mensagens UDS (modo swarm/ant)
  if (feature('UDS_INBOX')) await startUdsMessaging(...)
  // 2. Verificação de backup de terminal (iTerm2/Terminal.app)
  if (!getIsNonInteractiveSession()) { ... }
  // 3. setCwd() — deve ser chamado antes de qualquer código que dependa de cwd
  setCwd(cwd)
  // 4. Snapshot de configuração de Hooks (deve ser após setCwd())
  captureHooksConfigSnapshot()
  // 5. Criação de Worktree (se --worktree)
  if (worktreeEnabled) { await createWorktreeForSession(...) }
  // 6. Registro de tarefas em background (SessionMemory, colapso de contexto)
  if (!isBareMode()) initSessionMemory()
  // 7. Pré-busca de Plugin (paralela, não bloqueante)
  void getCommands(getProjectRoot())
  void import('./utils/plugins/loadPluginHooks.js').then(m => m.loadPluginHooks())
  // 8. Ativação de sinks de análise + primeiro evento de telemetria
  initSinks()
  logEvent('tengu_started', {})
  // 9. Verificação de release notes (modo interativo)
  if (!isBareMode()) await checkForReleaseNotes(...)
}
```

**Fase 4: Renderização REPL**

```typescript
// replLauncher.tsx
export async function launchRepl(root, appProps, replProps, renderAndRun) {
  const { App } = await import('./components/App.js')   // carregamento lazy da UI
  const { REPL } = await import('./screens/REPL.js')
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>)
}
```

Por fim, o Ink assume o terminal, a árvore de componentes React começa a renderizar e o REPL está pronto.

### Estratégia de Pré-busca Paralela

A otimização de inicialização do Claude Code segue o princípio de "**acionar o quanto antes, aguardar o quanto mais tarde possível**":

| Operação | Momento de acionamento | Momento de aguardo |
|------|----------|----------|
| Subprocesso MDM (`plutil/reg query`) | 1ª linha de efeito colateral de import em `main.tsx` | Antes de `applySafeConfigEnvironmentVariables()` |
| Pré-leitura do Keychain (OAuth + chave de API) | 3ª linha de efeito colateral de import em `main.tsx` | `ensureKeychainPrefetchCompleted()` |
| Pré-conexão TCP à Claude API | `preconnectAnthropicApi()` dentro de `init()` | Conexão reutilizada automaticamente na primeira requisição de API |
| Carregamento de plugin hooks | Fire-and-forget dentro de `setup()` | `processSessionStartHooks()` antes da renderização |
| Leitura de configs MCP | Início de `getClaudeCodeMcpConfigs()` | `getMcpToolsCommandsAndResources()` no modo interativo |

### Mecanismo de Carregamento Lazy

O Claude Code realiza carregamento lazy explícito de módulos grandes no caminho crítico de inicialização:

```typescript
// entrypoints/init.ts — comentário sobre carregamento lazy do OpenTelemetry
// initializeTelemetry is loaded lazily via import() in setMeterState() to defer
// ~400KB of OpenTelemetry + protobuf modules until telemetry is actually initialized.
// gRPC exporters (~700KB via @grpc/grpc-js) are further lazy-loaded within instrumentation.ts.

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  ...
}
```

Além disso, `replLauncher.tsx` só faz `import` dos componentes App e REPL no último momento, evitando que a árvore React seja avaliada antes do roteamento do Commander estar completo.

A função `feature()` do `bun:bundle` implementa DCE em tempo de compilação:

```typescript
// main.tsx:76-80
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null;
```

Em builds externos, esse código é completamente removido, reduzindo o tamanho do pacote.

### Detalhamento dos Passos de Inicialização do setup.ts

As 477 linhas do `setup.ts` giram em torno das seguintes restrições-chave:

1. **`setCwd()` deve ser chamado primeiro**: todas as operações subsequentes (hooks, configurações, carregamento de plugins) dependem do cwd correto
2. **O snapshot de Hooks deve ser após `setCwd()`**: garante a leitura do `.claude/settings.json` do diretório correto
3. **A criação de Worktree deve ser antes de `getCommands()`**: caso contrário, o comando `/eject` fica indisponível
4. **`initSinks()` deve ser após o registro de todas as tarefas em background**: garante que a fila de eventos de análise já esteja pronta

O modo `--bare` (chamadas headless de script/SDK) ignora uma grande quantidade de funcionalidades interativas: verificação de backup de terminal, pré-busca de plugin hooks, commit attribution, team memory watcher etc., minimizando o custo de inicialização de chamadas de script.

### Construção do Estado em bootstrap/state.ts

`state.ts` (1.758 linhas) mantém o estado global singleton de toda a sessão. O tipo central `State` abrange:

```typescript
// bootstrap/state.ts (definição parcial do tipo State)
type State = {
  originalCwd: string
  projectRoot: string          // raiz estável do projeto; não muda com worktree
  totalCostUSD: number
  cwd: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  isInteractive: boolean
  sessionId: SessionId
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  invokedSkills: Map<string, { skillName, skillPath, content, invokedAt, agentId }>
  // contadores de telemetria
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  // provedores de log/rastreamento
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null
  // ... ~60 campos no total
}
```

**Restrição de design**: O comentário `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` é uma guarda arquitetural. A regra ESLint `custom-rules/bootstrap-isolation` impede que `state.ts` seja importado por módulos que causariam dependências circulares. Todo o estado é acessado via funções setter/getter, sem expor objetos mutáveis diretamente.

---

## 1.5 Análise dos Pontos de Entrada

### Ponto de Entrada CLI (modo interativo)

`entrypoints/cli.tsx` é o ponto de entrada mais complexo, responsável por todo o roteamento de funcionalidades voltadas ao usuário:

**Caminho de inicialização**:
1. Commander.js analisa argv → identifica subcomando ou flag
2. `await init()` inicializa (memoized)
3. Processa configs MCP, políticas empresariais, integração Chrome
4. `await setup(cwd, permissionMode, ...)` inicializa a sessão
5. Ramifica com base no modo:
   - **Modo interativo**: `showSetupScreens()` → `launchRepl()` → React TUI
   - **Modo Print (`-p`)**: `runHeadless()` → `QueryEngine` → stdout
   - **Modo Resume**: `loadConversationForResume()` → retoma sessão histórica
   - **Modo Teleport**: assunção de sessão remota

**Opções CLI principais** (parcial):

| Flag | Funcionalidade |
|------|------|
| `--permission-mode` | `auto`/`manual`/`bypassPermissions` |
| `--mcp-config` | Configuração dinâmica de servidor MCP |
| `--worktree` | Cria isolamento git worktree |
| `--tmux` | Executa em sessão tmux |
| `--model` | Sobrepõe o modelo do loop principal |
| `--resume` | Retoma sessão histórica |

### Ponto de Entrada SDK (API programática)

Quando chamado via flag `-p` ou API programática do SDK, contorna o React TUI e entra diretamente no `QueryEngine.ts`:

- `isNonInteractiveSession = true`
- Ignora toda a renderização de UI (Ink)
- Saída em streaming para stdout via tipo `SDKMessage`
- Suporta saídas estruturadas como `SDKStatus`, `SDKPermissionDenial`, `SDKCompactBoundaryMessage`

O modo SDK também tem beta features exclusivas: `entrypoints/sdk/coreSchemas.ts` define schemas de entrada/saída JSON estruturado, e `entrypoints/agentSdkTypes.ts` define tipos exclusivos do SDK como `HookEvent`, `ModelUsage` etc.

### Ponto de Entrada MCP (modo servidor MCP)

```typescript
// entrypoints/mcp.ts
export async function startMCPServer(cwd, debug, verbose): Promise<void> {
  const server = new Server(
    { name: 'claude/tengu', version: MACRO.VERSION },
    { capabilities: { tools: {} } }
  )
  // ListTools: expõe todas as ferramentas do Claude Code como MCP tools
  server.setRequestHandler(ListToolsRequestSchema, async () => {
    const tools = getTools(toolPermissionContext)
    return { tools: await Promise.all(tools.map(async tool => ({
      ...tool,
      description: await tool.prompt(...),
      inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
    }))) }
  })
  // CallTool: delega execução para a implementação de Tool correspondente
  server.setRequestHandler(CallToolRequestSchema, async ({ params }) => {
    const tool = findToolByName(tools, params.name)
    const finalResult = await tool.call(args, toolUseContext, ...)
    return { content: [{ type: 'text', text: result }] }
  })
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

O modo MCP expõe reversamente todo o conjunto de ferramentas do Claude Code (BashTool, FileReadTool, GrepTool etc.) para clientes MCP externos, realizando "Claude Code como MCP server".

### Lógica Compartilhada pelos Três Pontos de Entrada

Independentemente do ponto de entrada, todos compartilham:
- Estado global `bootstrap/state.ts`
- Inicialização `entrypoints/init.ts` (garantida por memoize para executar apenas uma vez)
- Registro de ferramentas `Tool.ts`
- Todos os serviços em `services/` (cliente de API, sistema de permissões etc.)
- Sistema de ciclo de vida de Hooks

A diferença está em se renderizam ou não o React TUI e no formato de saída (texto interativo vs. JSON estruturado).

---

## 1.6 Análise das Decisões de Design

### Por que Bun em vez de Node.js

A partir do código, podemos observar as características de uso do Bun:

1. **Função `feature()` do `bun:bundle`**: Este é o mecanismo de feature flag em tempo de compilação exclusivo do Bun, com suporte a Dead Code Elimination. Amplamente usado em `main.tsx` (COORDINATOR_MODE, KAIROS, CHICAGO_MCP, UDS_INBOX etc.); builds externos removem completamente esse código experimental.

2. **API WebView do Bun** (referência condicional): `typeof Bun !== 'undefined' && 'WebView' in Bun`, indicando que algumas funcionalidades dependem de APIs exclusivas do Bun.

3. **Executável de arquivo único do Bun**: Os comentários mencionam `Bun has an issue with single-file executables where application arguments from process.argv leak into process.execArgv`, indicando que o produto de lançamento é um executável de arquivo único compilado com Bun.

4. **Desempenho**: A velocidade de inicialização e carregamento de módulos do Bun é significativamente superior à do Node.js, crucial para o TTFR de ferramentas CLI.

Ao mesmo tempo, mantém compatibilidade com Node.js 18+ (há verificação de versão do Node em `setup.ts`), para suportar ambientes não-Bun (CI, máquinas controladas por empresas).

### Por que usar React/Ink para UI de Terminal

As 81.546 linhas de código no diretório `components/` indicam que a complexidade da UI é extremamente alta. Se escrito manualmente com códigos ANSI brutos, o custo de manutenção seria incontrolável. A escolha do React/Ink traz:

1. **UI declarativa**: saída em streaming, estado de execução de ferramentas, caixas de diálogo de confirmação de permissão etc. podem ser acionados pelo estado React, em vez de controle imperativo do cursor
2. **Isolamento de componentes**: `screens/REPL.tsx` só precisa se preocupar com o layout geral; cada subfuncionalidade (campo de entrada, lista de mensagens, progresso de ferramentas) é encapsulada de forma independente
3. **Amigável ao hot reload**: durante o desenvolvimento, pode ser depurado com a mentalidade de React DevTools padrão
4. **Testabilidade**: componentes React podem ter testes unitários escritos com `@testing-library/react`, sem depender de um terminal real

### Filosofia de Otimização de Desempenho com Pré-busca Paralela

A otimização de inicialização do Claude Code tem um modelo de prioridade claro: **TTFR (Time To First Render) tem a mais alta prioridade, não "todas as inicializações concluídas"**.

Manifestações concretas:
- A leitura do Keychain (~65ms) é acionada no primeiro efeito colateral de import, não quando a chave de API é necessária
- A conexão de servidores MCP acontece em paralelo no background; o REPL renderiza sem esperar (o usuário vê a interface antes de o MCP terminar de conectar)
- Release notes, configurações do GrowthBook e plugin hooks são todos fire-and-forget

O custo é a necessidade de gerenciar cuidadosamente as condições de corrida de "ser consumido antes de a pré-busca terminar", controladas com precisão por pontos de await como `ensureKeychainPrefetchCompleted()`.

### Tradeoff entre Carregamento Lazy vs. Pré-carregamento

| Estratégia | Alvo | Razão |
|------|------|------|
| Pré-carregamento (efeito colateral de import) | Subprocesso MDM, Keychain | Intensivo em I/O; quanto mais cedo, melhor |
| Carregamento lazy (`await import()`) | OpenTelemetry (~400KB), gRPC (~700KB), componentes React TUI | Avaliação de módulo cara; não está no caminho crítico |
| Carregamento condicional (DCE com `feature()`) | COORDINATOR_MODE, KAIROS, CHICAGO_MCP | Funcionalidades experimentais; usuários externos não precisam |
| Atraso com `setImmediate()` | Hook de commit attribution | Evita bloquear o event loop na janela de microtarefas de `setup()` |

Essa estratégia em camadas faz com que o Claude Code realize apenas o "trabalho mínimo necessário para exibir a interface" durante a inicialização.

---

## 1.7 Padrões Transferíveis

### Padrão Geral de Otimização de Inicialização

A sequência de inicialização do Claude Code demonstra um framework de três camadas de otimização reutilizável: "**pré-aquecimento paralelo + carregamento lazy + DCE**":

**Padrão 1: Usar efeitos colaterais de módulo ES para pré-aquecimento de I/O**
```typescript
// Inserir I/O fire-and-forget entre declarações de import
import { startHeavyIORead } from './utils/heavyIO.js'
startHeavyIORead()  // Aciona imediatamente, sem await
import { SomethingElse } from './other.js'  // Carregamento paralelo
```
Aplicável para: qualquer dado de inicialização que "precisa ser lido, mas a leitura é lenta" (arquivos de configuração, credenciais, pré-conexões de rede).

**Padrão 2: Inicialização única com memoize**
```typescript
export const init = memoize(async (): Promise<void> => { ... })
```
Aplicável para: lógica de inicialização compartilhada por múltiplos pontos de entrada, prevenindo execução duplicada.

**Padrão 3: Modo `--bare` em camadas**
Chamadas de script/API não precisam de UI, verificações de terminal, analytics etc. Use `isBareMode()` para ignorar rapidamente, mantendo o baixo overhead de chamadas headless.

**Padrão 4: Separação de estado**
`bootstrap/state.ts` como módulo folha estritamente isolado (sem dependências circulares), acessado via setter/getter, com regras ESLint aplicadas. Isso permite que o módulo de estado seja importado com segurança em qualquer lugar.

### O Que o CLI do Doramagic Pode Aprender

Com base nessa análise, o Doramagic CLI pode adotar os seguintes padrões em seu design arquitetural:

1. **Separar o caminho crítico de inicialização**: separar rigorosamente o que "deve ser concluído antes da renderização" do que "pode ser concluído após a renderização", comentando os motivos (referência ao estilo de comentários `// ~65ms on every macOS startup` do Claude Code)

2. **Singleton de estado global + padrão de acessores**: referenciando `bootstrap/state.ts`, usar um módulo folha estrito para manter o estado da sessão, evitando que o estado fique espalhado por vários lugares

3. **Inicialização com `memoize`**: garantir que a inicialização execute apenas uma vez, independentemente do ponto de entrada chamado

4. **Separação de três modos**: interactive (React TUI) / headless (flag -p) / server (MCP), compartilhando a camada de ferramentas e serviços subjacentes

5. **Feature flag + DCE**: encapsular funcionalidades experimentais com feature flags, removidas automaticamente no lançamento

---

## 1.8 Índice do Código-Fonte

| Arquivo | Linhas | Conteúdo-chave |
|------|------|----------|
| `main.tsx` | 4.683 | Ponto de entrada principal, roteamento Commander, inicialização de estado, ramificações interativa/headless |
| `setup.ts` | 477 | Inicialização de sessão: cwd, hooks, worktree, pré-busca de plugins |
| `bootstrap/state.ts` | 1.758 | Singleton de estado global, definição do tipo `State`, todos os getters/setters |
| `entrypoints/init.ts` | ~400 | Inicialização global memoized: config, mTLS, proxy, carregamento lazy do OTel |
| `entrypoints/cli.tsx` | ~2.000 | Roteamento Commander.js, ramificações interactive/print/resume/teleport |
| `entrypoints/mcp.ts` | ~200 | Modo servidor MCP, exposição do conjunto de ferramentas |
| `entrypoints/sdk/coreSchemas.ts` | - | Schema de entrada/saída estruturada para modo SDK |
| `entrypoints/agentSdkTypes.ts` | - | Tipos exclusivos do SDK (HookEvent, ModelUsage etc.) |
| `replLauncher.tsx` | ~30 | Carregamento lazy de App + REPL, inicialização do React TUI |
| `QueryEngine.ts` | ~1.500 | Gerenciamento do ciclo de vida da sessão, núcleo do caminho headless |
| `Tool.ts` | - | Definição da interface de ferramentas (inputSchema, call, prompt etc.) |
| `tools/` | 50.828 | 30 implementações de ferramentas (BashTool/FileEditTool/AgentTool etc.) |
| `services/api/` | - | Chamadas à Claude API, retry, estatísticas de uso |
| `services/mcp/client.ts` | - | Gerenciamento de conexão do cliente MCP |
| `utils/startupProfiler.ts` | - | Pontos de instrumentação de desempenho `profileCheckpoint()` |
| `utils/secureStorage/keychainPrefetch.ts` | - | Pré-leitura paralela do macOS Keychain |
| `utils/settings/mdm/rawRead.ts` | - | Leitura paralela de configuração MDM |

### Localização de Código-Chave

- **Ponto de início do pré-aquecimento paralelo**: `main.tsx:12-20` (3 efeitos colaterais de import)
- **Inicialização memoized**: `entrypoints/init.ts:57` (`export const init = memoize(...)`)
- **Tipo de estado global**: `bootstrap/state.ts:30-200` (`type State = {...}`)
- **Definição do servidor MCP**: `entrypoints/mcp.ts:42` (`startMCPServer`)
- **Ponto de entrada de renderização REPL**: `replLauncher.tsx:14` (`launchRepl`)
- **Interface de ferramentas**: `Tool.ts:1-30` (`ToolInputJSONSchema`, `ToolUseContext`)
- **Ordem crítica de setup**: `setup.ts:77-230` (setCwd → captureHooksConfigSnapshot → worktree → tarefas em background)

---

*Contagem de caracteres do capítulo: ~9.800 | Data do snapshot do código-fonte: 2026-03-31*
