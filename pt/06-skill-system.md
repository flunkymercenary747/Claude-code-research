# Capítulo 6: Sistema de Skills

## 6.1 Visão Geral e Posicionamento

O sistema de Skills é uma das arquiteturas mais inovadoras do Claude Code. Ele codifica workflows reutilizáveis em arquivos Markdown e os aciona via comandos de barra (`/skill-name`) ou por invocação ativa da IA. Em essência, uma Skill é um "SOP para IA" — os passos, condições de julgamento e critérios de sucesso que especialistas humanos usam para executar tarefas complexas são registrados em formato Markdown estruturado, conferindo à IA capacidade de execução profissional reproduzível.

Ao contrário de simples Prompts, o sistema de Skills possui as seguintes características essenciais:

1. **Fusão declarativa + executiva**: O frontmatter declara metadados (permissões, modelo, condições de ativação), e o corpo contém instruções de execução
2. **Carregamento multi-fonte**: Bundled (embutido), usuário, projeto, plugin e fontes MCP, mesclados por prioridade
3. **Dois modos de execução**: inline (injeção no contexto da sessão atual) e fork (execução isolada em sub-agent independente)
4. **Ativação condicional**: Skills que se ativam automaticamente por caminho de arquivo via frontmatter `paths`
5. **Descoberta dinâmica**: Durante a sessão, conforme o usuário opera arquivos, Skills em diretórios mais profundos são descobertos e carregados automaticamente

O sistema de Skills não é um simples alias de comandos, mas um framework completo de orquestração de workflows.

---

## 6.2 Fundamentos Teóricos

### Padrão de Design para Workflows Reutilizáveis

O sistema de Skills resolve um problema central no uso de ferramentas de IA: **como o conhecimento especializado pode ser sedimentado e reproduzido?** A reutilização de código tradicional usa funções e classes, mas o "conhecimento" executado por IA são workflows descritos em linguagem natural, que não podem ser encapsulados diretamente em funções de código.

O design de Skills se inspira na ideia de SOP (Standard Operating Procedure) — registrar de forma estruturada o fluxo de execução, pontos de decisão e critérios de sucesso dos especialistas, fazendo com que a IA siga sempre o mesmo caminho de alta qualidade a cada execução.

### Definição de Workflow Declarativo vs. Imperativo

O sistema de Skills suporta ambos os estilos:

- **Declarativo**: Declara atributos como `allowed-tools`, `model`, `context` via frontmatter, deixando o sistema lidar automaticamente com controle de permissões e configuração do contexto de execução
- **Imperativo**: O corpo da Skill pode incorporar comandos shell (` `` `command` `` `) para execução direta, implementando "operações intercaladas nas instruções"

### A Filosofia Markdown-as-Code

Escolher Markdown em vez de JSON/YAML como formato de Skill foi uma decisão de design deliberada:

- **Legibilidade humana**: Desenvolvedores podem ler e editar Skills diretamente, entendendo sua intenção
- **Amigável para IA**: Os dados de treinamento da IA contêm grandes quantidades de Markdown, e a compreensão de Markdown pela IA é mais natural do que JSON
- **Estruturação progressiva**: Pode começar com prosa pura e adicionar gradualmente títulos, passos e regras, sem forçar uma estrutura completa
- **Amigável para controle de versão**: Diffs em Markdown são amigáveis para humanos, mudanças no workflow são visíveis à primeira vista na revisão de código

---

## 6.3 Formato e Estrutura de Dados da Skill

### Especificação de Formato do Arquivo Markdown de Skill

Arquivos Skill seguem uma estrutura de diretório fixa:

```
.claude/skills/<skill-name>/SKILL.md
```

O formato do arquivo é frontmatter + corpo Markdown:

```markdown
---
name: my-skill
description: Uma frase descrevendo o que este Skill faz
allowed-tools:
  - Bash(git:*)
  - Read
  - Write
when_to_use: |
  Usar quando o usuário quiser... Por exemplo: "cherry-pick to release", "hotfix".
argument-hint: "<branch-name>"
arguments:
  - branch_name
context: fork
model: opus
---

# My Skill

## Passos

### 1. Primeiro Passo
Operação específica...

**Critério de Sucesso**: Checkpoint que comprova a conclusão desta etapa
```

### Detalhes dos Campos do Frontmatter

