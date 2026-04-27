# Capítulo 9: Sistema de Memória

## 9.1 Visão Geral e Posicionamento

O sistema de memória do Claude Code é um dos subsistemas mais sofisticados e com maior investimento de engenharia em toda a toolchain. Ele resolve a limitação mais fundamental dos LLMs: a janela de contexto é zerada ao término de uma sessão. A cada nova sessão, o Claude começa do zero — sem saber quem é o usuário, quais são suas preferências, quais erros foram cometidos anteriormente ou quais são as normas da equipe.

O objetivo do sistema de memória é: **permitir que o Claude mantenha continuidade entre sessões, agindo como um verdadeiro colaborador de longo prazo.**

Do ponto de vista do volume de código-fonte, trata-se de um sistema considerável:
- Diretório `memdir/`: 7 arquivos, 1736 linhas
- `services/SessionMemory/`: 3 arquivos, 1026 linhas
- `services/extractMemories/`: 2 arquivos, 769 linhas
- `services/teamMemorySync/`: 5 arquivos, 2167 linhas

Total de aproximadamente 5700 linhas, representando cerca de 1,1% do codebase, mas com uma densidade de complexidade e raciocínio de design muito superior a essa proporção.

---

## 9.2 Bases Teóricas

### Correspondência com o Modelo de Memória Humana

A arquitetura do sistema corresponde explicitamente a três tipos de memória da ciência cognitiva:

| Memória Humana | Correspondente no Claude Code | Implementação Técnica |
|----------------|-------------------------------|----------------------|
| Memória de Trabalho (Working Memory) | Janela de contexto atual | Lista de mensagens da sessão, limpa ao término |
| Memória Episódica (Episodic Memory) | Session Memory | `~/.claude/projects/<slug>/session-memory.md`, atualizada continuamente durante a sessão |
| Memória Semântica (Semantic Memory) | Persistent Memory | `~/.claude/projects/<slug>/memory/*.md`, armazenada a longo prazo entre sessões |

Session Memory corresponde a "lembranças do momento" — registra o que está sendo feito na sessão atual e em que ponto está; Persistent Memory corresponde a "conhecimento acumulado" — preferências do usuário, lições aprendidas, contexto do projeto.

### A Escolha entre Grafo de Conhecimento e Memória Documental

O sistema optou por **documentos Markdown no sistema de arquivos** em vez de banco de dados ou índice vetorial. Essa escolha é explicitamente fundamentada em comentários em `memoryTypes.ts`:

> Memories are constrained to four types capturing context NOT derivable from the current project state. Code patterns, architecture, git history, and file structure are derivable (via grep/git/CLAUDE.md) and should NOT be saved as memories.
>
> (`memdir/memoryTypes.ts:7-12`)

Isso revela um princípio fundamental: **informações que podem ser consultadas em tempo real não precisam ser memorizadas.** A memória só deve armazenar contexto "não derivável" — preferências do usuário, lições históricas da equipe, motivações por trás das decisões do projeto. Essa abordagem difere fundamentalmente do grafo de conhecimento, que tende a incluir tudo que pode ser estruturado.

### Aplicação de Consistência Eventual na Memória

O design de sincronização do Team Memory adota explicitamente semântica de consistência eventual:

> - Pull overwrites local files with server content (server wins per-key).
> - Push uploads only keys whose content hash differs from serverChecksums (delta upload). Server uses upsert: keys not in the PUT are preserved.
> - **File deletions do NOT propagate**: deleting a local file won't remove it from the server, and the next pull will restore it locally.
>
> (`services/teamMemorySync/index.ts:19-22`)

O design de não propagar exclusões é intencional — a memória da equipe é um ativo "apenas acrescido"; exclusões acidentais não devem se tornar perdas permanentes. Isso é uma implementação conservadora do princípio de consistência eventual em sistemas distribuídos.

---

## 9.3 Arquitetura de Três Camadas de Memória

O sistema é composto por três camadas, ordenadas do ciclo de vida mais curto ao mais longo:

### Primeira Camada: Session Memory (Nível de Sessão)

**Caminho do arquivo**: `~/.claude/projects/<sanitized-cwd>/session-memory.md` (obtido via `getSessionMemoryPath()`)

Session Memory é um arquivo Markdown **mantido continuamente durante a sessão atual**, com estrutura de conteúdo fixa:

```markdown
# Session Title
# Current State
# Task specification
# Files and Functions
# Workflow
# Errors & Corrections
# Codebase and System Documentation
# Learnings
# Key results
# Worklog
```

(`services/SessionMemory/prompts.ts:14-36`, `DEFAULT_SESSION_MEMORY_TEMPLATE`)

