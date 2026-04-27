# Capítulo 11: Integração MCP

## 11.1 Visão Geral e Posicionamento

### O que é MCP

MCP (Model Context Protocol) é um protocolo aberto liderado pela Anthropic que define o formato padronizado de comunicação entre aplicações de IA e serviços externos de ferramentas. Essencialmente é um protocolo JSON-RPC 2.0, executando sobre múltiplas camadas de transporte (stdio, SSE, HTTP Streamable, WebSocket), especificando formatos de mensagem padrão para descoberta de ferramentas (`tools/list`), chamada de ferramentas (`tools/call`), gerenciamento de recursos (`resources/list`/`resources/read`), templates de Prompt (`prompts/list`/`prompts/get`) e outros.

### O Papel do MCP no Claude Code

As ferramentas integradas do Claude Code (Bash, Read, Edit, etc.) cobrem cenários de sistema de arquivos e desenvolvimento local. O MCP é posicionado como uma **interface aberta de extensão de ferramentas**: qualquer serviço de terceiros (Slack, GitHub, Jira, bancos de dados, automação de browser, etc.) pode implementar um servidor MCP, e o Claude Code, ao se conectar via protocolo padrão, pode chamar essas capacidades externas sem modificar o código central.

Na arquitetura, o Claude Code é um **cliente MCP** puro — não implementa nenhuma capacidade de servidor MCP (exceto responder a requisições `roots/list` para informar ao servidor o diretório de trabalho). As ferramentas de cada servidor MCP conectado são dinamicamente registradas como objetos Tool no formato `mcp__<serverName>__<toolName>`, compartilhando o mesmo framework de execução das ferramentas integradas.

### Volume de Código

A integração MCP envolve aproximadamente 12.310 linhas de código TypeScript, distribuídas nos seguintes arquivos:

| Arquivo | Linhas | Responsabilidade |
|---------|--------|------------------|
| `services/mcp/client.ts` | 3.348 | Gerenciamento de conexão, descoberta de ferramentas, núcleo de execução |
| `services/mcp/config.ts` | 1.578 | Gerenciamento de configuração (fusão de múltiplas fontes, filtragem por política) |
| `services/mcp/auth.ts` | 2.465 | Autenticação OAuth 2.0 (incluindo XAA de acesso entre aplicações) |
| `services/mcp/utils.ts` | 575 | Filtragem de ferramentas, hash de nomes, detecção de Stale |
| `services/mcp/types.ts` | 258 | Definições de tipos (Transport, ServerConfig, estado de conexão) |
| `tools/MCPTool/classifyForCollapse.ts` | 604 | Classificação para colapso na UI (identificação de ferramentas Search/Read) |
| `tools/MCPTool/UI.tsx` | 402 | Renderização de resultados de execução de ferramenta |
| `services/mcp/channelPermissions.ts` | 240 | Relay de permissões de Channel |
| `services/mcp/channelNotification.ts` | 316 | Mecanismo de push de mensagens de Channel |
| `services/mcp/elicitationHandler.ts` | 313 | Tratamento de Elicitation (interação via formulário/URL) |
| `skills/mcpSkillBuilders.ts` | 44 | Registro de construtores de Skill (desacoplamento do grafo de dependências) |

---

## 11.2 Bases Teóricas

### Padrão de Extensão de Ferramentas Orientado a Protocolo

Sistemas de plugin tradicionais geralmente dependem de SDK fornecido pela aplicação hospedeira; desenvolvedores de plugins devem conhecer as interfaces internas do hospedeiro. MCP adota o padrão **orientado a protocolo (protocol-driven)**: todas as interações entre hospedeiro (Claude Code) e plugin (servidor MCP) são realizadas via mensagens JSON-RPC padrão, permitindo que ambos evoluam de forma independente.

Isso está altamente alinhado com a filosofia de design do LSP (Language Server Protocol):

| Dimensão | LSP | MCP |
|----------|-----|-----|
| Padrão central | Editor ↔ servidor de linguagem | AI Agent ↔ servidor de ferramentas |
| Mecanismo de descoberta | Troca de capabilities em `initialize` | `tools/list`, `resources/list`, `prompts/list` |
| Camada de transporte | stdio, LSP over TCP | stdio, SSE, HTTP Streamable, WebSocket |
| Comunicação bidirecional | Suportada | Suportada (notifications, elicitation) |
| Negociação de versão | Suportada | Suportada (`protocolVersion`) |