A seguir estão todos os campos analisados pela função `parseSkillFrontmatterFields` (`loadSkillsDir.ts:184`):

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `name` | string | Nome de exibição (pode diferir do nome do diretório) |
| `description` | string | Descrição em uma frase, exibida no `/help` |
| `allowed-tools` | string[] | Lista de ferramentas em whitelist, suporta padrão de prefixo `Bash(git:*)` |
| `argument-hint` | string | Dica de parâmetro quando acionado pelo usuário, como `"<branch-name>"` |
| `arguments` | string[] | Lista de nomes de parâmetros, usada para substituição de variáveis `$arg_name` |
| `when_to_use` | string | Informa à IA quando chamar esta Skill ativamente, com frases de ativação |
| `version` | string | Número de versão da Skill |
| `model` | string | Override de modelo, como `opus`, `sonnet`, `inherit` para herdar |
| `disable-model-invocation` | boolean | Impede chamada ativa pela IA, somente acionamento manual pelo usuário |
| `user-invocable` | boolean | Se visível no `/help` (padrão `true`) |
| `context` | `"fork"` | Quando definido, executa em sub-agent isolado |
| `agent` | string | Especifica o tipo de agent |
| `effort` | EffortValue | Influencia a profundidade de raciocínio do modelo |
| `paths` | string[] | Padrões de caminho com sintaxe gitignore, usados para ativação condicional |
| `hooks` | HooksSettings | Configuração de hooks durante a execução da Skill |
| `shell` | FrontmatterShell | Configuração de execução de comandos shell inline |

### Tipo SkillDefinition

`bundledSkills.ts` define `BundledSkillDefinition` (linhas 12-41), enquanto Skills do sistema de arquivos correspondem ao tipo `Command` (`src/types/command.js`). Os dois convergem em um objeto `Command` unificado em `createSkillCommand` (`loadSkillsDir.ts:269`):

```typescript
// loadSkillsDir.ts:316-400
return {
  type: 'prompt',
  name: skillName,
  description,
  allowedTools,
  argumentHint,
  argNames: argumentNames.length > 0 ? argumentNames : undefined,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context: executionContext,
  agent,
  effort,
  paths,
  contentLength: markdownContent.length,
  isHidden: !userInvocable,
  progressMessage: 'running',
  loadedFrom,
  hooks,
  skillRoot: baseDir,
  async getPromptForCommand(args, toolUseContext) { ... }
} satisfies Command
```

---

## 6.4 Mecanismo de Carregamento de Skills

### Fluxo Completo de Carregamento do loadSkillsDir

`getSkillDirCommands` (`loadSkillsDir.ts:638`) é o ponto de entrada de todo o fluxo de carregamento, usando `lodash-es/memoize` para cache dos resultados, evitando I/O redundante:

```
Na inicialização
  ├── policySettings: ~/.claude-managed/.claude/skills/ (controle corporativo)
  ├── userSettings:   ~/.claude/skills/
  ├── projectSettings: .claude/skills/ (do cwd até home)
  ├── additionalDirs: diretórios extras especificados por --add-dir
  └── legacyCommands: .claude/commands/ (compatibilidade retroativa)

Durante a sessão (descoberta dinâmica)
  └── quando o usuário lê/escreve arquivos → discoverSkillDirsForPaths() → addSkillDirectories()
```

Os resultados de carregamento passam por deduplicação via `realpath` (`loadSkillsDir.ts:728-763`), evitando carregamento duplicado causado por symlinks.

### Prioridade de Carregamento Multi-Fonte

O comentário do código descreve explicitamente a prioridade de carregamento (`loadSkillsDir.ts:677-714`):

```
managed (política corporativa) < user (nível usuário) < project (nível projeto) < additional (--add-dir)
```

Este é o princípio de "quanto mais específico, maior a prioridade": projeto sobrepõe usuário, pois projetos têm necessidades específicas.

**Casos especiais:**
- Modo `--bare`: pula descoberta automática, carrega apenas diretórios explicitamente especificados por `--add-dir`
- `skillsLocked` (política plugin-only): proíbe carregamento de Skills de usuário/projeto, permite apenas fontes Plugin
- Variável de ambiente `CLAUDE_CODE_DISABLE_POLICY_SKILLS`: pula Skills de nível managed

