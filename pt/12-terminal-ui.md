# Capítulo 12: UI de Terminal e Motor de Renderização

## 12.1 Visão Geral e Posicionamento

O Claude Code é um assistente interativo de codificação por IA que roda no terminal; sua camada de UI precisa alcançar uma experiência próxima à GUI em um ambiente de terminal de texto puro: renderização de mensagens em streaming, edição com atalhos Vim, seleção de texto com mouse, destaque de busca, sobreposição de múltiplos painéis. Esses requisitos estão muito além das capacidades de ferramentas CLI tradicionais.

A camada de UI de terminal do Claude Code é construída sobre três decisões técnicas centrais:

1. **Renderizador Ink Personalizado** (`src/ink/`): reforma profunda do framework React + Ink, introduzindo modelo de tela com double buffering, motor de diff ANSI, aceleração de rolagem por hardware e posicionamento físico de cursor ciente de IME.
2. **Árvore de Componentes Declarativa** (`src/screens/REPL.tsx`, `src/components/`): usa componentes React para descrever a estrutura da UI; o motor de renderização é responsável por traduzir o DOM virtual para uma matriz de caracteres de terminal.
3. **Modo Vim Completo** (`src/vim/`): implementação de máquina de estados independente, cobrindo o conjunto completo de modos NORMAL/INSERT, operators/motions/textObjects, além de dot-repeat e registradores.

O ritmo de todo o pipeline de renderização é controlado por `FRAME_INTERVAL_MS = 16` (`src/ink/constants.ts:1`), taxa de frames teórica de 62,5fps, alinhada com a frequência de atualização do monitor.

---

## 12.2 Bases Teóricas

### Princípios Fundamentais da Renderização de Terminal

A renderização de terminal é essencialmente escrever um fluxo de bytes em um file descriptor; o emulador de terminal é responsável por fazer o parse e atualizar a tela. Os mecanismos-chave incluem:

- **Códigos de escape ANSI**: sequências de controle começando com `\x1b[`, controlando posição do cursor (CSI CUP), cor (SGR), área de rolagem (DECSTBM), rastreamento de mouse (modos privados DEC), etc. O diretório `src/ink/termio/` do Claude Code encapsula essas sequências como constantes nomeadas (`CURSOR_HOME`, `ENTER_ALT_SCREEN`, `ENABLE_MOUSE_TRACKING`, etc.).
- **termios / modo raw**: após entrar no modo raw, cada tecla pressionada chega imediatamente ao programa, sem passar por buffer de linha ou eco. O Claude Code assume o controle do stdin via `setRawMode(true)` na inicialização, restaurando-o ao sair via `Ink.detachForShutdown()` (`ink.tsx:942-950`).
- **Alt Screen (?1049h/?1049l)**: buffer de tela alternativo; ao entrar não polui o histórico de rolagem da tela principal; ao sair restaura o estado original. No modo tela cheia (`isFullscreenEnvEnabled()`), o REPL é envolto no componente `<AlternateScreen>`.

### Renderização com Double Buffering

O Claude Code usa dois buffers de frame (front e back) para implementar double buffering (`ink.tsx:196-197`):

```typescript
// ink.tsx:196-197
this.frontFrame = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
this.backFrame  = emptyFrame(this.terminalRows, this.terminalColumns, this.stylePool, this.charPool, this.hyperlinkPool);
```

A cada frame de renderização:
1. O Renderer escreve o DOM virtual React no `backFrame` (novo frame)
2. O motor de diff LogUpdate compara `frontFrame` (frame antigo) com `backFrame`, gerando a sequência mínima de patches
3. A sequência de patches é escrita no terminal; `frontFrame` ↔ `backFrame` são trocados (`ink.tsx:594`)

Assim, o terminal recebe apenas os caracteres que realmente mudaram, em vez de redesenhar o frame inteiro a cada vez.

### Aplicação de React em Ambiente não-DOM (Custom Renderer)

A interface de renderizador do React (`react-reconciler`) permite qualquer ambiente hospedeiro arbitrário. O Claude Code implementa um Custom Renderer para terminal em `src/ink/reconciler.ts` (512 linhas):