LSP resolve o problema de explosão M×N de "cada editor precisa integrar cada linguagem"; MCP resolve o mesmo problema de "cada ferramenta de IA precisa integrar cada serviço externo".

### Princípios de Design do Protocolo Cliente-Servidor

Duas escolhas de design chave do MCP têm profundo impacto na implementação do Claude Code:

**Negociação de Capacidades (Capability Negotiation)**: O servidor declara o subconjunto de funcionalidades suportadas (`tools`, `prompts`, `resources`, `elicitation`, `experimental`) via `ServerCapabilities` na conexão; o cliente só chama funcionalidades que o servidor declarou. Isso significa que o Claude Code não precisa escrever branches especiais para cada tipo de servidor — decide o comportamento uniformemente verificando `capabilities`.

**Anotações de Ferramentas (Tool Annotations)**: A versão MCP de março de 2025 introduziu o campo `tool.annotations`, permitindo que servidores declarem marcadores semânticos como `readOnlyHint`, `destructiveHint`, `openWorldHint`. O Claude Code mapeia diretamente esses marcadores para os métodos `isReadOnly()`, `isDestructive()`, `isOpenWorld()` das ferramentas, podendo tomar decisões de segurança sem manter uma whitelist estática de nomes de ferramentas.

---

## 11.3 Arquitetura do Cliente MCP

### Interface Central da Classe MCPClient

O Claude Code não implementa diretamente o cliente MCP, mas encapsula a classe `Client` fornecida por `@modelcontextprotocol/sdk`. `connectToServer` é a função de entrada central (`client.ts`), usando `lodash/memoize` para caching a nível de conexão, com chave de cache `${name}-${jsonStringify(serverRef)}`:

```typescript
// client.ts (aproximadamente linha 540)
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers: number; ... },
  ): Promise<MCPServerConnection> => {
    // ... inicializa transport com base em serverRef.type
    const client = new Client(
      { name: 'claude-code', title: 'Claude Code', version: MACRO.VERSION ?? 'unknown' },
      { capabilities: { roots: {}, elicitation: {} } },
    )
    // ... conexão, timeout, negociação de capacidades
  },
  getServerCacheKey,
)
```

### Gerenciamento de Conexão (Estabelecimento, Manutenção, Encerramento)

**Estabelecimento de conexão**: `connectToServer` cria o transport correspondente com base em `serverRef.type`, então inicia `client.connect(transport)`, definindo um timeout de 30 segundos (`getConnectionTimeoutMs()`, substituível via variável de ambiente `MCP_TIMEOUT`):

```typescript
// client.ts (aproximadamente linha 1000)
const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    transport.close().catch(() => {})
    reject(new TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS(
      `MCP server "${name}" connection timed out after ${getConnectionTimeoutMs()}ms`,
      'MCP connection timeout',
    ))
  }, getConnectionTimeoutMs())
  connectPromise.then(() => clearTimeout(timeoutId), () => clearTimeout(timeoutId))
})
await Promise.race([connectPromise, timeoutPromise])
```

**Manutenção da conexão**: implementa detecção de erros e reconexão automática sobrescrevendo `client.onerror` e `client.onclose`. Para transports remotos (SSE/HTTP), mantém um contador `consecutiveConnectionErrors`; após 3 erros de terminal consecutivos (`ECONNRESET`/`ETIMEDOUT`/`EPIPE`, etc.) aciona `closeTransportAndRejectPending`, que chama `client.close()` rejeitando todos os `callTool()` pendentes e limpa o cache memoize, reconectando automaticamente na próxima requisição:

```typescript
// client.ts (aproximadamente linha 1250)
client.onclose = () => {
  // limpa todos os caches relacionados; reconexão acionada na próxima chamada
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
  connectToServer.cache.delete(key)
}
```

**Tratamento de expiração de sessão**: servidores MCP com transport HTTP podem retornar HTTP 404 + código de erro JSON-RPC `-32001` (Session Not Found). O Claude Code detecta esse padrão específico de erro, aciona reconexão e realiza nova tentativa transparente em `fetchToolsForClient.call()` (máximo 1 vez):

```typescript
// client.ts (aproximadamente linha 150)
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error ? (error as Error & { code?: number }).code : undefined
  if (httpStatus !== 404) return false
  return error.message.includes('"code":-32001') || error.message.includes('"code": -32001')
}
```