Não é limpa ao término da sessão; em vez disso, é lida pelo mecanismo de Auto Compact durante a compressão do contexto e injetada na nova janela de contexto como "episódio anterior".

**Restrições de estrutura de dados**:
- Limite de 2000 tokens por seção (`MAX_SECTION_LENGTH = 2000`)
- Limite total de 12000 tokens (`MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000`)
- Quando o limite é excedido, o sistema adiciona um aviso no prompt e solicita que o Agent comprima

**Ciclo de vida**: Vinculado à sessão do projeto atual, lida quando o Auto Compact é acionado

### Segunda Camada: Persistent Memory (Memória Persistente entre Sessões)

**Caminho do arquivo**: `~/.claude/projects/<sanitized-git-root>/memory/`

Esta é a camada central de memória de longo prazo. Cada memória é armazenada como um arquivo `.md` individual, com YAML frontmatter:

```markdown
---
name: {{memory name}}
description: {{one-line description — used to decide relevance in future conversations, so be specific}}
type: {{user, feedback, project, reference}}
---

{{memory content — for feedback/project types, structure as: rule/fact, then **Why:** and **How to apply:** lines}}
```

(`memdir/memoryTypes.ts:230-240`, `MEMORY_FRONTMATTER_EXAMPLE`)

A lógica de resolução de caminho é gerenciada por `getAutoMemPath()` (`memdir/paths.ts:173-190`), com a seguinte ordem de prioridade:

1. Variável de ambiente `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` (usada no cenário Cowork multiusuário)
2. `autoMemoryDirectory` em `settings.json` (confiável apenas para fontes policy/local/user; **não confiável** para projectSettings, para evitar que repositórios maliciosos sequestrem o caminho de escrita)
3. `<memoryBase>/projects/<sanitized-git-root>/memory/` (padrão)

As git worktrees são unificadas ao git root canônico (`findCanonicalGitRoot`), garantindo que diferentes worktrees do mesmo repositório compartilhem a mesma memória.

**Ciclo de vida**: Permanente, até que o usuário exclua explicitamente ou o Agent atualize/exclua

### Terceira Camada: Team Memory (Memória Compartilhada da Equipe)

**Caminho do arquivo**: `~/.claude/projects/<sanitized-git-root>/memory/team/` (valor retornado por `getTeamMemPath()`)

Team Memory é um subdiretório da Persistent Memory, sincronizado via API REST entre todos os membros autenticados do mesmo repositório GitHub. É uma extensão sobre Auto Memory; `isTeamMemoryEnabled()` verifica primeiro `isAutoMemoryEnabled()` para garantir que o sistema pai esteja habilitado.

**Ciclo de vida**: Mantido pelo servidor da Anthropic, persistido entre usuários e máquinas

---

## 9.4 Mecanismo de Índice MEMORY.md

MEMORY.md é o **arquivo de índice** da camada Persistent Memory, não o arquivo de conteúdo. O sistema diferencia explicitamente esses dois em vários lugares:

> `MEMORY.md` is an index, not a memory — each entry should be one line, under ~150 characters: `- [Title](file.md) — one-line hook`. It has no frontmatter. Never write memory content directly into `MEMORY.md`.
>
> (`services/extractMemories/prompts.ts:buildExtractAutoOnlyPrompt`)

### Especificação de Formato

Cada linha do MEMORY.md é um link apontando para um arquivo de memória específico:

```
- [Preferência do usuário: respostas concisas](feedback_terse_responses.md) — usuário não gosta de resumos no final das respostas
- [Contexto do projeto: refatoração do middleware Auth](project_auth_rewrite.md) — exigência de conformidade jurídica, não dívida técnica
```

O MEMORY.md é carregado no prompt do sistema a cada início de sessão, portanto seu tamanho impacta diretamente o consumo de tokens de cada requisição.

### Limite Duplo: 200 Linhas / 25KB

O sistema define limites duplos rígidos em `memdir/memdir.ts`:

```typescript
export const MAX_ENTRYPOINT_LINES = 200
// ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that
// slip past the line cap (p100 observed: 197KB under 200 lines).
export const MAX_ENTRYPOINT_BYTES = 25_000
```

(`memdir/memdir.ts:30-33`)

A lógica de truncamento é implementada em `truncateEntrypointContent()` (`memdir/memdir.ts:55-102`): primeiro trunca por número de linhas, depois por bytes (cortando na quebra de linha mais próxima para evitar truncamento no meio de uma linha). Após o truncamento, uma mensagem de aviso é adicionada indicando que o índice é muito longo.

**Intenção de design**: aproximadamente 125 caracteres × 200 linhas ≈ 25KB, um limite razoável validado por dados reais (percentil p97). O limite de bytes trata casos extremos de "menos de 200 linhas, mas com linhas extremamente longas" (observado no p100: 197KB sem exceder o limite de linhas).