- O tipo de elemento hospedeiro é o `dom.DOMElement` personalizado (`src/ink/dom.ts`)
- O motor de layout usa Yoga (implementação do Flexbox do Facebook), acionado no callback `onComputeLayout` (`ink.tsx:239-258`)
- O modo React Concurrent Root (`ConcurrentRoot`) permite renderização interruptível, evitando bloqueio prolongado da thread principal

---

## 12.3 Renderizador Ink Personalizado

### Personalizações do Claude Code ao Ink

`src/ink/ink.tsx` (1.722 linhas) é o controlador central de toda a camada UI, definindo a classe `Ink`. Não se trata do pacote Ink original diretamente, mas de uma versão profundamente modificada; os pontos centrais de aprimoramento incluem:

**Double buffering + gerenciamento de frames front/back**: o Ink original gera strings diretamente; a versão personalizada introduz objetos Frame (`Frame`) e a matriz `Screen`.

**Monitoramento de performance**: cada frame registra a duração de cada fase — renderer, diff, optimize, write — além das métricas de layout Yoga (visited/measured/cacheHits), expostas via callback `onFrame` (`ink.tsx:772-788`):

```typescript
// ink.tsx:772-788
this.options.onFrame?.({
  durationMs: performance.now() - renderStart,
  phases: {
    renderer: rendererMs,
    diff: diffMs,
    optimize: optimizeMs,
    write: writeMs,
    patches: diff.length,
    yoga: yogaMs,
    commit: commitMs,
    yogaVisited: yc.visited,
    yogaMeasured: yc.measured,
    yogaCacheHits: yc.cacheHits,
    yogaLive: yc.live,
  },
  flickers,
});
```

**Transferência para TUI externo**: fornece métodos `enterAlternateScreen()` / `exitAlternateScreen()`, pausando o controle do Ink, fechando o protocolo kitty keyboard, permitindo que editores externos (vim, nano) funcionem normalmente, restaurando completamente ao retornar (`ink.tsx:357-419`).

**Auto-recuperação SIGCONT**: `handleResume` trata o retorno do processo de pausa (comando fg); na tela principal reinicia o buffer; na tela alternativa re-entra e limpa a tela (`ink.tsx:280-301`).

### Mecanismo de Double Buffering + Atualização por Diff

`LogUpdate.render()` em `src/ink/log-update.ts` é o núcleo do motor de diff. Recebe `prev: Frame` e `next: Frame`, retornando um conjunto de operações de patch `Diff`:

```
Tipos de patch Diff:
- { type: 'stdout', content: string }    — escrita direta no terminal
- { type: 'cursorMove', x, y }           — movimento relativo do cursor
- { type: 'cursorTo', x, y }             — posicionamento absoluto do cursor
- { type: 'styleStr', str }              — troca de estilo SGR
- { type: 'hyperlink', uri }             — hiperlink OSC 8
- { type: 'cursorHide' | 'cursorShow' }  — visibilidade do cursor
- { type: 'clear', count }               — apagar caracteres
- { type: 'clearTerminal', reason }      — redesenho completo (provoca flicker)
```

O motor de diff é otimizado em `src/ink/optimizer.ts` com um único pass (`optimize(diff)`), mesclando `cursorMove` adjacentes, colapsando `cursorTo` consecutivos, concatenando `styleStr` adjacentes, cancelando pares cursor hide/show, reduzindo a quantidade de escrita real.

### Buffer de Tela Int32Array

`src/ink/screen.ts` define o tipo `Screen`, usando `Int32Array` em vez de arrays de objetos para armazenar células da tela, eliminando pressão de GC (`screen.ts:356-370`):

```typescript
// screen.ts:356-370 (comentário)
// Screen uses a packed Int32Array instead of Cell objects to eliminate GC
// pressure. For a 200x120 screen, this avoids allocating 24,000 objects.
//
// Cell data is stored as 2 Int32s per cell in a single contiguous array:
//   word0: charId (full 32 bits — index into CharPool)
//   word1: styleId[31:17] | hyperlinkId[16:2] | width[1:0]
```

