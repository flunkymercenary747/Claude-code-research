# Relatório de Análise Panorâmica da Arquitetura do Claude Code

> **Linha de base do código-fonte**: Snapshot do código-fonte TypeScript do Claude Code (2026-03-31, ~512K LOC, ~1.900 arquivos)
> **Data de análise**: 2026-04-02
> **Escala do relatório**: 14 capítulos, 428KB

---

## Contexto do Projeto

### Origem do Código-Fonte

Este relatório é baseado no snapshot completo do código-fonte TypeScript do Claude Code vazado em 2026-03-31. O snapshot contém 512.664 linhas de código TypeScript (`.ts` + `.tsx`), distribuídas em 1.884 arquivos, abrangendo 35 diretórios de nível superior. O código-fonte está armazenado no servidor mini em `~/Documents/openclaw/Doramagic/docs/research/claude-code-instructkr/src/`.

### Metodologia de Análise

Adotou-se a arquitetura de **14 subagentes em análise paralela**: 5 agentes Opus responsáveis pelos capítulos de maior complexidade central (visão geral da arquitetura, Query Engine, orquestração de Agentes, modelo de segurança, gerenciamento de contexto), e 9 agentes Sonnet para os capítulos restantes. Cada agente leu o código-fonte de forma independente, extraiu padrões arquiteturais e validou as conclusões da análise competitiva, sendo revisado pelo editor-chefe ao final.

Essa própria metodologia é uma validação prática da arquitetura multiagente do Claude Code (Capítulo 4) — usando o Claude Code para analisar o Claude Code.

### Comparação com win4r/cc-notebook

win4r/cc-notebook é uma análise da comunidade sobre o mesmo código-fonte. Este relatório realiza melhorias significativas nas seguintes dimensões:

- **Capítulo independente sobre o Sistema de Ferramentas** (Capítulo 3): o cc-notebook não analisou separadamente o sistema de ferramentas; este relatório preenche essa lacuna crítica
- **Validação a nível de código-fonte**: cada afirmação arquitetural é acompanhada de nome de arquivo, número de linha e trecho de código, não de relato secundário
- **Ancoragem teórica**: cada capítulo abre com fundamentos teóricos acadêmicos (teoria da informação, teoria de cache, ciências cognitivas etc.), situando as implementações de engenharia em um quadro de conhecimento mais amplo

---

## Diagrama Arquitetural Panorâmico

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Arquitetura em Camadas do Claude Code            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     Camada de Interação com o Usuário           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │ Terminal  │  │ Command  │  │  Slash   │  │  Vim Mode     │  │    │
│  │  │ UI (Ink)  │  │ Parser   │  │ Commands │  │  (máquina     │  │    │
│  │  │  Cap.12   │  │  Cap.7   │  │  Cap.7   │  │  de estados)  │  │    │
│  │  │           │  │          │  │          │  │  Cap.12       │  │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────────────┘  │    │
│  └───────┼──────────────┼──────────────┼──────────────────────────┘    │
│          │              │              │                                 │
│  ┌───────▼──────────────▼──────────────▼──────────────────────────┐    │
│  │                     Camada de Orquestração de Sessão            │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │    │
│  │  │ Query Engine │  │    Agent     │  │  Prompt Engineering  │ │    │
│  │  │  (loop princ.)│  │ Orchestrator │  │  (orquestração do   │ │    │
│  │  │  Cap.2        │  │  Cap.4        │  │  prompt de sistema) │ │    │
│  │  │               │  │               │  │  Cap.5              │ │    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │    │
│  └─────────┼────────────────┼──────────────────────┼─────────────┘    │
│            │                │                      │                    │
│  ┌─────────▼────────────────▼──────────────────────▼─────────────┐    │
│  │                     Camada de Execução de Capacidades          │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │ Sistema  │  │  Sistema │  │  MCP     │  │  Bash/Read/  │  │    │
│  │  │ de       │  │  de      │  │ Client   │  │  Edit/Grep   │  │    │
│  │  │ Ferramen.│  │  Skills  │  │  Cap.11  │  │  (ferramen.  │  │    │
│  │  │  Cap.3   │  │  Cap.6   │  │          │  │  integradas) │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     Camada de Persistência de Estado           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │    │
│  │  │ Gestão de    │  │   Sistema de │  │  Retomada de     │    │    │
│  │  │ Contexto     │  │   Memória    │  │  Sessão          │    │    │
│  │  │ (compressão  │  │  Cap.9       │  │  Cap.9           │    │    │
│  │  │ de contexto) │  │              │  │                  │    │    │
│  │  │  Cap.8       │  │              │  │                  │    │    │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘    │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐    │
│  │                     Camada de Infraestrutura                   │    │
│  │  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐ │    │
│  │  │ Segurança│  │ Rastreador   │  │ Feature  │  │  OTel    │ │    │
│  │  │ e Perm.  │  │ de Custo e   │  │  Flags   │  │Telemetria│ │    │
│  │  │  Cap.10  │  │ Sel. Modelo  │  │  Cap.14  │  │  Cap.14  │ │    │
│  │  │          │  │  Cap.13      │  │          │  │          │ │    │
│  │  └──────────┘  └──────────────┘  └──────────┘  └──────────┘ │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Dependências Centrais entre Subsistemas