### Relação com Arquivos de Memória

Escrever uma memória é uma **operação em dois passos**:
1. Escrever o arquivo de conteúdo (`user_role.md`, `feedback_testing.md`, etc.)
2. Adicionar uma entrada apontando para o MEMORY.md

Na leitura, apenas os arquivos selecionados por `findRelevantMemories` são lidos (veja a seção 9.7); o próprio MEMORY.md fica permanentemente no prompt do sistema.

---

## 9.5 Quatro Tipos de Memória

O sistema restringe toda a memória a quatro tipos — uma das decisões de design mais importantes. As definições de tipo estão em `memdir/memoryTypes.ts` (constante `MEMORY_TYPES`):

### Tipo user

**Cenários de uso**: papel do usuário, objetivos, responsabilidades, background de conhecimento

**Quando acionar**: sempre que o papel, preferências, responsabilidades ou nível de conhecimento do usuário forem identificados

**Finalidade**: ajustar o modo de resposta para o nível cognitivo e necessidades do usuário específico

**Escopo**: sempre private (privado), mesmo no modo Team Memory

**Contra-exemplos (conteúdo que não deve ser salvo)**:
> Avoid writing memories about the user that could be viewed as a negative judgement or that are not relevant to the work you're trying to accomplish together.
>
> (`memdir/memoryTypes.ts:65`)

### Tipo feedback

**Cenários de uso**: correções e confirmações do usuário sobre o modo de trabalho — tanto "não faça isso" quanto "continue fazendo assim"

**Requisitos de estrutura**:
- A regra em si
- Linha `**Why:**` (fornecendo o motivo para julgar se se aplica em casos extremos)
- Linha `**How to apply:**` (quando e onde tem efeito)

**Design único**: requer explicitamente o registro tanto de **lições de falhas quanto de confirmações de sucesso**:

> Record from failure AND success: if you only save corrections, you will avoid past mistakes but drift away from approaches the user has already validated, and may grow overly cautious.
>
> (`memdir/memoryTypes.ts:113`)

**Quando acionar**: quando o usuário diz "não faça isso" (correção explícita) ou "exatamente isso"/"perfeito" (confirmação implícita, mais difícil de identificar)

**Escopo**: private por padrão; apenas salvo como team quando a orientação é claramente uma norma a nível de projeto (como estratégia de testes, restrições de build)

### Tipo project

**Cenários de uso**: informações sobre trabalho em andamento, objetivos, planos, bugs ou eventos que **não podem ser derivados do código ou do histórico git**

**Requisitos de estrutura**:
- O fato/decisão em si
- Linha `**Why:**` (motivação — geralmente restrições, prazos ou necessidades de stakeholders)
- Linha `**How to apply:**` (como influencia recomendações)

**Regra importante**: ao salvar, datas relativas devem ser convertidas para absolutas ("próxima quinta" → "2026-04-08"), garantindo que a memória seja interpretável com o passar do tempo.

**Escopo**: team por padrão (contexto do projeto é inerentemente compartilhado)

**Característica de decaimento**: memórias do tipo project decaem mais rapidamente; o campo Why ajuda a julgar se a memória ainda é válida.

### Tipo reference

**Cenários de uso**: ponteiros para localizações de informações em sistemas externos (projetos Linear, canais Slack, dashboards Grafana, etc.)

**Quando acionar**: ao descobrir a localização de um recurso externo e sua finalidade

**Escopo**: geralmente team

**Exemplo típico**:

```
user: check the Linear project "INGEST" for pipeline bug context
assistant: [saves reference memory: pipeline bugs tracked in Linear "INGEST"]
```

### Conteúdo que Não Deve Ser Salvo (excluído explicitamente)

`WHAT_NOT_TO_SAVE_SECTION` lista explicitamente seis categorias de conteúdo que não devem ser salvos (`memdir/memoryTypes.ts:196-207`):

1. Padrões de código, convenções, arquitetura, caminhos de arquivo — deriváveis do estado atual do projeto
2. Histórico git, mudanças recentes — `git log`/`git blame` são fontes autorizadas
3. Soluções de debug ou métodos de correção — a correção está no código, o contexto está na mensagem de commit
4. Conteúdo já documentado no CLAUDE.md
5. Detalhes de tarefas temporárias: trabalho em andamento, estado temporário, contexto da sessão atual
6. **Mesmo conteúdo que o usuário solicite explicitamente salvar** — se o usuário pedir para salvar uma lista de PRs, deve-se perguntar "há algo surpreendente ou não óbvio nisso? Isso sim valeria a pena salvar"

---

## 9.6 Extração Automática de Memória