`createScreen` aloca um `ArrayBuffer`, montando simultaneamente duas views (`Int32Array` e `BigInt64Array`); a segunda é usada para limpeza em lote via `resetScreen` (`screen.ts:472-490`):

```typescript
// screen.ts:472-490
const buf = new ArrayBuffer(size << 3) // 8 bytes por célula
const cells = new Int32Array(buf)
const cells64 = new BigInt64Array(buf)
```

Caracteres e estilos são internalizados via **objetos Pool compartilhados** (`CharPool`, `StylePool`, `HyperlinkPool`), compartilhando IDs entre Screens, permitindo que `blitRegion` copie diretamente IDs inteiros sem re-internalizar e que `diffEach` use comparações de inteiros em vez de comparações de strings (`screen.ts:17-31`).

O bit-0 do `StylePool.intern()` codifica visibilidade (`screen.ts:148-161`): IDs ímpares indicam que o estilo é visível em espaços (cor de fundo, cor invertida, sublinhado), permitindo ao renderizador pular espaços invisíveis com uma única máscara de bit, evitando escritas desnecessárias.

### Rolagem por Hardware (DECSTBM)

`log-update.ts` implementa a otimização de rolagem por hardware DECSTBM (`log-update.ts:149-185`):

Quando no modo alt-screen o `scrollTop` de `ScrollBox` muda, em vez de reescrever toda a área de rolagem, envia `CSI top;bot r` (define área de rolagem) + `CSI n S` (rola n linhas para cima); o hardware do terminal executa o deslocamento de conteúdo de forma acelerada, sendo necessário apenas preencher as novas linhas. Simultaneamente aplica `shiftRows()` no `prev.screen` para simular a rolagem, de modo que o loop de diff subsequente encontre naturalmente as linhas com diferenças em vez de reescrever tudo.

Essa otimização só é habilitada quando `decstbmSafe=true` — quando o terminal externo não suporta DEC 2026 (operações atômicas BSU/ESU), terminais como tmux renderizariam o estado intermediário (após rolagem, antes de completar a nova linha), causando saltos visíveis; nesse caso, retorna ao diff completo (`log-update.ts:158-167`).

### Cache de 16.384 Caracteres

A classe `Output` em `src/ink/output.ts` (o motor interno de medição e saída de caracteres do renderizador) mantém um `charCache`, cacheando o mapeamento de caractere para largura medida. Quando o cache excede 16.384 entradas, é limpo e reconstruído completamente (`output.ts:204`):

```typescript
// output.ts:204
if (this.charCache.size > 16384) this.charCache.clear()
```

Da mesma forma, `src/ink/line-width-cache.ts` mantém um cache de largura de linha (máximo 4.096 entradas), para reutilizar chamadas `stringWidth` de linhas inalteradas em grande quantidade durante streaming (`line-width-cache.ts`):

```typescript
// line-width-cache.ts
// During streaming, text grows but completed lines are immutable.
// Caching stringWidth per-line avoids re-measuring hundreds of
// unchanged lines on every token (~50x reduction in stringWidth calls).
const MAX_CACHE_SIZE = 4096
```

### Estratégias de Otimização de Performance

| Estratégia | Localização | Efeito |
|-----------|-------------|--------|
| Atraso de renderização por microtask | `ink.tsx:212-216` | scheduleRender usa `queueMicrotask`, garantindo que useLayoutEffect complete antes da renderização, evitando atraso de um frame na posição do cursor |
| throttle 60fps | `ink.tsx:213` | `throttle(deferredRender, 16, {leading:true, trailing:true})`, estabilizando a taxa de frames |
| Timer de drain quatro vezes por frame | `ink.tsx:757-758` | drain contínuo de rolagem usa `FRAME_INTERVAL_MS >> 2` (aproximadamente 4ms), maximizando a fluidez da rolagem |
| Reset periódico do Pool | `ink.tsx:600-603` | Resetar pools de char/hyperlink a cada 5 minutos, evitando crescimento de memória em sessões longas |
| Retângulo de dano (damage rectangle) | `screen.ts:382` | A cada frame, faz diff apenas na região retangular realmente escrita, não em toda a tela |
| Ancoragem de cursor na tela alternativa | `ink.tsx:578-591` | Antes do diff a cada frame, usa `ALT_SCREEN_ANCHOR_CURSOR` para resetar o cursor virtual para (0,0), evitando deriva do cursor em terminais como tmux |