```
Query Engine ──────► Claude API (claude.ts, 3.419 linhas)
     │
     ├──► Sistema de Ferramentas ──► 40+ ferramentas integradas
     │         │
     │         ├──► MCP Client ──► servidores MCP externos
     │         │
     │         └──► Sistema de Skills ──► definições de fluxo em Markdown
     │
     ├──► Agent Orchestrator ──► motor compartilhado runAgent()
     │         │
     │         ├──► Subagent (em processo)
     │         ├──► Coordinator Mode (notificação assíncrona)
     │         └──► Team Mode (caixa de correio em arquivo / Tmux)
     │
     ├──► Gestão de Contexto ──► microcompactação → compactação total (degradação progressiva)
     │
     ├──► Camada de Segurança ──► 8 camadas de defesa em profundidade
     │         │
     │         ├──► Análise AST (parser Bash)
     │         ├──► Sandbox (sandbox-exec / bwrap)
     │         └──► Permission Rules + Hooks
     │
     └──► Rastreador de Custo ──► faturamento em tempo real por modelo
```

### Fluxo de Dados (ciclo de vida de uma interação do usuário)

```
Entrada do usuário → Command Parser → roteamento REPL
                               │
                    ┌──────────┴──────────┐
                    ▼                      ▼
              Comando slash           Entrada em linguagem natural
              (execução imediata)          │
                                          ▼
                                  Montagem do Prompt
                                  (prompt de sistema + CLAUDE.md
                                   + descrições de ferramentas + histórico)
                                          │
                                          ▼
                              ┌── Loop principal do Query Engine ──┐
                              │                                     │
                              ▼                                     │
                        Requisição streaming à Claude API           │
                              │                                     │
                              ▼                                     │
                     ┌── Classificação da resposta ──┐             │
                     │                               │             │
                     ▼                               ▼             │
                  Saída de texto               Chamada de tool     │
                  (renderizada na UI)               │              │
                                                    ▼              │
                                           Verificação de perm.    │
                                           (Security)              │
                                                    │              │
                                           ┌────────┴──────┐       │
                                           ▼               ▼       │
                                        Permitido      Negado/     │
                                           │           Perguntar   │
                                           ▼               │       │
                                      Execução de tool     │       │
                                           │               │       │
                                           ▼               ▼       │
                                  Resultado injetado ──────────────►│
                                           │                        │
                                           ▼                        │
                                  Verificação da janela de contexto │
                                  ──► Acionar compactação?          │
                                           │                        │
                                           └────────────────────────┘
                                                 (loop até o modelo parar)
```

---

## Estatísticas de Tamanho do Código

### Distribuição do Código por Subsistema