### Mecanismo de Extração Automática com Fork Agent

A extração de memória usa o padrão "Fork Agent" — cria um contexto de Agent idêntico à sessão principal e executa em segundo plano de forma assíncrona, sem bloquear o fluxo principal do diálogo.

O núcleo desse mecanismo é `runForkedAgent()`; o Agent de extração compartilha o prompt cache da sessão pai, maximizando a taxa de acerto de cache:

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool,
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,   // não grava no registro da sessão principal, evita condição de corrida
  maxTurns: 5,            // limite rígido para evitar loops de verificação
})
```

(`services/extractMemories/extractMemories.ts:258-267`)

O comentário sobre `maxTurns: 5` explica a intenção:

> Well-behaved extractions complete in 2-4 turns (read → write). A hard cap prevents verification rabbit-holes from burning turns.

A estratégia eficiente do Agent de extração é explicitamente projetada para "completar em 2 turnos":
- **1º turno**: emite em paralelo todas as requisições FileRead para os arquivos que precisam ser atualizados
- **2º turno**: emite em paralelo todas as requisições FileWrite/FileEdit

### Momento de Acionamento (Stop Hooks)

A extração é acionada **após o término de cada ciclo completo de consulta** — ou seja, quando o modelo produz uma resposta final sem tool_use, via `handleStopHooks` chamando `executeExtractMemories()`.

O estado é gerenciado via closures; as variáveis-chave incluem:

```typescript
let lastMemoryMessageUuid: string | undefined    // cursor: até onde extraiu da última vez
let inProgress = false                           // evita execução concorrente
let pendingContext: {...} | undefined            // chamadas que chegam durante execução são armazenadas aqui
let turnsSinceLastExtraction = 0                // para controle de throttle
```

(`services/extractMemories/extractMemories.ts:225-240`)

**Estratégia de controle de concorrência**: se uma nova chamada chega enquanto a extração está em andamento, ela é "stashed" (armazenada em `pendingContext`) em vez de descartada. Quando a extração atual termina, imediatamente executa uma "trailing extraction" com o contexto mais recente, garantindo que as últimas mensagens não sejam perdidas.

**Regra de exclusão mútua**: se o Agent principal já escreveu arquivos de memória (`hasMemoryWritesSince` detecta isso), o Fork Agent pula a extração desta vez e apenas avança o cursor:

```typescript
if (hasMemoryWritesSince(messages, lastMemoryMessageUuid)) {
  // o Agent principal escreveu, pula o fork agent, avança o cursor
  lastMemoryMessageUuid = lastMessage.uuid
  return
}
```

(`services/extractMemories/extractMemories.ts:198-209`)

### Análise do Prompt de Extração

A filosofia central de design do prompt de extração é **eficiência de informação**:

```typescript
function opener(newMessageCount: number, existingMemories: string): string {
  return [
    `You are now acting as the memory extraction subagent. Analyze the most recent ~${newMessageCount} messages above...`,
    `You have a limited turn budget. ${FILE_EDIT_TOOL_NAME} requires a prior ${FILE_READ_TOOL_NAME}...`,
    `You MUST only use content from the last ~${newMessageCount} messages... Do not waste any turns attempting to investigate or verify that content further — no grepping source files, no reading code...`,
    manifest,  // pré-injeta a lista de memórias existentes, evitando que o Agent gaste um turno em ls
  ].join('\n')
}
```

(`services/extractMemories/prompts.ts:20-47`)

A pré-injeção do manifesto de memórias existentes (`existingMemories`) é uma otimização chave — evita que o Agent desperdice um turno listando o diretório, fornecendo diretamente no prompt uma lista estruturada de arquivos (nome do arquivo, tipo, timestamp, description).

### Mecanismo de Acionamento do Session Memory

O Session Memory usa um mecanismo de acionamento diferente — via `postSamplingHooks` em vez de Stop Hooks, avaliando se é necessário atualizar após cada amostragem do modelo:

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
  ...
}
```

(`services/SessionMemory/sessionMemory.ts:130-150`)

Limites de acionamento padrão (`DEFAULT_SESSION_MEMORY_CONFIG`, `services/SessionMemory/sessionMemoryUtils.ts:29-33`):

| Parâmetro | Valor Padrão | Descrição |
|-----------|--------------|-----------|
| `minimumMessageTokensToInit` | 10.000 | Tokens mínimos necessários para inicializar a memória de sessão |
| `minimumTokensBetweenUpdate` | 5.000 | Crescimento mínimo de tokens entre duas atualizações |
| `toolCallsBetweenUpdates` | 3 | Número mínimo de chamadas de ferramenta entre duas atualizações |

Esses valores podem ser ajustados dinamicamente via configuração remota do GrowthBook (`tengu_sm_config`).