### Lógica de Descoberta e Correspondência de Skills

**Descoberta estática** (na inicialização): `getSkillDirCommands` escaneia diretórios `~/.claude/skills/` de todos os níveis, suporta apenas formato de diretório (`skill-name/SKILL.md`), não suporta arquivos `.md` individuais.

**Descoberta dinâmica** (durante a sessão): Quando o usuário lê/escreve arquivos, `discoverSkillDirsForPaths` (`loadSkillsDir.ts:861`) percorre o caminho do arquivo para cima, verificando em cada diretório se existe `.claude/skills/`, carregando após descoberta via `addSkillDirectories`. Diretórios marcados como `.gitignore` são ignorados (evitando contaminação de Skills em `node_modules`).

**Ativação condicional** (frontmatter paths): Skills com campo `paths` ficam inicialmente invisíveis para o modelo, armazenadas no Map `conditionalSkills`. Quando o usuário opera arquivos com caminhos correspondentes, `activateConditionalSkillsForPaths` (`loadSkillsDir.ts:997`) usa a biblioteca `ignore` (sintaxe gitignore) para correspondência; quando há acerto, as Skills são movidas para `dynamicSkills` e ativadas.

---

## 6.5 Fluxo de Execução do SkillTool

### Caminho Completo de /skill-name até a Execução

`SkillTool` (`tools/SkillTool/SkillTool.ts:330`) é uma implementação padrão de `Tool`; a IA executa uma Skill chamando esta ferramenta. O caminho completo de execução é o seguinte:

```
Usuário digita /skill-name ou IA decide chamar SkillTool
  │
  ├── validateInput (SkillTool.ts:353)
  │     ├── Remove barra inicial (tratamento de compatibilidade)
  │     ├── Verifica prefixo _canonical_ (Skill remoto, experimental)
  │     ├── findCommand() busca o Command registrado
  │     ├── Verifica flag disableModelInvocation
  │     └── Confirma type === 'prompt'
  │
  ├── checkPermissions (SkillTool.ts:431)
  │     ├── Verifica regras de negação (deny)
  │     ├── Verifica Skill canonical remoto (permitido automaticamente)
  │     ├── Verifica regras de permissão (allow)
  │     ├── skillHasOnlySafeProperties() → permite Skills seguras automaticamente
  │     └── Padrão: exibe popup para o usuário (behavior: 'ask')
  │
  └── call (SkillTool.ts:580)
        ├── Verifica context === 'fork' → executeForkedSkill()
        │     └── prepareForkedCommandContext() + runAgent() (sub-agent isolado)
        └── Caso contrário (inline) → processPromptSlashCommand()
              └── Injeta newMessages + contextModifier na sessão atual
```

### Injeção de Contexto da Skill

Na execução inline, `call` retorna `newMessages` e `contextModifier` (`SkillTool.ts:767-840`):

- **newMessages**: Lista de mensagens após expansão da Skill, injetadas no contexto da conversa atual
- **contextModifier**: Função que modifica `ToolUseContext`, usada para:
  - Empilhar `allowedTools` (permissões de ferramentas declaradas pela Skill)
  - Sobrescrever `mainLoopModel` (se a Skill especificou um modelo)
  - Sobrescrever `effortValue` (se a Skill especificou um esforço)

Note que `contextModifier` usa padrão de chamada em cadeia (`SkillTool.ts:777`), tratando corretamente o empilhamento de múltiplos contextModifiers em vez de simplesmente sobrescrever.

### Substituição de Variáveis na Skill

`getPromptForCommand` em `createSkillCommand` (`loadSkillsDir.ts:343-398`) executa as seguintes substituições antes de retornar o conteúdo da Skill:

1. **Substituição de argumentos**: `$arg_name` → `substituteArguments()` injeta parâmetros fornecidos pelo usuário
2. **Variável de diretório**: `${CLAUDE_SKILL_DIR}` → caminho absoluto do diretório onde o arquivo Skill está localizado
3. **ID de Sessão**: `${CLAUDE_SESSION_ID}` → ID da sessão atual
4. **Execução de comandos shell**: `` !`command` `` → resultado da execução inline (apenas para Skills não-MCP)

Skills MCP têm execução de comandos shell desabilitada (`loadSkillsDir.ts:372`), impedindo que Skills não confiáveis remotas injetem comandos shell arbitrários.