| Subsistema | Linhas de código centrais | Arquivos centrais | Proporção |
|--------|-------------|-----------|------|
| Segurança e Permissões (Cap.10) | ~25.000 | 30+ | 4,9% |
| Integração MCP (Cap.11) | ~12.310 | 10+ | 2,4% |
| Orquestração de Agentes (Cap.4) | ~8.700 | 12 | 1,7% |
| Query Engine (Cap.2) | ~7.418 | 8 | 1,4% |
| Sistema de Memória (Cap.9) | ~5.700 | 17 | 1,1% |
| Gestão de Contexto (Cap.8) | ~6.000 | 13+ | 1,2% |
| Sistema de Ferramentas (Cap.3) | ~4.000+ | 40+ diretórios | 0,8%+ |
| Camada API (claude.ts) | 3.419 | 1 | 0,7% |
| **Subtotal acima** | **~72.500+** | — | **~14%** |
| Restante (UI/renderização/Skills/comandos/config. etc.) | ~440.000 | — | ~86% |
| **Total** | **512.664** | **1.884** | **100%** |

### Top 20 Arquivos Centrais (por número de linhas)

| Posição | Arquivo | Linhas | Subsistema |
|------|------|------|-----------|
| 1 | `services/api/claude.ts` | 3.419 | Chamada de API/streaming/retry/cache |
| 2 | `services/mcp/client.ts` | 3.348 | Gerenciamento de conexão MCP |
| 3 | `services/mcp/auth.ts` | 2.465 | Autenticação OAuth MCP |
| 4 | `services/teamMemorySync/` (5 arquivos) | 2.167 | Sincronização de memória de equipe |
| 5 | `query.ts` | 1.729 | Loop principal de consulta |
| 6 | `memdir/` (7 arquivos) | 1.736 | Gerenciamento de diretório de memória |
| 7 | `services/tools/toolExecution.ts` | 1.745 | Motor de execução de ferramentas |
| 8 | `services/mcp/config.ts` | 1.578 | Gerenciamento de configuração MCP |
| 9 | `inProcessRunner.ts` | 1.552 | Backend InProcess do Agent |
| 10 | `AgentTool.tsx` | 1.397 | Ponto de entrada unificado do Agent |
| 11 | `QueryEngine.ts` | 1.295 | Gerenciamento de estado a nível de sessão |
| 12 | `teammateMailbox.ts` | 1.183 | Protocolo de caixa de correio em arquivo |
| 13 | `utils/collapseReadSearch.ts` | 1.109 | Colapso de resultados de leitura/busca |
| 14 | `spawnMultiAgent.ts` | 1.093 | Criação de múltiplos Agentes |
| 15 | `utils/toolResultStorage.ts` | 1.040 | Armazenamento a frio de resultados de ferramentas |
| 16 | `runAgent.ts` | 973 | Motor de execução de Agentes |
| 17 | `SendMessageTool.ts` | 917 | Roteamento de 5 vias de mensagens |
| 18 | `UI.tsx` (Agent) | 871 | Renderização de progresso do Agent |
| 19 | `Tool.ts` | 792 | Abstração central de ferramentas |
| 20 | `extractMemories/` (2 arquivos) | 769 | Extração de memória |

---

## Guia de Leitura dos Capítulos

### Capítulo 1: Visão Geral da Arquitetura e Fluxo de Inicialização
**Arquivo**: [01-architecture-overview.md](01-architecture-overview.md)

**Insight central**: O Claude Code é um aplicativo TUI de terminal baseado em React/Ink, com Bun como runtime principal, acionando a Claude API por meio de um loop REPL para executar tarefas de programação agentic.

**Descobertas-chave**:
- Escolha de stack tecnológica precisa: Bun + TypeScript + React/Ink + Zod v4 + OpenTelemetry, cada escolha com justificativa de engenharia clara
- 512.664 linhas de código distribuídas em 1.884 arquivos e 35 diretórios de nível superior — um projeto de engenharia maduro e de grande escala
- O sistema de feature flags adota um esquema duplo: GrowthBook + bun:bundle, realizando corte de funcionalidades internas em tempo de compilação

**Prioridade de leitura recomendada**: ★★★★★ — Ponto de partida obrigatório para construir um modelo mental global

---

### Capítulo 2: Query Engine — Núcleo de Interação com LLM
**Arquivo**: [02-query-engine.md](02-query-engine.md)