---

## 12.4 Interface Principal REPL

### Estrutura de Componentes do REPL.tsx

`src/screens/REPL.tsx` (5.005 linhas) é o componente de interface principal do Claude Code e também o maior arquivo único de todo o projeto. Suas importações de nível superior excedem 150 linhas, cobrindo gerenciamento de sessão, sistema de permissões, conexões MCP, Swarm multiagente, integração de voz e quase todos os outros módulos funcionais.

A estrutura de renderização do REPL (simplificada):

```
KeybindingSetup
└── MCPConnectionManager
    ├── AlternateScreen (modo tela cheia)
    │   └── Box (layout principal)
    │       ├── AnimatedTerminalTitle (título da aba do terminal)
    │       ├── FullscreenLayout
    │       │   ├── Messages (lista de mensagens)
    │       │   ├── VirtualMessageList (rolagem virtual)
    │       │   └── TranscriptModeFooter / TranscriptSearchBar
    │       ├── PermissionRequest / ElicitationDialog / PromptDialog
    │       ├── vários Callouts / Survey / Dialog
    │       └── PromptInput (área de entrada)
    └── DevBar (barra de debug ant-only)
```

O modo tela cheia decide via branch condicional se encapsula em `<AlternateScreen>` (`REPL.tsx:4998-5001`):

```typescript
// REPL.tsx:4998-5001
if (isFullscreenEnvEnabled()) {
  return <AlternateScreen mouseTracking={isMouseTrackingEnabled()}>
      {mainReturn}
    </AlternateScreen>;
}
return mainReturn;
```

### Fluxo de Renderização de Mensagens

O estado de mensagens é mantido por `useState<MessageType[]>` do React; `handleMessageFromStream` processa eventos SSE em streaming para adicionar/atualizar mensagens. A lista de mensagens é renderizada no componente `<Messages>` ou `<VirtualMessageList>`; o último usa rolagem virtual para renderizar apenas a área visível, evitando um grande número de nós DOM em diálogos longos.

O REPL contém múltiplas otimizações para evitar re-renderizações desnecessárias: `AnimatedTerminalTitle` é extraído separadamente como nó folha, evitando que o tick de animação de 960ms re-renderize a árvore REPL inteira (comentário em `REPL.tsx` explica: `Before extraction, the tick was ~1 REPL render/sec for the duration of every turn, dragging PromptInput and friends along`).

`useDeferredValue(searchQuery)` é usado para adiar cálculos de busca; `React.memo` cacheia na subárvore `PromptInput`, evitando que re-renderizações frequentes ao digitar caracteres se propaguem para cima.

### Pipeline de Processamento de Entrada

A entrada do usuário entra no `parse-keypress.ts` do Ink via stdin em modo raw, é convertida em objetos `ParsedKey`, e distribuída para callbacks `useInput` do nó com foco atual via FocusManager. `GlobalKeybindingHandlers` e `CommandKeybindingHandlers` fornecem mecanismo de registro de atalhos globais, com prioridade maior que o `useInput` a nível de componente.

A entrada eventualmente chega ao componente `PromptInput`, que roteia para `VimTextInput` ou `TextInput` com base no modo atual (Vim/Standard).

---

## 12.5 Modo Vim

### Arquitetura de Implementação do Modo Vim

O diretório `src/vim/` (5 arquivos, ~1.513 linhas) implementa uma camada completa de atalhos Vim. A arquitetura é extremamente concisa: **tipos são documentação** — todas as transições de máquina de estados são codificadas em tipos TypeScript; a verificação exaustiva do TypeScript garante completude no tratamento de estados.

Divisão de responsabilidades dos arquivos:
- `types.ts`: definições de tipos de máquina de estados + conjuntos de constantes
- `transitions.ts`: funções de transição (entrada principal `transition()`)
- `motions.ts`: funções puras de movimento de cursor (`resolveMotion()`)
- `operators.ts`: funções de execução de operadores (delete/change/yank, etc.)
- `textObjects.ts`: cálculo de limites de objetos de texto