**Encerramento da conexão**: transports stdio usam escalada de sinal em três fases: primeiro `SIGINT` (espera 100ms), depois `SIGTERM` (espera 400ms), finalmente `SIGKILL`; tempo total máximo de desconexão 600ms, evitando bloquear a saída do CLI.

### Descoberta e Registro Dinâmico de Ferramentas

`fetchToolsForClient` (com cache LRU de capacidade 20) envia `tools/list` ao servidor, encapsulando cada ferramenta como um objeto Tool conforme a interface interna `Tool`:

- **Regra de nomenclatura**: `mcp__${normalizeNameForMCP(serverName)}__${toolName}` (formato concatenado com underscores)
- **Truncamento de descrição**: descriptions com mais de `MAX_MCP_DESCRIPTION_LENGTH = 2048` caracteres são truncadas e acrescidas de `… [truncated]`, evitando que documentação excessivamente longa de servidores que geram OpenAPI polua o contexto
- **Mapeamento de permissões**: `tool.annotations.readOnlyHint` → `isReadOnly()`, `tool.annotations.destructiveHint` → `isDestructive()`
- **Classificação para colapso**: chama `classifyMcpToolForCollapse(serverName, toolName)` para determinar se é uma ferramenta do tipo Search/Read

Da mesma forma, `fetchCommandsForClient` envia `prompts/list`, convertendo Prompts MCP em objetos `/comando`; `fetchResourcesForClient` envia `resources/list`, injetando ferramentas `ListMcpResourcesTool` e `ReadMcpResourceTool` para servidores que suportam recursos.

### Camada de Transporte de Mensagens

O Claude Code suporta 6 tipos de transporte:

| Tipo | Cenário de Uso | Classe Transport |
|------|----------------|-----------------|
| `stdio` | Subprocesso local (maioria dos servidores da comunidade) | `StdioClientTransport` |
| `sse` | Servidor SSE remoto (com OAuth) | `SSEClientTransport` |
| `sse-ide` | SSE interno de extensão IDE (sem OAuth) | `SSEClientTransport` (configuração simplificada) |
| `http` | MCP Streamable HTTP (especificação mais recente) | `StreamableHTTPClientTransport` |
| `ws` | Transport WebSocket | `WebSocketTransport` personalizado |
| `ws-ide` | WebSocket interno de extensão IDE | `WebSocketTransport` (com `X-Claude-Code-Ide-Authorization`) |

Em cenários especiais, servidores MCP de Chrome Extension e de Computer Use rodam **em modo in-process**, criando pipeline de memória via `createLinkedTransportPair()`, evitando o overhead de ~325 MB de subprocesso.

O transport HTTP tem um detalhe de engenharia importante: cada requisição POST precisa carregar o header `Accept: application/json, text/event-stream` (exigido pela especificação MCP Streamable HTTP); o Claude Code injeta esse header uniformemente via `wrapFetchWithTimeout`, prevenindo que alguns ambientes de runtime o percam:

```typescript
// client.ts (aproximadamente linha 460)
const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
// em wrapFetchWithTimeout:
if (!headers.has('accept')) {
  headers.set('accept', MCP_STREAMABLE_HTTP_ACCEPT)
}
```

---

## 11.4 Gerenciamento de Configuração MCP

### Formato de Configuração de Servidor

`types.ts` usa Zod para definir 7 Schemas de configuração de servidor, agregados como `McpServerConfigSchema` via `z.union([...])`:

```typescript
// types.ts (linhas 28-115, resumo)
export const McpStdioServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('stdio').optional(), // compatibilidade retroativa: sem campo type = stdio
  command: z.string().min(1),
  args: z.array(z.string()).default([]),
  env: z.record(z.string(), z.string()).optional(),
}))

export const McpSSEServerConfigSchema = lazySchema(() => z.object({
  type: z.literal('sse'),
  url: z.string(),
  headers: z.record(z.string(), z.string()).optional(),
  oauth: McpOAuthConfigSchema().optional(),
}))
// + HTTP / WebSocket / SDK / claudeai-proxy ...
```

`ScopedMcpServerConfig` adiciona os campos `scope` (origem da configuração) e `pluginSource` (identificador da fonte que fornece o plugin) à configuração base, para uso na verificação de permissões de Channel.

### Fusão de Configurações de Múltiplas Fontes (enterprise > local > project > user > dynamic)

`getClaudeCodeMcpConfigs` (`config.ts`) implementa fusão de múltiplas camadas de configuração, com prioridade decrescente:

1. **enterprise** (`managed-mcp.json`): modo exclusivo empresarial; quando este arquivo existe, bloqueia todas as outras fontes
2. **local** (privado do projeto, armazenado na configuração global do usuário, vinculado ao CWD)
3. **project** (`.mcp.json`, percorre a árvore de diretórios para cima, mais próximo tem prioridade)
4. **user** (campo `mcpServers` no global `~/.claude/config.json`)
5. **dynamic** (injeção em runtime via parâmetro CLI `--mcp-config`)

A configuração de projeto requer uma **porta de aprovação do usuário** adicional: ao encontrar pela primeira vez um servidor em `.mcp.json`, um diálogo de aprovação é exibido. `getProjectMcpServerStatus()` lê as configurações `enabledMcpjsonServers`/`disabledMcpjsonServers`, retornando `approved`/`rejected`/`pending`. No modo não interativo (parâmetro `-p`, chamada SDK) com `isSettingSourceEnabled('projectSettings')`, a aprovação é automática.

Após a fusão de configurações, também é executada **deduplicação**: servidores plugin são deduplicados por "assinatura" (para servidores stdio usam array de comandos, para servidores remotos usam URL), evitando que o mesmo servidor subjacente seja conectado duas vezes; Connectors do claude.ai também evitam duplicação com configuração manual através do mesmo mecanismo.

### Expansão de Variáveis de Ambiente

Arquivos de configuração podem usar a sintaxe `${ENV_VAR}`; `expandEnvVarsInString` (`config.ts`/`envExpansion.ts`) expande ao ler a configuração. Variáveis indefinidas são coletadas em uma lista `missingVars` e reportadas ao usuário.

---

## 11.5 Sistema de Autenticação MCP

### Integração OAuth 2.0

`ClaudeAuthProvider` (`auth.ts`) implementa a interface `OAuthClientProvider` do SDK MCP, gerenciando o ciclo de vida completo do OAuth. O fluxo de autenticação segue o fluxo de código de autorização RFC 6749 + PKCE (Proof Key for Code Exchange), recebendo callback via servidor HTTP local:

1. **Descoberta de metadados**: primeiro testa RFC 9728 (`/.well-known/oauth-protected-resource`), com fallback para RFC 8414 (`/.well-known/oauth-authorization-server`), e por último tenta descoberta ciente do caminho (para manter compatibilidade retroativa)
2. **DCR (Registro Dinâmico de Cliente)**: registra automaticamente o cliente OAuth na primeira autenticação; `clientId`/`clientSecret` são armazenados no Keychain do sistema
3. **Troca de Token**: porta local aleatória recebe o código de autorização e troca por access_token + refresh_token
4. **Renovação de Token**: verifica expiração e renova antes de cada chamada via `checkAndRefreshOAuthTokenIfNeeded()`; em caso de falha, realiza nova tentativa inteligente

**Camada de compatibilidade Slack**: alguns servidores OAuth (notavelmente o Slack) retornam HTTP 200 no endpoint de token, mas com corpo de erro, violando a expectativa do RFC 6749. O Claude Code reescreve essas respostas para HTTP 400 via `normalizeOAuthErrorBody`, fazendo com que a lógica de classificação de erros do SDK funcione corretamente:

```typescript
// auth.ts (aproximadamente linha 250)
export async function normalizeOAuthErrorBody(response: Response): Promise<Response> {
  if (!response.ok) return response
  // detecta OAuthErrorResponse disfarçado como 200
  const result = OAuthErrorResponseSchema.safeParse(parsed)
  if (!result.success) return new Response(text, response)
  // normaliza código de erro não padrão do Slack 'invalid_refresh_token' para 'invalid_grant'
  const normalized = NONSTANDARD_INVALID_GRANT_ALIASES.has(result.data.error)
    ? { error: 'invalid_grant', error_description: ... }
    : result.data
  return new Response(jsonStringify(normalized), { status: 400, ... })
}
```

### Suporte a Múltiplos Métodos de Autenticação

Além do OAuth padrão, o Claude Code também suporta:

- **Step-Up Auth**: algumas operações requerem escopo de permissão elevado; o servidor retorna HTTP 403 com novos requisitos de escopo; o Claude Code detecta isso e reaciona o fluxo OAuth
- **XAA (Cross-App Access / SEP-990)**: em cenários empresariais, faz login uma vez via IdP unificado (suportando OIDC) para autorizar múltiplos servidores MCP, usando o fluxo composto RFC 8693 (Token Exchange) + RFC 7523 (JWT Bearer), sem necessidade de abrir janela de browser separada para cada servidor
- **Static Headers**: injeta headers de autenticação estáticos via arquivo de configuração ou script `headersHelper` (adequado para autenticação por API Key)