**Insight central**: O Query Engine é o "coração" do Claude Code — 7.400 linhas de código (apenas 1,4%) acionam o caminho mais crítico do produto: cada interação entre o usuário e Claude passa por aqui.

**Descobertas-chave**:
- A arquitetura central é um Async Generator Pipeline (pipeline de corrotinas), orquestrando respostas em streaming, chamadas de ferramentas, compressão de contexto e rastreamento de custos em um único async generator
- Trata pelo menos 12 tipos de exceção: estouro da janela de contexto, esgotamento de orçamento de tokens, falhas de API, interrupções do usuário, aprovações de permissão, etc.
- O mecanismo de orçamento de tokens e decisão de auto-continue garante que tarefas longas não sejam interrompidas pelo stop do modelo

**Prioridade de leitura recomendada**: ★★★★★ — Chave para entender o funcionamento de todo o produto

---

### Capítulo 3: Sistema de Ferramentas
**Arquivo**: [03-tool-system.md](03-tool-system.md)

**Insight central**: O sistema de ferramentas é o único canal entre a intenção do LLM e o mundo real — não é um acessório, mas um ativo central de engenharia.

**Descobertas-chave**:
- Padrão Self-Describing Tools: cada ferramenta auto-declara suas capacidades por meio de campos como `searchHint`, `prompt` e `userFacingSchema`, permitindo seleção dinâmica pelo modelo
- 40+ subdiretórios de ferramentas, todos orquestrados pelo `toolExecution.ts` (1.745 linhas)
- o cc-notebook não analisou separadamente este sistema; este capítulo preenche a lacuna crítica

**Prioridade de leitura recomendada**: ★★★★☆ — Leitura obrigatória para desenvolvedores de Agentes

---

### Capítulo 4: Orquestração de Agentes e Arquitetura Multiagente
**Arquivo**: [04-agent-orchestration.md](04-agent-orchestration.md)

**Insight central**: Três modos progressivos de colaboração (Subagent / Coordinator / Team Mode) compartilham o mesmo motor central `runAgent()`, realizando comportamentos distintos por combinação de parâmetros — uma das decisões de design mais elegantes.

**Descobertas-chave**:
- 8.700 linhas de código, 12 módulos centrais — o subsistema mais complexo de toda a arquitetura do produto
- O Team Mode usa um protocolo de caixa de correio em arquivo (`teammateMailbox.ts`, 1.183 linhas) para colaboração paralela persistente
- O Coordinator Mode usa XML `<task-notification>` para comunicação totalmente assíncrona, com isolamento via AbortController independente

**Prioridade de leitura recomendada**: ★★★★★ — Caso de referência para design de sistemas multiagentes

---

### Capítulo 5: Engenharia de Prompt
**Arquivo**: [05-prompt-engineering.md](05-prompt-engineering.md)

**Insight central**: A engenharia de Prompt é o subsistema de maior complexidade implícita em todo o sistema — um ajuste de 3 linhas em um `systemPromptSection` pode afetar simultaneamente a qualidade do comportamento do modelo, a taxa de acerto do Prompt Cache, a cobrança de tokens e a consistência entre sessões.

**Descobertas-chave**:
- O prompt de sistema de 8.000+ tokens não é uma "descrição", mas uma programação precisa de comportamento
- A estratégia em camadas do Prompt Cache (`cacheScope: 'org'` / `'global'`) reduz o custo de tokens de milhões de requisições por uma ordem de magnitude
- O marcador `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` delimita com precisão o limite do prefixo compartilhado, maximizando a taxa de acerto do cache global

**Prioridade de leitura recomendada**: ★★★★★ — Leitura obrigatória para engenheiros de Prompt, referência para design de Prompt em nível comercial

---

### Capítulo 6: Sistema de Skills
**Arquivo**: [06-skill-system.md](06-skill-system.md)

**Insight central**: Skills são "SOP para IA" — fluxos de trabalho de especialistas humanos codificados em formato Markdown, conferindo à IA capacidade de execução profissional e reprodutível.