---

## 9.7 Recuperação Inteligente de Memória

### Sonnet Seleciona até 5 Memórias Relevantes

A recuperação de memória não é leitura em massa, mas sim **primeiro escaneia frontmatter, depois usa Sonnet para selecionar as até 5 memórias mais relevantes**.

O fluxo central está em `findRelevantMemories()` (`memdir/findRelevantMemories.ts:32-66`):

1. `scanMemoryFiles()` escaneia o diretório de memórias, lê as primeiras 30 linhas (frontmatter) de cada arquivo, retornando `MemoryHeader[]`
2. Filtra memórias já exibidas em turnos anteriores (`alreadySurfaced`), reservando 5 slots para novo conteúdo
3. Usa Sonnet para chamar `selectRelevantMemories()`, selecionando os nomes dos arquivos mais relevantes com base na query e nas descriptions dos arquivos
4. Retorna os caminhos e mtimes das memórias selecionadas

### Lógica de Julgamento de Relevância

O system prompt do Sonnet é cuidadosamente elaborado:

```typescript
const SELECT_MEMORIES_SYSTEM_PROMPT = `You are selecting memories that will be useful to Claude Code as it processes a user's query. ...
Return a list of filenames for the memories that will clearly be useful (up to 5). Only include memories that you are certain will be helpful...
- If you are unsure if a memory will be useful, do not include it. Be selective and discerning.
- If a list of recently-used tools is provided, do not select memories that are usage reference for those tools (Claude Code is already exercising them). DO still select memories containing warnings, gotchas, or known issues...`
```

(`memdir/findRelevantMemories.ts:13-23`)

**Design chave**: documentação de referência de ferramentas usadas recentemente não deve ser selecionada (não é necessário consultar a documentação quando já está em uso), mas memórias sobre **armadilhas/problemas conhecidos** das mesmas ferramentas ainda devem ser selecionadas (são mais necessárias quando a ferramenta está sendo usada).

A chamada de API usa saída estruturada (JSON Schema) para garantir formato parseável:

```typescript
output_format: {
  type: 'json_schema',
  schema: {
    type: 'object',
    properties: {
      selected_memories: { type: 'array', items: { type: 'string' } },
    },
    required: ['selected_memories'],
    additionalProperties: false,
  },
},
```

(`memdir/findRelevantMemories.ts:97-108`)

### Modo de Injeção de Memória no Contexto

As memórias selecionadas são injetadas com tags `<system-reminder>` antes das mensagens do usuário (`wrapMessagesInSystemReminder`). Memórias com mais de 1 dia recebem um aviso de frescor:

```typescript
export function memoryFreshnessText(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d <= 1) return ''
  return (
    `This memory is ${d} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

(`memdir/memoryAge.ts:38-47`)

Esse design resolve um problema real: usuários relataram "afirmações confiantes baseadas em memórias desatualizadas" — caminhos de arquivo ou nomes de função referenciados já foram modificados, mas as citações na memória faziam as afirmações parecerem mais confiáveis, não mais suspeitas.

**Mecanismo anti-deriva**: `MEMORY_DRIFT_CAVEAT` é injetado no system prompt, exigindo que o Agent verifique o estado atual antes de responder com base em memórias:

> Memory records can become stale over time. Before answering or building assumptions based solely on information in memory records, verify that the memory is still correct by reading the current state of the files or resources. If a recalled memory conflicts with current information, trust what you observe now — and update or remove the stale memory.
>
> (`memdir/memoryTypes.ts:163-167`)

---

## 9.8 Sincronização do Team Memory

### Mecanismo de Sincronização via API REST

O Team Memory é sincronizado no servidor via `services/teamMemorySync/`, com o design da API descrito completamente no início de `index.ts`:

```
GET  /api/claude_code/team_memory?repo={owner/repo}            → TeamMemoryData
GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → apenas metadados+hashes
PUT  /api/claude_code/team_memory?repo={owner/repo}            → upsert entries
404  = ainda sem dados
```

(`services/teamMemorySync/index.ts:10-13`)

Depende de **autenticação OAuth** (requer `CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`), usando repositório GitHub (`owner/repo`) como escopo.

**Mecanismo Watcher**: `watcher.ts` usa `fs.watch({recursive: true})` para monitorar mudanças no diretório team, com debounce de 2 segundos antes de acionar push (`DEBOUNCE_MS = 2000`). Deliberadamente escolhe o `fs.watch` nativo em vez de chokidar:

> chokidar 4+ dropped fsevents, and Bun's fs.watch fallback uses kqueue, which requires one open fd per watched file — with 500+ team memory files that's 500+ permanently-held fds.
>
> (`services/teamMemorySync/watcher.ts:67-71`)