### Gerenciamento de Token

Os dados do token são armazenados no armazenamento seguro do sistema (macOS Keychain / Linux Secret Service), com chave `${serverName}|${SHA256(config)[:16]}`, garantindo que servidores com o mesmo nome mas configurações diferentes usem slots de token independentes.

`auth-cache` (`mcp-needs-auth-cache.json`) registra servidores que retornaram 401 recentemente, com TTL de 15 minutos, evitando testar repetidamente servidores que inevitavelmente falharão a cada inicialização. A leitura do cache é compartilhada via Promise (`authCachePromise`), prevenindo N leituras concorrentes do mesmo arquivo em conexões em lote.

---

## 11.6 Execução de Ferramentas MCP

### Fluxo de Execução do MCPTool

Quando o LLM decide chamar `mcp__slack__send_message`, o fluxo de execução é:

1. **Roteamento**: a função `call()` registrada por `fetchToolsForClient` é chamada, com o input JSON gerado pelo LLM como parâmetro
2. **Verificação de reconexão**: `ensureConnectedClient(client)` verifica se a conexão ainda é válida, reconectando se necessário
3. **Notificação de progresso**: emite evento `mcp_progress: started` via callback `onProgress`
4. **Chamada de ferramenta**: `callMCPToolWithUrlElicitationRetry` (que encapsula `callMCPTool`) envia requisição `tools/call` ao servidor
5. **Processamento de resultado**: tratamento especial para imagens e conteúdo binário grande (persistindo em disco, passando referência); truncamento de conteúdo de texto excessivamente grande
6. **Notificação de progresso**: emite evento `mcp_progress: completed` (incluindo duração)

Lógica de nova tentativa transparente em caso de expiração de sessão:

```typescript
// client.ts (aproximadamente linha 2100)
const MAX_SESSION_RETRIES = 1
for (let attempt = 0; ; attempt++) {
  try {
    const connectedClient = await ensureConnectedClient(client)
    const mcpResult = await callMCPToolWithUrlElicitationRetry({ ... })
    return { data: mcpResult.content, ... }
  } catch (error) {
    if (error instanceof McpSessionExpiredError && attempt < MAX_SESSION_RETRIES) {
      continue // nova tentativa automática uma vez
    }
    throw error
  }
}
```

### classifyForCollapse — Classificação para Colapso de Contexto de Resultados de Ferramentas

`classifyForCollapse.ts` mantém dois conjuntos estáticos Set: `SEARCH_TOOLS` (aproximadamente 100 nomes de ferramentas) e `READ_TOOLS` (aproximadamente 300 nomes de ferramentas), cobrindo 40+ servidores MCP populares (Slack, GitHub, Linear, Datadog, Sentry, Jira, Asana, Gmail, Grafana, PagerDuty, etc.).

Regras de classificação: o nome da ferramenta é primeiro normalizado por `normalize()` (conversão uniforme de camelCase/kebab-case para snake_case), depois verifica se está em algum dos dois Sets:

```typescript
// classifyForCollapse.ts (linhas 587-598)
function normalize(name: string): string {
  return name
    .replace(/([a-z])([A-Z])/g, '$1_$2')
    .replace(/-/g, '_')
    .toLowerCase()
}

export function classifyMcpToolForCollapse(
  _serverName: string,
  toolName: string,
): { isSearch: boolean; isRead: boolean } {
  const normalized = normalize(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
  }
}
```

**Intenção de design**: os resultados de ferramentas do tipo Search/Read geralmente são longos, mas têm valor limitado para o raciocínio subsequente do LLM (estado intermediário de recuperação). Após marcação, a camada UI pode colapsar esses resultados no histórico do diálogo, economizando espaço visual e janela de contexto. Observe que a classificação é **conservadora** (ferramentas desconhecidas não são colapsadas) e **baseada apenas no nome da ferramenta**, sem distinguir o nome do servidor, pois os nomes de ferramentas dos servidores mais populares são identificadores estáveis entre instâncias.

### Controle de Permissões e Sandbox