**Descobertas-chave**:
- Fusão declarativa + executiva: frontmatter declara metadados (permissões, modelo, condições de acionamento); o corpo contém as instruções de execução
- Carregamento de múltiplas fontes e mesclagem por prioridade: bundled / nível-usuário / nível-projeto / nível-Plugin / origem MCP
- Mecanismo de ativação condicional: ativação automática por caminho de arquivo via frontmatter `paths`, com suporte a descoberta dinâmica

**Prioridade de leitura recomendada**: ★★★★☆ — Referência direta para o design de Skills do Doramagic

---

### Capítulo 7: Sistema de Comandos
**Arquivo**: [07-command-system.md](07-command-system.md)

**Insight central**: O sistema de comandos reflete uma clara separação de responsabilidades — comandos são responsáveis pelo "disparo", ferramentas pela "execução" e o LLM pela "decisão".

**Descobertas-chave**:
- Três tipos de comandos: PromptCommand (injeção no diálogo LLM), LocalCommand (execução local) e LocalJSXCommand (renderização com UI)
- Design de carregamento lazy: comandos carregados via `load(): Promise<Module>` sob demanda, distribuindo o custo de inicialização para a primeira chamada
- Duas extensões-chave do padrão Command clássico GoF: carregamento lazy + valor de retorno tipado

**Prioridade de leitura recomendada**: ★★☆☆☆ — Aplicação de padrão de design convencional; leitura conforme necessidade

---

### Capítulo 8: Gestão de Contexto
**Arquivo**: [08-context-management.md](08-context-management.md)

**Insight central**: A gestão de contexto é essencialmente um problema de compressão de informação — o Claude Code projeta um sistema progressivo de degradação, do sem perdas ao com perdas, para manter a continuidade de sessões de horas dentro da janela de 200K tokens.

**Descobertas-chave**:
- Estratégia de compressão em três camadas: microcompactação (cache_edits, sem perdas) → colapso de resultados de ferramentas (collapseReadSearch) → compactação total via Fork Agent (com perdas)
- Os 9 capítulos do Prompt de resumo definem implicitamente uma função de distorção — quais informações são mais intolerantes à perda
- Invariantes de segurança: o par tool_use/tool_result não pode ser cortado, proteção recursiva, mecanismo de Circuit Breaker

**Prioridade de leitura recomendada**: ★★★★☆ — Referência crítica para desenvolvimento de Agentes de sessão longa

---

### Capítulo 9: Sistema de Memória
**Arquivo**: [09-memory-system.md](09-memory-system.md)

**Insight central**: O sistema de memória permite ao Claude manter continuidade entre sessões — mapeando a tricotomia da ciência cognitiva (memória de trabalho / memória episódica / memória semântica) para uma arquitetura de três camadas (janela de contexto / Session Memory / Persistent Memory).

**Descobertas-chave**:
- 5.700 linhas de código implementam o ciclo de vida completo da memória entre sessões: extração, armazenamento, sincronização e carregamento
- A sincronização de memória de equipe (`teamMemorySync/`, 2.167 linhas) é o maior módulo único, suportando cenários de colaboração multipessoal
- O diretório `memdir/` implementa uma estrutura de organização de memória semelhante a um sistema de arquivos

**Prioridade de leitura recomendada**: ★★★☆☆ — Referência para Agentes que necessitam de memória de longo prazo

---

### Capítulo 10: Modelo de Segurança e Permissões
**Arquivo**: [10-security-permission.md](10-security-permission.md)

**Insight central**: O sistema de segurança é o subsistema com maior volume de código (~25.000 linhas), implementando 8 camadas de defesa em profundidade — do parser AST ao sandbox do SO, cada camada independente e acumulável.

**Descobertas-chave**:
- Modelo de ameaças único: o próprio modelo de IA pode ser induzido por prompt injection a executar operações perigosas
- Parser de AST Bash em TypeScript puro para entender a estrutura de comandos (expressões regulares não conseguem distinguir `echo "rm -rf /"` de `rm -rf /`)
- Sandbox duplo: verificação de permissão na camada da aplicação + isolamento na camada do SO (macOS sandbox-exec / Linux bwrap), com o SO servindo de rede de segurança mesmo que a camada da aplicação falhe

**Prioridade de leitura recomendada**: ★★★★★ — Leitura obrigatória para engenheiros de segurança, referência de design de segurança para AI Agents