### Interação entre Skill e Ferramentas

No modo de execução Forked (`executeForkedSkill`, `SkillTool.ts:121`), a Skill roda em um sub-agent completamente isolado:

- Inicia um agent independente via `runAgent()`, com orçamento de tokens próprio
- Mensagens de uso de ferramenta durante a execução são reportadas via callback `onProgress`, permitindo que a UI mostre progresso
- Os resultados de execução são extraídos via `extractResultText` para texto final, retornado ao agent pai
- A memória é liberada via `clearInvokedSkillsForAgent` (`SkillTool.ts:286`)

---

## 6.6 Listagem Completa e Análise das Bundled Skills

Skills embutidas são registradas via `registerBundledSkill()` (`bundledSkills.ts:55`), inicializadas na inicialização do CLI. A seguir, a análise de todas as 17 Skills embutidas:

### 1. `update-config` (`updateConfig.ts`, 475 linhas)

**Funcionalidade**: Configura o `settings.json` do Claude Code, incluindo todos os itens de configuração como Permissions, Hooks, Model e MCP.

**Características**: O corpo da Skill é gerado dinamicamente — usa `toJSONSchema(SettingsSchema())` para gerar automaticamente documentação JSON Schema a partir do schema Zod, garantindo que a documentação esteja sempre sincronizada com os tipos reais. Inclui documentação completa de Hooks (todos os eventos Hook, tipos de Hook, formato de saída JSON).

**Cenários de ativação**: Quando o usuário quer configurar automação de comportamento, regras de permissão, variáveis de ambiente ou configurações de modelo.

### 2. `schedule` (`scheduleRemoteAgents.ts`, 447 linhas)

**Funcionalidade**: Gerencia Agents remotos agendados (acionadores cron), cria, atualiza, lista e executa tarefas agendadas.

**Características**: Antes de chamar, verifica múltiplas pré-condições (OAuth tokens, informações do repositório, conectores MCP, ambientes cloud), injetando essas informações dinâmicas no prompt da Skill. Interage com o usuário via ferramenta `AskUserQuestion`.

**Cenários de ativação**: Quando o usuário quer criar um agent Claude Code agendado (como revisão diária de código, relatórios automáticos).

### 3. `keybindings-help` (`keybindings.ts`, 339 linhas)

**Funcionalidade**: Ajuda o usuário a personalizar atalhos de teclado, modificando `~/.claude/keybindings.json`.

**Características**: Gera documentação dinamicamente a partir de constantes de código via `generateContextsTable()` e `generateActionsTable()`, e lista atalhos não remapeáveis via `generateReservedShortcuts()`, prevenindo operações acidentais pelo usuário.

**Cenários de ativação**: Quando o usuário quer rebinding de atalhos, adicionar combinações de teclas ou modificar a tecla de envio.

### 4. `lorem-ipsum` (`loremIpsum.ts`, 282 linhas)

**Funcionalidade**: Gera texto de placeholder com quantidade fixa de palavras de token único, para contagem de tokens e testes de desempenho.

**Características**: Usa lista de palavras de token único validada pela API, garantindo que o parâmetro `lorem` controle com precisão o número de tokens. Frequentemente usado em benchmarks e análise de faturamento de tokens.

**Cenários de ativação**: Textos de teste com número exato de tokens.

### 5. `skillify` (`skillify.ts`, 197 linhas)

**Funcionalidade**: Converte automaticamente o processo de operação da sessão atual em um arquivo SKILL.md reutilizável.

**Características**: Este é o mecanismo de "auto-replicação" do sistema de Skills. Ao ler o histórico de memória de sessão e mensagens do usuário, guia o usuário por 4 rodadas de diálogo `AskUserQuestion` para confirmar nome do workflow, passos, parâmetros e condições de ativação, gerando finalmente um SKILL.md em formato padrão e gravando-o em disco.

**Restrição**: Disponível apenas para `USER_TYPE === 'ant'` (funcionários internos da Anthropic).

**Cenários de ativação**: Ao final da sessão, quando o usuário quer consolidar o fluxo de operação que acabou de completar em uma Skill reutilizável.

### 6. `claude-api` (`claudeApi.ts`, 196 linhas + `claudeApiContent.ts`, 220 linhas)

**Funcionalidade**: Ajuda desenvolvedores a usar a Claude API ou o Anthropic SDK para construir aplicações.