Antes da execução de ferramentas MCP, `checkPermissions()` é chamado, retornando o estado `passthrough` (ou seja, sempre exibindo o prompt de permissão); o prompt inclui ação de atalho sugerindo ao usuário adicionar o nome da ferramenta à lista de regras `allow`.

O timeout de chamada de ferramenta é controlado pela variável de ambiente `MCP_TOOL_TIMEOUT`, com padrão `100_000_000` milissegundos (aproximadamente 27,8 horas, próximo a "infinito"), permitindo que servidores MCP com operações demoradas completem normalmente.

---

## 11.7 Sistema de Channel MCP

O sistema Channel é um uso estendido do MCP: permitindo que plataformas de mensagens externas (Telegram, Discord, iMessage, Slack, etc.) enviem mensagens para sessões Claude Code em andamento (feature flag: `KAIROS`/`KAIROS_CHANNELS`, gate de runtime: `tengu_harbor`).

### Gerenciamento de Permissões de Channel

`channelPermissions.ts` implementa um mecanismo de **delegação de permissões**: quando o Claude Code encontra uma operação que requer aprovação do usuário, pode simultaneamente enviar um prompt ao celular do usuário através do servidor Channel; o usuário responde com `yes <ID de 5 letras>`, o servidor analisa e notifica o Claude Code para aprovação via evento `notifications/claude/channel/permission`.

O ID de 5 letras usa um alfabeto de 25 caracteres (sem `l` para evitar confusão com `1`/`I`), gerado via hash FNV-1a, com filtragem de palavras inadequadas (lista `ID_AVOID_SUBSTRINGS`, aproximadamente 24 palavras), garantindo que não apareçam conteúdos impróprios em mensagens de trabalho:

```typescript
// channelPermissions.ts (linhas 86-110)
export function shortRequestId(toolUseID: string): string {
  let candidate = hashToId(toolUseID)
  for (let salt = 0; salt < 10; salt++) {
    if (!ID_AVOID_SUBSTRINGS.some(bad => candidate.includes(bad))) {
      return candidate
    }
    candidate = hashToId(`${toolUseID}:${salt}`)
  }
  return candidate
}
```

O servidor Channel deve declarar simultaneamente as capacidades `capabilities.experimental['claude/channel']` e `capabilities.experimental['claude/channel/permission']` para se tornar um relay de permissões, evitando a abertura acidental de limites de segurança.

### Mecanismo de Notificação de Channel

`channelNotification.ts` define a lógica completa de controle para receber mensagens de entrada (`gateChannelServer`), verificando em sequência:

1. Declaração de capacidades do servidor (`claude/channel`)
2. Interruptor de runtime (`tengu_harbor`)
3. Autenticação OAuth (suporta apenas login com conta claude.ai, não suporta API Key)
4. Política de equipe/empresa (`channelsEnabled: true`)
5. Parâmetro `--channels` da sessão (canais declarados explicitamente como confiáveis pelo usuário)
6. Verificação de origem do Marketplace (evita que `slack@evil` imite `slack@anthropic`)

Mensagens de entrada são encapsuladas no formato `<channel source="serverName" meta_key="value">content</channel>` e injetadas na fila da sessão; após o polling do `SleepTool` (intervalo de aproximadamente 1 segundo), o modelo decide como responder.

### Tratamento de Elicitation

`elicitationHandler.ts` trata requisições de interação iniciadas pelo servidor (especificação MCP Elicitation). Suporta dois modos:

- **Modo form**: o servidor solicita que o usuário preencha um formulário (campo `requestedSchema` define JSON Schema)
- **Modo url**: o servidor solicita que o usuário acesse uma URL para completar uma operação (como autorização OAuth)

Fluxo de processamento: primeiro executa o sistema de Hook (resposta programática); se o Hook não responder, enfileira a requisição em `AppState.elicitation.queue`, aguardando a UI renderizar o formulário ou abrir o browser; após a ação do usuário, o callback `respond()` aciona a resposta:

```typescript
// elicitationHandler.ts (linhas 69-90)
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. tenta primeiro o Hook (resposta programática)
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. exibe UI, aguarda resposta do usuário
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: { queue: [...prev.elicitation.queue, { respond: resolve, ... }] },
    }))
  })
  return await response
})
```

O modo url também suporta `ElicitationCompleteNotificationSchema`: após o servidor concluir a operação, notifica o Claude Code ativamente; o item correspondente na fila é marcado como `completed: true`, e a UI atualiza o status de exibição de acordo.