---

### Capítulo 11: Integração MCP
**Arquivo**: [11-mcp-integration.md](11-mcp-integration.md)

**Insight central**: O Claude Code é um cliente MCP puro — 12.310 linhas de código implementam gerenciamento completo de conexões, autenticação OAuth, descoberta de ferramentas e registro dinâmico.

**Descobertas-chave**:
- Ferramentas MCP são registradas dinamicamente no formato `mcp__<serverName>__<toolName>`, compartilhando o mesmo framework de execução das ferramentas integradas
- Autenticação OAuth 2.0 com suporte a XAA (Cross-Application Access)
- Suporte a quatro camadas de transporte: stdio / SSE / HTTP Streamable / WebSocket

**Prioridade de leitura recomendada**: ★★★☆☆ — Leitura obrigatória para desenvolvedores do ecossistema MCP

---

### Capítulo 12: UI de Terminal e Motor de Renderização
**Arquivo**: [12-terminal-ui.md](12-terminal-ui.md)

**Insight central**: O Claude Code realiza uma profunda transformação do framework React/Ink — renderização com buffer duplo, motor de diff ANSI, aceleração de rolagem por hardware, posicionamento de cursor com consciência de IME, alcançando uma experiência interativa próxima a GUI em um terminal de texto puro.

**Descobertas-chave**:
- Intervalo entre frames do pipeline de renderização `FRAME_INTERVAL_MS = 16` (teoricamente 62,5 fps), alinhado com a taxa de atualização do monitor
- Implementação completa do modo Vim: máquina de estados independente, cobrindo modos NORMAL/INSERT, conjunto completo de operators/motions/textObjects
- Renderizador Ink personalizado (`src/ink/`) substitui o pipeline de renderização nativo do Ink

**Prioridade de leitura recomendada**: ★★☆☆☆ — Referência aprofundada para desenvolvedores de frameworks TUI; outros leitores podem ignorar

---

### Capítulo 13: Seleção de Modelos e Controle de Custos
**Arquivo**: [13-model-cost.md](13-model-cost.md)

**Insight central**: Três princípios de design — prioridade à intenção do usuário (cadeia de sobreposição em múltiplas camadas), transparência total de custos (exibição obrigatória de valores), nenhum downgrade silencioso (Overload Fallback deve alertar).

**Descobertas-chave**:
- Cadeia de prioridade de seleção de modelo: comando `/model` → flag `--model` → variável de ambiente → arquivo de configuração, camadas superiores sobrepõem as inferiores
- Tabela de preços precisos integrada (`modelCost.ts`), com rastreamento de custo em tempo real por modelo
- O Overload Fallback de Opus → Sonnet nunca muda silenciosamente; deve exibir um aviso ao usuário

**Prioridade de leitura recomendada**: ★★★☆☆ — Referência prática para design de roteamento em sistemas multimodelos

---

### Capítulo 14: Feature Flags e Observabilidade
**Arquivo**: [14-feature-flags-observability.md](14-feature-flags-observability.md)

**Insight central**: Feature Flags de esquema duplo (Dead Code Elimination em tempo de compilação via bun:bundle + GrowthBook em tempo de execução) combinados com os três pilares do OpenTelemetry, construindo um sistema de observabilidade de ponta a ponta, do corte em compilação ao rastreamento em execução.

**Descobertas-chave**:
- Flags em tempo de compilação removem completamente o código interno via Dead Code Elimination, prevenindo engenharia reversa
- Analytics de dupla via: Datadog (monitoramento externo) + log de eventos interno (BigQuery interno), prefixo de eventos `tengu_*`
- Design com privacidade em primeiro lugar: conteúdo de código e caminhos de arquivo não são registrados por padrão

**Prioridade de leitura recomendada**: ★★☆☆☆ — Referência para design de observabilidade de SaaS em larga escala

---

## Recomendações de Leitura

### Se você é desenvolvedor de AI Agent

**Rota recomendada**: Cap.1 → Cap.2 → Cap.4 → Cap.3 → Cap.8 → Cap.10