macOS usa FSEvents (O(1) file descriptors), Linux usa inotify (O(número de subdiretórios)), ambos superiores à solução kqueue do chokidar.

### Bloqueio Otimista (If-Match)

O upload usa controle de concorrência otimista, carregando ETag (checksum) via header HTTP `If-Match`:

```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

(`services/teamMemorySync/index.ts:uploadTeamMemory`)

Quando o servidor retorna 412 Precondition Failed, indica conflito (outro usuário modificou a memória compartilhada nesse período). O sistema usa o endpoint `GET ?view=hashes` (leve, retorna apenas hash SHA-256 por chave, sem corpo de conteúdo) para atualizar os `serverChecksums` locais e recalcular o delta para nova tentativa:

```typescript
const MAX_CONFLICT_RETRIES = 2
```

### Estratégia de Resolução de Conflitos

A estratégia é **servidor vence (server wins per-key)** — o conteúdo do servidor sobrescreve o local no Pull. O delta push só envia chaves onde o hash local difere do servidor; o servidor usa semântica upsert (chaves não incluídas no PUT são preservadas).

O limite de upload em lote (`MAX_PUT_BODY_BYTES = 200_000`) evita que o corpo da requisição seja muito grande e rejeitado pelo API Gateway (observado que o gateway retorna HTML 413 em cerca de 256-512KB, diferente do 413 estruturado da camada de aplicação). Quando excede o limite, divide automaticamente em múltiplos PUTs sequenciais; a semântica upsert garante segurança:

```typescript
export function batchDeltaByBytes(
  delta: Record<string, string>,
): Array<Record<string, string>> {
  // empacotamento ganancioso: agrupa por bytes, cada lote não excede MAX_PUT_BODY_BYTES
  ...
}
```

(`services/teamMemorySync/index.ts:batchDeltaByBytes`)

**Supressão de falhas permanentes**: alguns erros (no_oauth, no_repo, 4xx exceto 409/429) não podem ser corrigidos por nova tentativa. Ao detectar esses erros, o sistema define `pushSuppressedReason`, impedindo que os pushes acionados pelo watcher entrem em loop infinito de novas tentativas (observado que um dispositivo sem OAuth emitiu 167K eventos push em 2,5 dias).

---

## 9.9 Análise das Decisões de Design

### Por que Sistema de Arquivos em vez de Banco de Dados

O design de sistema de arquivos + Markdown tem várias vantagens-chave:

1. **Operável diretamente pelo Agent**: as ferramentas FileRead/FileWrite/FileEdit são as ferramentas nativas do Claude; não é necessária nenhuma camada de API adicional. O Agent escreve memórias e código usando o mesmo conjunto de ferramentas, minimizando a carga cognitiva.

2. **Inspecionável pelo usuário**: `~/.claude/projects/.../memory/` é uma pasta comum; o usuário pode diretamente `ls`, `cat`, `vim` — completamente transparente.

3. **Compatível com Git**: arquivos Markdown suportam nativamente diff, grep, git history, conveniente para o cálculo de delta do Team Memory.

4. **Evita abstrações desnecessárias**: banco de dados requer migrações de schema, estratégia de backup, camada de consulta — excesso de engenharia para "alguns centenas de KB de arquivos Markdown".

### Por que Limitar o Tamanho do MEMORY.md

O limite de 200 linhas / 25KB tem dados de medição reais (valores observados em p97/p100). O motivo central:

- MEMORY.md é carregado no system prompt em **cada requisição**; o tamanho impacta diretamente o consumo de tokens
- Um índice muito grande comprime o espaço de contexto realmente útil
- O limite forçado encoraja usuários e Agents a manter o índice refinado, com apenas um "gancho" por linha em vez de conteúdo

Isso é um design clássico de "usar restrições para promover qualidade" — não porque tecnicamente não seja possível acomodar mais, mas para orientar o uso correto via restrições.

### Considerações de Segurança na Memória

O sistema tem múltiplas camadas de design de segurança:

**Proteção contra path traversal**: `teamMemPaths.ts` implementa três níveis de verificação — primeiro verificação em nível de string para `..`, travessia URL encoded, ataques de normalização Unicode, depois resolve symlinks via `realpath` para verificar o caminho real no sistema de arquivos:

```typescript
// PSR M22186: path.resolve() does NOT resolve symlinks.
// An attacker who can place a symlink inside teamDir pointing outside
// would pass a resolve()-based containment check.
```

(`memdir/teamMemPaths.ts:130-133`)

**Varredura de segredos**: ao escrever no Team Memory, `scanForSecrets()` varre 30 padrões de credenciais de alta confiança (da biblioteca de regras gitleaks), incluindo formatos de token das principais plataformas como AWS, GCP, GitHub, Anthropic, OpenAI. A varredura é executada tanto **antes do upload** quanto **antes da escrita**:

- `checkTeamMemSecrets()` de `teamMemSecretGuard.ts` intercepta escritas na fase `validateInput` de FileWriteTool/FileEditTool
- `readLocalTeamMemory()` escaneia novamente antes do push, pulando arquivos com informações sensíveis

**Controle de ferramentas com mínimo privilégio**: a função `canUseTool` do Agent de extração permite apenas:
- FileRead/Grep/Glob (somente leitura)
- Comandos Bash somente leitura (ls/find/cat/stat/wc/head/tail)
- FileEdit/FileWrite, e o caminho deve estar dentro do diretório de memória

```typescript
return denyAutoMemTool(
  tool,
  `only Read, Grep, Glob, read-only Bash, and Edit/Write within ${memoryDir} are allowed`,
)
```

(`services/extractMemories/extractMemories.ts:171-176`)

**Isenção de segurança para ProjectSettings**: a configuração `autoMemoryDirectory` confia apenas nas fontes policy/local/user, excluindo explicitamente projectSettings (`.claude/settings.json`):

> SECURITY: projectSettings (.claude/settings.json committed to the repo) is intentionally excluded — a malicious repo could otherwise set autoMemoryDirectory: "~/.ssh" and gain silent write access to sensitive directories.
>
> (`memdir/paths.ts:113-116`)

---

## 9.10 Padrões Transferíveis

A seguir estão os padrões do sistema de memória do Doramagic que podem ser diretamente adotados:

### Padrão Um: Princípio do Não Derivável

**O que deve ser memorizado**: qualquer informação que possa ser obtida consultando o estado atual (código, arquivos, git) não vale a pena memorizar. A memória só deve armazenar "contexto histórico" — por que essa decisão foi tomada, quais armadilhas foram encontradas, preferências implícitas do usuário.

**Aplicação no Doramagic**: as camadas "UNSAID" e "WHY" extraídas pelo Soul Extractor se alinham naturalmente com esse princípio. Os documentos de regras do OpenClaw são consultáveis e não precisam ser memorizados; mas "esta regra do OpenClaw causou uma falha de publicação" é o tipo de lição que vale a pena memorizar.

### Padrão Dois: Escrita em Dois Passos + Índice Leve

O padrão de escrita em dois passos de arquivo + índice garante que o índice seja sempre refinado (limite forçado de 150 caracteres por linha), enquanto os arquivos de conteúdo podem ser detalhados. O consumo de tokens do índice é fixo; a leitura do conteúdo é sob demanda.

**Aplicação no Doramagic**: o `MEMORY.md` do sistema de memória é semelhante ao "diretório de blocos" do Doramagic — um índice leve carregável, apontando para arquivos detalhados que podem ser expandidos sob demanda.

### Padrão Três: Extração em Segundo Plano com Fork Agent

Não bloquear o diálogo principal, compartilhar prompt cache, maximizar a taxa de acerto de cache — é o padrão padrão para tarefas de pós-processamento em segundo plano. Detalhes de implementação chave:
- `skipTranscript: true` evita escrever no registro da sessão principal
- `maxTurns: N` evita que o Agent fique preso em loops de verificação
- Mecanismo de cursor (`lastMemoryMessageUuid`) garante que apenas incrementos sejam processados cada vez
- Stash + trailing run garantem que mensagens recentes não sejam perdidas quando o Agent está ocupado

### Padrão Quatro: Consciência de Frescor

A memória não é um fato permanentemente válido, mas uma observação com validade temporal. O sistema:
1. Adiciona uma dica de "N dias atrás" ao recuperar
2. Planta instruções anti-deriva no system prompt (verificar antes de citar)
3. Exige que o Agent atualize ativamente memórias desatualizadas em vez de preservá-las

Isso é especialmente relevante para o cenário de "extração de conhecimento" do Doramagic — WHY/UNSAID extraídos ficam desatualizados com a evolução do projeto e precisam de um mecanismo similar para manter o frescor.

### Padrão Cinco: Varredura de Segredos em Etapas Anteriores

Antes de qualquer escrita "cross-boundary" (escrever em espaço compartilhado, upload de rede), os segredos devem ser varridos. A biblioteca de regras gitleaks fornece um conjunto de padrões de alta confiança que pode ser reutilizado diretamente. Design chave: a varredura é executada na fase `validateInput` da ferramenta de escrita (não posteriormente), garantindo que segredos não toquem nenhum caminho de persistência.

---

## 9.11 Índice de Código-Fonte

| Arquivo | Linhas | Responsabilidade Central |
|---------|--------|--------------------------|
| `services/SessionMemory/sessionMemory.ts` | 495 | Lógica principal do Session Memory: julgamento de condição de acionamento, chamada ao Fork Agent, API de acionamento manual |
| `services/SessionMemory/prompts.ts` | 324 | Template do Session Memory, construção do prompt de atualização, análise de tamanho de seção |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | Gerenciamento de estado do Session Memory: configuração, julgamento de limite, utilitários de espera/sincronização |
| `services/extractMemories/extractMemories.ts` | 615 | Extração da Persistent Memory: chamada ao Fork Agent, estado de closure, controle de concorrência |
| `services/extractMemories/prompts.ts` | 154 | Construção do prompt de extração: duas variantes (auto-only e combinada com Team Memory) |
| `memdir/memdir.ts` | 507 | Lógica de truncamento do MEMORY.md, construção do prompt de memória, garantia de criação de Diretório |
| `memdir/paths.ts` | 278 | Resolução de caminho Auto Memory, julgamento de habilitação/desabilitação, verificação de segurança de caminho |
| `memdir/memoryTypes.ts` | 271 | Definições dos quatro tipos de memória, formato frontmatter, princípios de recall/anti-deriva/não derivável |
| `memdir/findRelevantMemories.ts` | 141 | Seleção de recall pelo Sonnet: escanear frontmatter → 5 memórias relevantes |
| `memdir/memoryScan.ts` | 94 | Primitivas de escaneamento de diretório: leitura de frontmatter, formatação de manifesto |
| `memdir/memoryAge.ts` | 53 | Cálculo de frescor: dias, texto legível por humanos, aviso de staleness |
| `memdir/teamMemPaths.ts` | 292 | Caminhos do Team Memory, proteção contra path traversal (três camadas de verificação), resolução de symlinks |
| `memdir/teamMemPrompts.ts` | 100 | Construção do prompt combinado Team Memory + Auto Memory |
| `services/teamMemorySync/index.ts` | 1256 | Núcleo de sincronização: lógica fetch/push, bloqueio otimista, fragmentação em lote, nova tentativa em conflito |
| `services/teamMemorySync/watcher.ts` | 387 | Monitoramento de arquivo: debounce push, supressão de falhas permanentes, ciclo de vida start/stop |
| `services/teamMemorySync/secretScanner.ts` | 324 | 30 regras de varredura de segredos (subconjunto gitleaks), funções utilitárias de redação |
| `services/teamMemorySync/types.ts` | 156 | Schemas Zod: TeamMemoryData, tipos de resultado de sincronização, SkippedSecretFile |
| `services/teamMemorySync/teamMemSecretGuard.ts` | 44 | Interceptação de segredos antes da escrita: integração com validateInput de FileWriteTool/FileEditTool |

**Referência rápida de constantes-chave**:

| Constante | Valor | Localização |
|-----------|-------|-------------|
| `MAX_ENTRYPOINT_LINES` | 200 | `memdir/memdir.ts:30` |
| `MAX_ENTRYPOINT_BYTES` | 25.000 | `memdir/memdir.ts:33` |
| `MAX_SECTION_LENGTH` (Session Memory por seção) | 2.000 tokens | `SessionMemory/prompts.ts:9` |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12.000 tokens | `SessionMemory/prompts.ts:10` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumMessageTokensToInit` | 10.000 | `sessionMemoryUtils.ts:29` |
| `DEFAULT_SESSION_MEMORY_CONFIG.minimumTokensBetweenUpdate` | 5.000 | `sessionMemoryUtils.ts:30` |
| `DEFAULT_SESSION_MEMORY_CONFIG.toolCallsBetweenUpdates` | 3 | `sessionMemoryUtils.ts:31` |
| Limite de recall | 5 entradas | `findRelevantMemories.ts:SELECT_MEMORIES_SYSTEM_PROMPT` |
| Limite de arquivos de memória | 200 | `memoryScan.ts:MAX_MEMORY_FILES` |
| Linhas de leitura de frontmatter | 30 | `memoryScan.ts:FRONTMATTER_MAX_LINES` |
| Timeout do Team Memory | 30.000ms | `teamMemorySync/index.ts:TEAM_MEMORY_SYNC_TIMEOUT_MS` |
| Atraso de debounce do push | 2.000ms | `teamMemorySync/watcher.ts:DEBOUNCE_MS` |
| Tamanho máximo por arquivo | 250.000 bytes | `teamMemorySync/index.ts:MAX_FILE_SIZE_BYTES` |
| Tamanho máximo do corpo da requisição PUT | 200.000 bytes | `teamMemorySync/index.ts:MAX_PUT_BODY_BYTES` |
| Número de regras de varredura de segredos | 30 | `secretScanner.ts:SECRET_RULES` |