**Características**:
- Detecta automaticamente a linguagem do projeto atual (escaneando extensões de arquivo, suportando Python/TypeScript/Java/Go/Ruby/C#/PHP/curl)
- Carregamento tardio (conteúdo `.md` de 247KB é carregado apenas quando chamado), evitando impacto no tempo de inicialização
- Contém documentação de API específica por linguagem, padrões do Agent SDK, streaming etc.
- Usa o mecanismo `files` para gravar documentação em diretório temporário, permitindo que o modelo leia sob demanda via ferramentas Read/Grep

**Cenários de ativação**: Código com `import anthropic` ou usuário perguntando como usar a Claude API.

### 7. `batch` (`batch.ts`, 124 linhas)

**Funcionalidade**: Decomõe alterações de código em larga escala (como migrações, refatorações, renomeações em massa) em 5-30 agents de worktree paralelos para execução.

**Características**: Modelo de execução em três fases — Plan (entra em Plan Mode para pesquisa aprofundada e decomposição) → Spawn Workers (inicia agents de background paralelos com `isolation: "worktree"`) → Track Progress (renderiza tabela de status em tempo real). Cada worker opera em um git worktree independente, sem interferência mútua, abrindo PRs após conclusão.

**Cenários de ativação**: Migrações de código em larga escala, refatoração de todo o repositório, modificações em massa.

### 8. `loop` (`loop.ts`, 92 linhas)

**Funcionalidade**: Repete um prompt ou comando de barra em intervalos fixos.

**Características**: Analisa intervalos de tempo de forma inteligente (suporta formatos de prefixo como `5m`, `2h` e formato de sufixo como `every 20m`), convertendo-os em expressões cron e chamando `ScheduleCronTool` para registrar tarefas agendadas. Executa imediatamente uma vez após a configuração, sem esperar o primeiro acionamento agendado.

**Cenários de ativação**: Quando o usuário quer verificar periodicamente o status de implantação ou executar uma Skill ciclicamente.

### 9. `remember` (`remember.ts`, 82 linhas)

**Funcionalidade**: Revisa entradas de auto-memory e propõe promovê-las para `CLAUDE.md`, `CLAUDE.local.md` ou memória de equipe.

**Características**: Adota o princípio de "propor primeiro, confirmar depois" — não modifica arquivos diretamente, mas exibe um relatório categorizado (aguardando promoção / aguardando limpeza / em dúvida / sem ação), executando apenas após aprovação do usuário. Distingue entre convenções de nível de projeto (CLAUDE.md), preferências pessoais (CLAUDE.local.md) e conhecimento organizacional (memória de equipe).

**Restrição**: Disponível apenas para `USER_TYPE === 'ant'` e quando o recurso de auto-memory está habilitado.

**Cenários de ativação**: Quando o usuário quer organizar memórias, evitando acúmulo ilimitado de auto-memory.

### 10. `simplify` (`simplify.ts`, 69 linhas)

**Funcionalidade**: Realiza revisão de código em três dimensões no git diff atual (reutilização de código, qualidade de código, eficiência) e corrige diretamente os problemas encontrados.

**Características**: Inicia simultaneamente três sub-agents paralelos, cada um responsável por:
- **Agent de Reutilização de Código**: Identifica reinvenção de rodas, aponta funções utilitárias existentes
- **Agent de Qualidade de Código**: Identifica estado redundante, inflação de parâmetros, copiar e colar, abstrações vazadas etc.
- **Agent de Eficiência**: Identifica computações desnecessárias, falta de concorrência, padrões N+1, vazamentos de memória etc.

Após a conclusão dos três agents, os achados são mesclados e as correções são feitas diretamente, não apenas relatadas.

**Cenários de ativação**: Revisão de qualidade após completar um trecho de código; também chamado automaticamente pelo fluxo do worker da Skill `batch`.

### 11. `debug` (`debug.ts`)

**Funcionalidade**: Diagnostica logs de depuração da sessão atual do Claude Code, ajudando a resolver problemas.

**Características**: Lê as últimas linhas dos logs de depuração via tail (no máximo 64KB), evitando picos de memória causados por arquivos de log muito grandes em sessões longas. Para usuários não Anthropic, habilita o logging de debug antes de ler. Marcado como `disableModelInvocation: true`, impedindo chamada automática pela IA (somente acionamento manual pelo usuário).

### 12. `stuck` (`stuck.ts`)

**Funcionalidade**: Diagnostica outros processos do Claude Code congelados ou travados na máquina e envia o relatório para um canal do Slack.

**Características**: Ferramenta de diagnóstico interno da Anthropic. Detecta anomalias como alta CPU (≥90% sustentado), estado D (I/O suspenso), estado T (parado por Ctrl+Z), estado Z (processo zumbi), alta memória (≥4GB). Usa estrutura de duas mensagens para enviar relatório do Slack (resumo de nível superior + detalhes na thread).

### 13. `verify` (`verify.ts`)

**Funcionalidade**: Verifica se as alterações de código atendem às expectativas executando a aplicação.

**Características**: Lê o corpo da Skill (análise de SKILL.md) de `verifyContent.ts`, gravando arquivos auxiliares em diretório temporário via mecanismo `files`. Disponível apenas para `USER_TYPE === 'ant'`.

### 14. `claudeInChrome` (`claudeInChrome.ts`)

**Funcionalidade**: Inicia uma sessão headless conectada a um Chrome real, com extensão Side Panel; Claude pode controlar o navegador em tempo real.

### 15. `claudeCodeGuide` (embutida no sistema `AgentTool`)

Usada para o fluxo de orientação interno do Claude Code.

---

## 6.7 Relação entre Skills e Commands

### Fronteiras entre os Dois

No design do Claude Code, Skill e Command eram conceitos diferentes, mas agora estão unificados:

- **Historicamente**: O diretório `/commands/` armazenava comandos de prompt simples (arquivos `.md`), e o diretório `/skills/` armazenava workflows mais complexos com estrutura de diretório (`skill-name/SKILL.md`)
- **Atualmente**: Ambos são carregados por `loadSkillsDir.ts`, convertidos uniformemente para o tipo `Command`, e `/commands/` é marcado como `loadedFrom: 'commands_DEPRECATED'` (`loadSkillsDir.ts:608`)

A diferença real atualmente está apenas no caminho de carregamento:
- `/skills/skill-name/SKILL.md`: Novo formato, recomendado, suporta `baseDir` (Skills podem carregar arquivos auxiliares)
- `/commands/skill-name.md` ou `/commands/skill-name/SKILL.md`: Formato antigo, compatível retroativamente

### Quando Usar Skill vs. Command

| Cenário | Abordagem Recomendada |
|---------|----------------------|
| Workflows com múltiplos arquivos (Skill com recursos auxiliares) | Formato de diretório `/skills/` |
| Reutilização simples de prompt (um único arquivo md basta) | Ainda pode usar `/commands/` (compatível) |
| Necessidade de variável `${CLAUDE_SKILL_DIR}` | Obrigatório usar formato de diretório `/skills/` |
| Necessidade de recursos embutidos `files:` (bundled skill) | `BundledSkillDefinition.files` |
| Embutido no binário do CLI | `registerBundledSkill()` |

---

## 6.8 Análise de Decisões de Design

### Por Que Escolher Markdown em Vez de JSON/YAML

As instruções de execução da Skill (corpo) são escritas em linguagem natural para que a IA as entenda e siga. JSON/YAML só pode codificar dados estruturados, não pode expressar diretamente instruções complexas como "primeiro procure arquivos relevantes, depois analise dependências, cuidando para não modificar arquivos de teste".

Markdown combina ambos: frontmatter (YAML) para metadados estruturados, e o corpo (Markdown) para instruções de execução legíveis por humanos. Esta é uma escolha pragmática de formato.

### Controle de Permissão da Skill

O controle de permissão usa um mecanismo de "whitelist + perguntar" (`SkillTool.ts:871-900`):

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'progressMessage', 'contentLength', 'argNames',
  'model', 'effort', 'source', 'pluginInfo', 'disableNonInteractive',
  'skillRoot', 'context', 'agent', 'getPromptForCommand', 'frontmatterKeys',
  // CommandBase properties...
  'name', 'description', 'hasUserSpecifiedDescription', ...
  // NOT included: 'allowedTools', 'hooks', 'paths', etc.
])
```

`skillHasOnlySafeProperties()` verifica se uma Skill usa apenas "propriedades seguras" — se a Skill não declarou propriedades sensíveis como `allowedTools`, `hooks`, `paths`, a execução é permitida automaticamente sem confirmação do usuário. Este é um bom design de segurança: novas propriedades adicionadas são inseguras por padrão e precisam de revisão explícita antes de serem adicionadas à whitelist.

### Mecanismo de Escrita Segura de Arquivos

Skills embutidas incorporam arquivos auxiliares via campo `files`, usando medidas de segurança rígidas ao gravar em disco (`bundledSkills.ts:171-194`):

```typescript
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'
    : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW
```

Usa `O_NOFOLLOW | O_EXCL` para prevenir ataques de symlink, permissões de arquivo são `0o600` (somente leitura/escrita pelo proprietário). O diretório de gravação contém um nonce aleatório gerado a cada inicialização do processo, prevenindo ataques de predição de caminho.

### Estratégia de Integração de MCP Skills

MCP Skills implementam uma elegante inversão de dependência via `mcpSkillBuilders.ts` (`mcpSkillBuilders.ts:1-43`):

A lógica de descoberta MCP (`mcpSkills.ts`) precisa usar `createSkillCommand` e `parseSkillFrontmatterFields`, mas o import direto causaria dependências circulares. A solução é:

1. `loadSkillsDir.ts` chama `registerMCPSkillBuilders()` na inicialização do módulo para registrar essas duas funções
2. `mcpSkills.ts` as obtém via `getMCPSkillBuilders()` quando necessário

Este design também resolve limitações técnicas do bundle Bun: no bundle Bun, imports dinâmicos com variáveis (não literais) não podem ser resolvidos, então não é possível usar `await import(variable)` — apenas este padrão de registro funciona.

---

## 6.9 Padrões Transferíveis

### Comparação com o Sistema de Skills do Doramagic

| Dimensão | Claude Code Skill | Doramagic Skill |
|----------|------------------|-----------------|
| Formato de arquivo | `SKILL.md` (Markdown + YAML frontmatter) | `SKILL.md` (mesmo formato) |
| Estrutura de diretório | `~/.claude/skills/name/SKILL.md` | `~/.openclaw/skills/name/SKILL.md` |
| Engine de execução | SkillTool (chamada de ferramenta IA) | Chamada de ferramenta OpenClaw |
| Prioridade de fonte | policy < user < project < plugin | Regras da plataforma OpenClaw |
| Skills embutidas | 15+, compiladas no binário | Em construção |
| Substituição de argumentos | `$arg_name`, frontmatter `arguments` | Mesmo mecanismo |
| Contexto de execução | inline / fork (sub-agent) | inline (fase atual) |
| Ativação condicional | frontmatter `paths` | Ainda não implementado |
| Descoberta dinâmica | Descoberta automática disparada por operações de arquivo | Ainda não implementado |

### Padrões Essenciais Transferíveis

**1. Padrão `skillify`: Auto-replicação de Workflows**

A Skill `skillify` do Claude Code é um design extremamente elegante — faz a IA analisar suas próprias operações recentes e guia o usuário por diálogo para consolidá-las em uma Skill reutilizável. O Doramagic pode igualmente implementar um `/dora-skillify`, consolidando um processo bem-sucedido de extração de conhecimento em uma Skill específica do projeto.

**2. Mecanismo de Chamada Ativa pela IA via `when_to_use`**

O campo frontmatter `when_to_use` informa à IA quando chamar ativamente uma Skill, sem que o usuário precise digitar explicitamente o comando de barra. As Skills do Doramagic também devem valorizar este campo, permitindo que a extração de conhecimento seja acionada automaticamente no momento certo.

**3. Descoberta Dinâmica de Skills e Ativação Condicional**

O mecanismo de ativar Skills por caminho de arquivo é muito adequado para cenários de conhecimento específicos de projeto do Doramagic: quando o usuário opera arquivos de um domínio, ativa automaticamente a Skill de extração do domínio correspondente (como ativar Skills de análise de arquitetura frontend ao operar arquivos TypeScript).

**4. Gerenciamento de Recursos Auxiliares via Mecanismo `files`**

Skills embutidas embalam documentação de referência e código de exemplo no pacote Skill via campo `files`, com o modelo lendo sob demanda em vez de injetar tudo no contexto de uma vez. Skills grandes do Doramagic (como Soul Extractor) podem adotar este padrão para gerenciar templates de extração e materiais de referência.

**5. Modelo de Segurança: Whitelist allowedTools + Permissão Automática para Skills Seguras**

Uma Skill só pode usar ferramentas declaradas no frontmatter. O Claude Code diferencia ainda mais entre "Skills seguras" (sem permissões especiais) e "Skills que requerem confirmação" (com allowedTools/hooks), permitindo automaticamente as primeiras para reduzir fricção. Este modelo de permissão vale a pena ser adotado pela plataforma OpenClaw.

---

## 6.10 Índice de Código-Fonte

| Arquivo | Linhas | Função |
|---------|--------|--------|
| `skills/loadSkillsDir.ts` | 1.087 | Core de carregamento de Skills: descoberta, análise, deduplicação, ativação condicional, descoberta dinâmica |
| `skills/bundledSkills.ts` | 220 | Registro de Skills embutidas, extração de arquivos, escrita segura |
| `tools/SkillTool/SkillTool.ts` | 1.108 | Ferramenta de execução de Skills: validação, permissões, execução inline/fork |
| `skills/mcpSkillBuilders.ts` | 44 | Registro de construtores MCP Skill (resolve dependências circulares) |
| `skills/bundled/updateConfig.ts` | 475 | update-config: assistente de configuração settings.json |
| `skills/bundled/scheduleRemoteAgents.ts` | 447 | schedule: gerenciamento de agents remotos agendados |
| `skills/bundled/keybindings.ts` | 339 | keybindings-help: configuração de atalhos de teclado |
| `skills/bundled/loremIpsum.ts` | 282 | lorem-ipsum: texto de placeholder com contagem exata de tokens |
| `skills/bundled/skillify.ts` | 197 | skillify: consolidação automática de workflows de sessão em Skill |
| `skills/bundled/claudeApi.ts` | 196 | claude-api: assistente de desenvolvimento Claude API (multilíngue) |
| `skills/bundled/claudeApiContent.ts` | 220 | Conteúdo de documentação de 247KB do claude-api (inlined na build) |
| `skills/bundled/batch.ts` | 124 | batch: alterações em worktree paralelo em larga escala |
| `skills/bundled/loop.ts` | 92 | loop: execução repetida de prompt em intervalos |
| `skills/bundled/remember.ts` | 82 | remember: revisão e promoção de memórias |
| `skills/bundled/simplify.ts` | 69 | simplify: revisão de código em três dimensões com correção |
| `skills/bundled/debug.ts` | ~60 | debug: diagnóstico de logs de depuração de sessão |
| `skills/bundled/stuck.ts` | ~60 | stuck: diagnóstico de processos congelados + relatório Slack |
| `skills/bundled/verify.ts` | ~30 | verify: verificação de alterações de código executando a aplicação |
| `skills/bundled/claudeInChrome.ts` | ~40 | claude-in-chrome: controle do navegador Chrome |
| `skills/bundled/index.ts` | — | Ponto de entrada de registro de todas as Skills embutidas |

**Índice de Funções-Chave:**

| Função | Arquivo:Linha | Descrição |
|--------|--------------|-----------|
| `getSkillDirCommands` | `loadSkillsDir.ts:638` | Ponto de entrada principal de carregamento (memoized) |
| `parseSkillFrontmatterFields` | `loadSkillsDir.ts:184` | Análise de campos do frontmatter |
| `createSkillCommand` | `loadSkillsDir.ts:269` | Construção do objeto Command |
| `loadSkillsFromSkillsDir` | `loadSkillsDir.ts:407` | Carregamento do diretório `/skills/` |
| `discoverSkillDirsForPaths` | `loadSkillsDir.ts:861` | Descoberta dinâmica de diretórios de Skills |
| `activateConditionalSkillsForPaths` | `loadSkillsDir.ts:997` | Ativação de Skills condicionais |
| `registerBundledSkill` | `bundledSkills.ts:55` | Registro de Skills embutidas |
| `executeForkedSkill` | `SkillTool.ts:121` | Execução no modo Fork |
| `skillHasOnlySafeProperties` | `SkillTool.ts:871+` | Verificação de Skill segura |
| `registerMCPSkillBuilders` | `mcpSkillBuilders.ts:31` | Registro de construtores MCP |