### Operações Suportadas (motions, operators, textObjects)

**Operators**: `d` (delete), `c` (change), `y` (yank) — definidos em `types.ts:OPERATORS`.

**Simple Motions** (`types.ts:SIMPLE_MOTIONS`):
- Movimentos básicos: `h`/`l`/`j`/`k`
- Movimentos de palavra: `w`/`b`/`e`/`W`/`B`/`E`
- Posições de linha: `0`/`^`/`$`

**Find Motions**: `f`/`F`/`t`/`T` (busca de caractere); `;`/`,` para repetir/reverter.

**Text Objects** (`types.ts:TEXT_OBJ_TYPES`): `w`/`W` (palavra), `"`/`'`/`` ` `` (aspas), `(`/`)`/`b` (parênteses), `[`/`]` (colchetes), `{`/`}`/`B` (chaves), `<`/`>` (ângulos).

**Operações de linha**: `dd`/`cc`/`yy`, `D`/`C`/`Y`, `o`/`O` (abrir linha), `J` (juntar), `>>`/`<<` (indentação).

**Outros comandos**: `r` (substituir char), `~` (alternar maiúsculas/minúsculas), `x` (deletar char), `p`/`P` (colar), `u` (desfazer), `G`/`gg` (pular para linha), `.` (dot-repeat).

Tratamento especial: `cw`/`cW` muda até o final da palavra em vez do início da próxima (comportamento clássico do vim, `operators.ts:91-102`); chips de referência de imagem são reconhecidos como unidades atômicas; movimentos de palavra não param no meio de um chip (`operators.ts:333-337`).

### Máquina de Estados e Troca de Modos

`VimState` é o tipo de nível superior (`types.ts:52-55`):

```typescript
// types.ts:52-55
export type VimState =
  | { mode: 'INSERT'; insertedText: string }
  | { mode: 'NORMAL'; command: CommandState }
```

O modo INSERT registra o texto inserido para dot-repeat; o modo NORMAL mantém a sub-máquina de estados `CommandState`.

`CommandState` tem 11 estados (`types.ts:62-75`), cobrindo idle, acúmulo de count, operator pending, operatorCount, operatorFind, operatorTextObj, find, g-prefix, operatorG, replace, indent. O diagrama de estados está registrado em ASCII art no início de `types.ts` (`types.ts:1-26`):

```
idle ──┬─[d/c/y]──► operator
       ├─[1-9]────► count
       ├─[fFtT]───► find
       ├─[g]──────► g
       ├─[r]──────► replace
       └─[><]─────► indent