O que você mais precisa entender é o padrão de orquestração async generator do Query Engine (Cap.2) e a arquitetura de colaboração de Agentes em três camadas (Cap.4). O sistema de ferramentas (Cap.3) demonstra como projetar interfaces de ferramentas auto-descritivas e extensíveis. A gestão de contexto (Cap.8) resolve os desafios centrais de engenharia em cenários de sessão longa. O modelo de segurança (Cap.10) aborda o problema de fronteira de confiança incontornável em AI Agents.

**Ganho central**: A implementação completa de engenharia do loop de Agent, e quão grande é a lacuna entre "o demo funciona" e "pronto para produção".

---

### Se você é engenheiro de Prompt

**Rota recomendada**: Cap.5 → Cap.6 → Cap.8 → Cap.9 → Cap.2

A engenharia de prompt de sistema (Cap.5) é o núcleo — programação de comportamento com 8.000+ tokens, Prompt Cache em camadas, marcadores de limite dinâmico. O sistema de Skills (Cap.6) mostra como estruturar e consolidar conhecimento de fluxo de trabalho. A gestão de contexto (Cap.8) revela a estratégia de retenção de informação em cenários de compressão/resumo de Prompts. O sistema de memória (Cap.9) é a solução de persistência do contexto de Prompt entre sessões.

**Ganho central**: Engenharia de Prompt em nível comercial vai muito além de "escrever uma descrição" — é uma disciplina cruzada de programação de comportamento, otimização de cache e controle de custos.

---

### Se você é engenheiro de segurança

**Rota recomendada**: Cap.10 → Cap.3 → Cap.11 → Cap.2 → Cap.14

O modelo de segurança (Cap.10) é o mais importante — 25.000 linhas de código, 8 camadas de defesa em profundidade, análise AST de Bash, sandbox do SO. O sistema de ferramentas (Cap.3) mostra como a verificação de permissão é embutida no pipeline de execução de ferramentas. A integração MCP (Cap.11) envolve fronteiras de confiança de extensões de terceiros. O Query Engine (Cap.2) e seu tratamento de 12 tipos de exceção revela a superfície de ataque. A observabilidade (Cap.14) mostra a infraestrutura de rastreamento de eventos de segurança.

**Ganho central**: A segurança de AI Agent é um modelo de ameaças inteiramente novo — o próprio modelo pode tornar-se um vetor de ataque; validação de entrada tradicional é insuficiente.

---

### Se você é desenvolvedor do Doramagic

**Rota recomendada**: Cap.6 → Cap.3 → Cap.5 → Cap.4 → Cap.9

O sistema de Skills (Cap.6) é diretamente comparável à forma de Skills do Doramagic — frontmatter declarativo + corpo executivo, carregamento de múltiplas fontes, ativação condicional: esses padrões de design são diretamente aproveitáveis. O sistema de ferramentas (Cap.3) e seu padrão auto-descritivo tem valor de referência para o design de ferramentas do OpenClaw. A engenharia de Prompt (Cap.5) demonstra estratégias de gerenciamento de prompts de sistema em larga escala. A orquestração de Agentes (Cap.4) inspira pipelines futuros de extração multiagente. O sistema de memória (Cap.9) tem valor de referência para o sistema de acumulação de experiência do Doramagic.

**Ganho central**: O sistema de Skills do Claude Code valida que o caminho de "SOP para IA" é correto — o design de Skills do Doramagic pode avançar nos ombros de gigantes.

---

## Glossário