```

A função `transition()` (`transitions.ts:55-73`) despacha para funções de tratamento especializadas com base no tipo `CommandState` atual; cada função retorna `{ next?: CommandState; execute?: () => void }`. Se `execute` existe, roda imediatamente; se `next` existe, avança o estado; se nenhum existir, volta para idle.

`PersistentState` (`types.ts:80-87`) persiste entre comandos: `lastChange` (gravação de dot-repeat), `lastFind` (repetição com `;`/`,`), `register` (área de transferência), `registerIsLinewise` (julgamento de paste em nível de linha).

A multiplicação de count é implementada em `fromOperatorCount` (`transitions.ts`): `effectiveCount = state.count * motionCount`, suportando `3d2w` = deletar 6 palavras. O limite de count é `MAX_VIM_COUNT = 10000` (`types.ts:96`), prevenindo entradas maliciosas.

---

## 12.6 Análise de Componentes Centrais

### Componente de Entrada PromptInput

`src/components/PromptInput/PromptInput.tsx` (2.338 linhas) é um dos componentes com maior densidade funcional do Claude Code, carregando quase toda a lógica de interação relacionada a entrada.

**Escala dos Props**: a definição do tipo `Props` se estende por aproximadamente 60 linhas, contendo valor de entrada, modo, estado Vim, pilha de stash, callbacks de submissão, contexto de permissões, clientes MCP, seleção de IDE, integração de voz, etc.

**Injeção de entrada externa**: `insertTextRef` expõe a interface `{ insert, setInputWithCursor, cursorOffset }`, permitindo que reconhecimento de fala (STT) insira texto na posição do cursor em vez de substituir toda a entrada (`PromptInput.tsx:180-200`).

**Gerenciamento de cursor**: rastreia o estado `cursorOffset` independentemente, distinguindo entre mudanças internas (via `trackAndSetInput` encapsulando `onInputChange`) e injeções externas (entrada de voz); mudanças externas movem o cursor para o final (`PromptInput.tsx:164-176`).

**Roteamento de modo**: renderiza `<VimTextInput>` ou `<TextInput>` com base em `isVimModeEnabled()`; ambos compartilham a interface `BaseTextInputProps`.

**Constantes de layout inferior** (`PromptInput.tsx`):
```typescript
const PROMPT_FOOTER_LINES = 5;
const MIN_INPUT_VIEWPORT_LINES = 3;
```
O slot de entrada inferior tem altura máxima de 50%, reservando 5 linhas para footer, bordas e dicas de status.

### Interface de Configurações

`src/components/Settings/Config.tsx` (1.821 linhas) implementa a interface de configurações do comando `/config`.

**Navegação orientada por busca**: Config entra por padrão no modo busca (`isSearchMode = true`); o usuário pode digitar diretamente para filtrar itens de configuração sem precisar rolar com teclas de direção. `maxVisible = Math.max(5, paneCap - 10)` calcula dinamicamente o número de itens visíveis com base na altura do terminal.

**Categorização de configurações**: o array `settingsItems: Setting[]` define uniformemente todos os itens de configuração; cada item é um dos seguintes: `boolean` (alternância), `enum` (seleção enumerada) ou `managedEnum` (componente personalizado).

**Rastreamento de mudanças e restauração**: o estado `changes` registra todas as modificações; a ref `isDirty` é usada pela tecla Escape para determinar se é necessário escrever em disco ou restaurar, evitando gravações em disco desnecessárias ao abrir e fechar sem mudanças (`Config.tsx`).

**Sistema de submenu**: o estado `showSubmenu` controla a expansão de submenus como Theme, Model, TeammateModel, ExternalIncludes, OutputStyle, Language, EnableAutoUpdates; comunica-se com o componente pai Settings via `setTabsHidden` para ocultar cabeçalhos de Tab.

**Arquitetura de configuração de múltiplas fontes**: as fontes de configuração são divididas em `localSettings` (nível de projeto) e `userSettings` (nível de usuário), lidas e escritas via `getSettingsForSource`/`updateSettingsForSource`; ao restaurar com Escape, usa snapshots iniciais para restaurar por fonte, em vez de ler a visão global mesclada (`Config.tsx`; comentário explica que isso é para suportar a semântica de deleção de chave via `undefined`).

### Seletor de Logs

`src/components/LogSelector.tsx` (1.574 linhas) implementa a interface de seleção de histórico de sessões (comando `/resume`).

**Exibição em árvore**: usa `<TreeSelect<LogTreeNode>>` para exibir sessões bifurcadas (fork) em hierarquia pai-filho; colapso/expansão gerenciado por `expandedGroupSessionIds`.

**Três modos de visualização**: `viewMode` pode ser `"list"` (lista padrão), `"search"` (busca) ou `"preview"` (prévia da sessão).

**Busca fuzzy com Fuse.js**: usa correspondência fuzzy com `FUSE_THRESHOLD = 0.3`, combinado com `DATE_TIE_THRESHOLD_MS = 60 * 1000` (resultados dentro de 1 minuto são ordenados por relevância; caso contrário, por tempo).

**Prévia de trecho de resultado de busca**: `extractSnippet()` pega `SNIPPET_CONTEXT_CHARS = 50` caracteres antes e depois da posição de correspondência, renderizando contexto com `chalk.dim` e destacando a palavra correspondente com cor (`LogSelector.tsx`).

**Constantes de busca profunda**:
```typescript
const DEEP_SEARCH_MAX_MESSAGES = 2000;
const DEEP_SEARCH_MAX_TEXT_LENGTH = 50000;
```
(atualmente `isDeepSearchEnabled = false`, funcionalidade aguardando habilitação)

**Otimização por React Compiler**: o `import { c as _c } from "react/compiler-runtime"` no topo do arquivo indica que foi compilado via React Compiler; `_c(247)` aloca 247 slots de cache memo, com grande quantidade de memorização condicional para evitar re-renderizações de subárvores.

---

## 12.7 Análise das Decisões de Design

### Por que Renderizador Ink Personalizado em vez de Uso Direto

O Ink original serializa a árvore React em string e a escreve de forma incremental com `log-update`, projetado para ferramentas CLI simples. Os requisitos do Claude Code estão além dos limites desse modelo:

1. **Seleção com mouse**: requer mapeamento de coordenadas de célula ao nível da tela (hit-test); o modelo de string não suporta.
2. **Destaque de busca**: requer sobrepor células invertidas em frames renderizados; é necessário operar na matriz de células.
3. **Rolagem por hardware DECSTBM**: requer consistência entre frames prev/next na área de rolagem; o diff de string não consegue simular o efeito de rolagem por hardware do terminal.
4. **Posicionamento de cursor IME / a11y**: requer saber exatamente as coordenadas físicas na tela do cursor de entrada, posicionando o cursor físico do terminal na posição correta (o texto de preedit do método de entrada CJK é renderizado na posição do cursor físico).
5. **Atomicidade da tela alternativa (BSU/ESU)**: requer encapsular erase + paint em `\x1b[?2026h`/`\x1b[?2026l`, evitando que terminais externos como tmux renderizem estados intermediários.

### Considerações de Performance com Double Buffering

A saída do terminal é uma operação intensiva de I/O; chamadas de sistema `write()` têm overhead fixo. A estratégia de diff com double buffering reduz a escrita por frame de O(tamanho da tela) para O(área de mudança). Comentários de medição mostram: frames de drain-only (apenas rolagem DECSTBM, sem commit React) geram aproximadamente ~10 patches, ~200 bytes de saída (`ink.tsx:756`), enquanto redesenho completo requer vários milhares de bytes.

A flag `prevFrameContaminated` é a rede de segurança de correção: quando operações como destaque de seleção, `resetFramesForAltScreen()`, `forceRedraw()` modificam o conteúdo do frontFrame, o próximo frame deve fazer diff completo em vez de depender da otimização de blit incremental (`ink.tsx:743`).

### Motivação de Design do Modo Vim

O núcleo de usuários do Claude Code são usuários intensivos de Vim — para eles, usar atalhos Vim na caixa de prompt não é opcional, mas um requisito básico. A implementação de máquina de estados pura (sem dependências, sem efeitos colaterais) torna a lógica Vim completamente testável em unidade, desacoplada da camada de renderização de terminal.

O comentário no início de `types.ts` revela a filosofia de design: **tipos são documentação**. O union type de `CommandState` descreve a máquina de estados de forma mais precisa do que qualquer comentário — a verificação exaustiva de switch do TypeScript garante que cada estado seja tratado; novas omissões de estados causam erros em tempo de compilação (`types.ts:1`).

---

## 12.8 Padrões Transferíveis

Os seguintes padrões de design podem ser adotados em outras aplicações de terminal ou frameworks de UI:

**1. Buffer de Tela Pool Compartilhado + ID Inteiro**
Usar `Int32Array` para armazenar IDs inteiros de caracteres/estilos em vez de strings ou objetos. Pool compartilhado entre frames reduz diff a comparações de inteiros, eliminando completamente a pressão de GC. Aplicável a qualquer cenário de diff de matriz retangular de alta frequência.

**2. Tipo como Máquina de Estados**
Definir máquina de estados com TypeScript union type; cada estado carrega exatamente os dados necessários; funções de transição usam switch exaustivo. O valor de retorno duplo `{next?, execute?}` de `transition()` + `TransitionResult` é extremamente conciso e pode ser portado diretamente para outros cenários de tratamento de teclas.

**3. Diff Incremental Orientado por Retângulo de Dano (Damage Rectangle)**
Registrar a região retangular realmente escrita (`damage`) durante a renderização; o diff só varre essa região. Aplicável a qualquer sistema de frame buffer — não apenas terminais, mas também UIs de jogos e exibições embarcadas podem usar esse padrão.

**4. Double Buffering + Rede de Segurança prevFrameContaminated**
Quando algumas operações não seguem o caminho de renderização normal (modificando diretamente o frame frontal), usar um bool flag para forçar diff completo no próximo frame, em vez de quebrar o invariante do double buffering. Mais eficiente do que redesenho completo a cada vez; mais correto do que permitir dados sujos.

**5. Atraso de Renderização por Microtask**
O padrão `scheduleRender = throttle(() => queueMicrotask(onRender), 16)` garante que efeitos de layout React completem antes da renderização, evitando atraso de um frame na posição do cursor. Igualmente aplicável em outros cenários React + renderizador personalizado.

**6. Cache de Largura de Linha (Otimização Específica para Streaming)**
O cálculo `stringWidth` para grandes quantidades de linhas inalteradas durante streaming é um hot spot. A estratégia simples de `Map + limpar quando cheio` de `line-width-cache.ts` resolve um overhead de cálculo de `~50x` em 20 linhas de código (citação do comentário original); pode ser portada diretamente para qualquer cenário de renderização de texto em streaming.

---

## 12.9 Índice de Código-Fonte

| Arquivo | Linhas | Responsabilidade Central |
|---------|--------|--------------------------|
| `src/ink/ink.tsx` | 1.722 | Classe Ink: gerenciamento de frames, agendamento de renderização, SIGCONT, transferência para tela alternativa, seleção com mouse, destaque de busca |
| `src/ink/screen.ts` | 1.300+ | Tipo Screen, layout de células Int32Array, CharPool/StylePool/HyperlinkPool |
| `src/ink/log-update.ts` | ~400 | LogUpdate: motor de diff de frames prev/next, rolagem por hardware DECSTBM |
| `src/ink/optimizer.ts` | ~100 | Otimizador de diff em single pass |
| `src/ink/reconciler.ts` | 512 | Implementação do Custom Renderer hospedeiro React |
| `src/ink/render-node-to-output.ts` | 1.462 | Renderização de nós DOM para Screen |
| `src/ink/constants.ts` | 1 | `FRAME_INTERVAL_MS = 16` |
| `src/ink/line-width-cache.ts` | ~30 | Cache de largura de linha LRU (4.096 entradas) |
| `src/ink/output.ts` | ~300 | Renderizador Output, cache de caracteres (16.384 entradas) |
| `src/vim/types.ts` | ~150 | Definições de tipo VimState, CommandState, PersistentState e constantes |
| `src/vim/transitions.ts` | ~350 | Entrada principal `transition()`, 11 funções de transição de estado |
| `src/vim/motions.ts` | ~80 | Função pura de movimento de cursor `resolveMotion()` |
| `src/vim/operators.ts` | ~500 | Funções de execução delete/change/yank/x/r/~/J/paste/indent/openLine |
| `src/vim/textObjects.ts` | ~220 | Busca de limites de objetos de texto como palavras/aspas/parênteses |
| `src/screens/REPL.tsx` | 5.005 | Interface principal REPL, gerenciamento de sessão, fluxo de mensagens, troca de modo tela cheia/normal |
| `src/components/PromptInput/PromptInput.tsx` | 2.338 | Componente de entrada, roteamento de modo Vim/Standard, gerenciamento de cursor, injeção STT |
| `src/components/Settings/Config.tsx` | 1.821 | Interface de configurações, navegação por busca, leitura/escrita de configuração multi-fonte, restauração com Escape |
| `src/components/LogSelector.tsx` | 1.574 | Seleção de histórico de sessões, busca Fuse.js, exibição em árvore de fork |

**Referência rápida de constantes-chave**:
- `FRAME_INTERVAL_MS = 16` (`constants.ts:1`)
- `MAX_CACHE_SIZE = 4096` (`line-width-cache.ts`)
- Limite do `charCache`: `16384` (`output.ts:204`)
- `MAX_VIM_COUNT = 10000` (`vim/types.ts:96`)
- Layout de célula: 8 bytes/célula, `word0=charId`, `word1=styleId[31:17]|hyperlinkId[16:2]|width[1:0]` (`screen.ts:356`)