| Termo | Definição |
|------|------|
| **Agentic Loop** | O LLM alternando ciclicamente "raciocínio → chamada de tool → injeção de resultado → novo raciocínio" até a conclusão da tarefa. Modo de operação central do Claude Code. |
| **Async Generator Pipeline** | Corrotina TypeScript `async function*`, usada pelo Query Engine para orquestrar a produção gradual de respostas em streaming. |
| **Auto-Continue** | Quando o modelo para devido a limites de token (`end_turn` mas com chamadas de ferramentas incompletas), o Query Engine injeta automaticamente uma instrução de continuação sem interromper a experiência do usuário. |
| **Cache Editing** | Capacidade de edição de cache da Claude API, permitindo excluir cópias em cache de mensagens antigas sem invalidar todo o cache. Base para microcompactação. |
| **CLAUDE.md** | Arquivo de instruções de nível usuário/projeto, similar ao `.editorconfig` mas voltado para IA. Injetado no prompt de sistema para orientar o comportamento do modelo. |
| **Coordinator Mode** | Segundo modo de orquestração de Agentes: comunicação totalmente assíncrona, coordenando múltiplos Agentes via notificações XML, cada um com AbortController independente. |
| **DCE (Dead Code Elimination)** | Remoção de código inacessível em tempo de compilação. A chamada `feature()` do bun:bundle faz com que funcionalidades internas sejam completamente deletadas em builds externos. |
| **Defense in Depth** | Estratégia de defesa em profundidade. O sistema de 8 camadas de segurança do Claude Code: modo de permissão → correspondência de regras → análise AST → validador de segurança → validação de caminho → classificador → Hook → sandbox. |
| **Fork Agent** | Agente independente iniciado durante a compactação total do contexto, responsável por ler o histórico completo da conversa e gerar um resumo estruturado, sem poluir a sessão atual. |
| **Frontmatter** | Bloco de metadados YAML no cabeçalho de um arquivo Markdown (delimitado por `---`). Arquivos de Skills o usam para declarar permissões, modelo, condições de acionamento, etc. |
| **GrowthBook** | Plataforma de Feature Flags em tempo de execução, com suporte a testes A/B e lançamentos graduais. O Claude Code o usa para controlar dinamicamente switches de funcionalidades e taxas de amostragem. |
| **Ink** | Framework de UI de terminal baseado em React. O Claude Code modificou profundamente seu pipeline de renderização (buffer duplo, diff ANSI, rolagem por hardware). |
| **MCP (Model Context Protocol)** | Protocolo aberto liderado pela Anthropic que define a comunicação padronizada entre aplicações de IA e serviços de ferramentas externas. Baseado em JSON-RPC 2.0. |
| **Micro-Compaction (Microcompactação)** | Primeira estratégia de gestão de contexto: limpa o cache de resultados antigos de ferramentas no servidor via cache editing, mantendo um placeholder local. Sem perdas. |
| **Overload Fallback** | Quando o modelo principal (ex.: Opus) está sobrecarregado, degradação transparente para um modelo secundário (ex.: Sonnet) — deve exibir aviso ao usuário, nunca mudar silenciosamente. |
| **Prompt Cache** | Mecanismo de cache de prompts da Claude API. Requisições com o mesmo prefixo compartilham cache, reduzindo significativamente o custo de tokens. Disponível nos escopos `org` e `global`. |
| **REPL** | Read-Eval-Print Loop, modo de interação central do Claude Code: ler entrada do usuário → processar com LLM → produzir saída → aguardar próxima rodada. |
| **Skill** | Definição de fluxo de trabalho em formato Markdown, acionada por comandos slash ou chamada proativa pela IA. Essencialmente uma "SOP para IA". |
| **Subagent** | Primeiro modo de orquestração de Agentes: subagente executado de forma síncrona/assíncrona dentro do processo principal, comunicando-se via valores de retorno de função. |
| **System Reminder** | Mensagem de nível de sistema injetada no histórico de conversa, usada para atualizar dinamicamente as informações de contexto do modelo (como data atual, lista de Skills disponíveis). |
| **Team Mode** | Terceiro modo de orquestração de Agentes: colaboração paralela persistente via protocolo de caixa de correio em arquivo, com suporte a três backends: Tmux Pane / InProcess / Remote. |
| **Tool Result Storage** | Armazenamento a frio em disco de resultados de execução de ferramentas (`toolResultStorage.ts`). Resultados de ferramentas após microcompactação são movidos do contexto para cá e recuperados sob demanda. |
| **TUI (Terminal User Interface)** | Interface de usuário de terminal — a interface interativa do Claude Code implementada em um terminal de texto puro, com renderização declarativa baseada em React/Ink. |

---

*Este relatório foi produzido em paralelo por 14 agentes de análise e revisado pelo editor-chefe. Em caso de discrepância, o código-fonte prevalece.*
