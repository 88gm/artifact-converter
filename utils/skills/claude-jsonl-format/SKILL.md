---
name: claude-jsonl-format
description: >-
  Base de conhecimento sobre o formato dos arquivos .jsonl de transcrição de sessão do
  Claude Code (~/.claude/projects/<slug>/<sessionId>.jsonl). Use ao ler, parsear, tipar
  ou validar esses arquivos — explica cada tipo de linha, os campos de cada um, a estrutura
  em árvore (uuid/parentUuid), e as armadilhas que quebram análises "determinísticas".
---

# Formato do JSONL de sessão do Claude Code

## 1. O que é o arquivo

- **JSON Lines**: uma linha = um objeto JSON = um evento, em ordem cronológica de escrita.
- **Local**: `~/.claude/projects/<cwd-slug>/<sessionId>.jsonl`. O `<cwd-slug>` é o diretório de
  trabalho com `/`, `\` e `:` trocados por `-`.
- **Append-only** durante a sessão; o arquivo é reaberto e anexado quando a sessão é retomada
  (`--resume` / `--continue`).
- **Não é uma lista linear.** As linhas de conversa (`user` / `assistant`) formam uma **árvore**
  via `uuid` ← `parentUuid`. Editar um prompt anterior ou usar rewind cria um novo ramo a partir
  de um nó antigo; os dois ramos ficam no arquivo. A "conversa real" é o caminho da raiz até a
  folha de maior `timestamp`.
- **Não é documentado oficialmente** e muda entre versões. Trate nomes de campo como estáveis
  por *minor*, não por *major*. Um parser robusto **ignora `type` desconhecido** e **linhas com
  JSON inválido** (a última linha pode estar truncada).

## 2. Regras de ouro para parsing determinístico

Estas três regras são obrigatórias. Ignorá-las produz números errados que *parecem* certos.

### 2.1. `usage.input_tokens` é um placeholder de streaming
Na maioria das linhas `assistant` ele vale `0`, `1` ou `2`. **Nunca** use `input_tokens`
sozinho como "tokens de entrada". O tamanho real do prompt é:

```
inputReal = cache_read_input_tokens + cache_creation_input_tokens + input_tokens
```

### 2.2. Linhas duplicadas por streaming
A mesma resposta pode aparecer em várias linhas com o **mesmo `requestId`** (e `apiBlockIndex`
incremental). Antes de somar tokens/custo, agrupe as linhas `assistant` por `requestId` e
mantenha **uma** por grupo — a de maior `apiBlockIndex` (ou a que tem `message.usage` mais
completo).

### 2.3. `cost-state` já vem calculado
`cost-state.totalCostUSD` e `modelUsage[].costUSD` já foram calculados pelo CLI com a tabela de
preços vigente. Para custo, **prefira o último `cost-state`** (maior `timestamp`). Só recalcule
a partir de tokens se precisar de granularidade por turno — e aí saiba que vai divergir por
arredondamento e por `hasUnknownModelCost`.

### 2.4. Classifique cada linha `user` antes de contar
Uma linha `type:"user"` pode ser três coisas diferentes:
- **prompt humano**: `message.content` é `string`, `isMeta` ausente/falso, `toolUseResult` ausente.
- **resultado de ferramenta**: tem `toolUseResult` e `message.content` é um array com
  `{type:"tool_result"}`.
- **mensagem meta / injeção**: `isMeta:true` (ex.: texto de skill, lembrete de contexto).

Só a primeira conta como "mensagem do usuário".

## 3. Tipos de linha

Todas as linhas de conversa carregam também: `sessionId`, `cwd`, `gitBranch`, `version`,
`userType`, `entrypoint`, `timestamp` (ISO-8601 UTC), `uuid`, `parentUuid`, `isSidechain`.

### 3.1. `user`
| Campo | Significado |
|---|---|
| `message.content` | `string` (prompt humano) ou `array` de blocos `{type:"tool_result", tool_use_id, content}` |
| `toolUseResult` | Objeto com o resultado bruto da ferramenta (ver §4) |
| `sourceToolAssistantUUID` / `sourceToolUseID` | Liga o resultado ao `tool_use` que o originou |
| `promptId` | Agrupa todas as linhas geradas a partir de um mesmo prompt humano |
| `promptSource` | `"typed"`, `"queued"`, … |
| `origin.kind` | `"human"` para prompt digitado |
| `permissionMode` | Modo de permissão no envio do prompt |
| `isMeta` | `true` = injeção de contexto, não é fala do usuário |
| `isSidechain` | `true` = turno dentro de um subagente |
| `toolDenialKind` | `"user-rejected"`, `"permission-rule"`, `"automode-blocked"` — ferramenta barrada |
| `interruptedMessageId` | Presente quando o usuário interrompeu o turno |
| `userFeedback` | Feedback estruturado (thumbs / comentário) sobre uma resposta |
| `classifierMetaLines` | Metadados internos de classificação |

### 3.2. `assistant`
| Campo | Significado |
|---|---|
| `message.model` | `"claude-sonnet-5"`, `"claude-haiku-4-5-20251001"`, … (uma sessão mistura modelos) |
| `message.id` | ID da mensagem da API |
| `message.content[]` | Blocos: `{type:"thinking", thinking, signature}`, `{type:"text", text}`, `{type:"tool_use", id, name, input}` |
| `message.stop_reason` | `"end_turn"`, `"tool_use"`, `"max_tokens"`, … |
| `message.usage` | Ver §5 |
| `requestId` | ID da requisição — **chave de deduplicação** |
| `apiBlockIndex` | Índice incremental de bloco dentro da mesma requisição |
| `effort` | `"low"` / `"medium"` / `"high"` — esforço de raciocínio do turno |
| `attributionSkill` | Nome da skill ativa quando o turno foi produzido |
| `attributionMcpServer` / `attributionMcpTool` | Servidor/ferramenta MCP atribuídos ao turno |
| `isApiErrorMessage` | `true` = a "resposta" é na verdade um erro de API |
| `isAbortedMidStream` | `true` = geração cortada no meio |
| `error` | Objeto de erro, quando houver |
| `slug` | Slug legível da sessão |

### 3.3. `system`
| `subtype` | Conteúdo |
|---|---|
| `turn_duration` | `durationMs` (tempo de parede do turno), `messageCount` (mensagens no turno) |
| `local_command` | Registro de `/comando` executado |
| `away_summary` | `content`: resumo em linguagem natural do que foi feito enquanto o usuário estava fora (pré-computado) |
| `informational` | Mensagens informativas do harness |
| `scheduled_task_fire` | Sessão iniciada/retomada por cron/routine |

### 3.4. `attachment`
Contexto injetado num turno. `attachment.type` mais comuns:
`total_tokens_reminder`, `task_reminder`, `skill_listing`, `deferred_tools_delta`,
`agent_listing_delta`, `mcp_instructions_delta`, `file`, `edited_text_file`, `directory`,
`plan_mode`, `plan_mode_exit`, `auto_mode`, `auto_mode_exit`, `command_permissions`,
`queued_command`, `hook_system_message`, `read_truncation_notice`, `already_read_file`,
`remote_session_change`, `prompt_snapshot`, `environment`, `model`, `date`.

### 3.5. `cost-state` (snapshot periódico, use o de maior `timestamp`)
| Campo | Significado |
|---|---|
| `totalCostUSD` | Custo acumulado da sessão em USD (já calculado) |
| `totalDuration` | Tempo total de parede (ms) desde `startTime` |
| `totalAPIDuration` / `totalAPIDurationWithoutRetries` | Tempo em chamadas de API, com e sem retries |
| `totalToolDuration` | Tempo em execução de ferramentas (ms) |
| `totalLinesAdded` / `totalLinesRemoved` | Linhas de código adicionadas/removidas na sessão |
| `startTime` | Epoch ms do início |
| `modelUsage[model]` | `{ inputTokens, outputTokens, cacheReadInputTokens, cacheCreationInputTokens, webSearchRequests, costUSD }` |
| `hasUnknownModelCost` | `true` = algum modelo sem preço conhecido → `totalCostUSD` é parcial |

### 3.6. `file-history-snapshot` / `file-history-delta`
Rastreio de arquivos versionados para rewind.
`trackingPath` (caminho do arquivo), `backup.version`, `backup.backupTime`, `snapshotMessageId`.

### 3.7. Linhas de metadados / UI
| `type` | Conteúdo |
|---|---|
| `ai-title` | `aiTitle` — título curto gerado para a sessão |
| `last-prompt` | `lastPrompt`, `leafUuid` — cache do último prompt |
| `mode` | `mode` atual (`normal`, `plan`, …) |
| `permission-mode` | `permissionMode` atual (`auto`, `default`, `acceptEdits`, `bypassPermissions`, `plan`) |
| `queue-operation` | `operation` (`enqueue`/…), `content` — fila de prompts e notificações de tarefa |
| `bridge-session` | `bridgeSessionId`, `ownerAccountUuid`, `ownerOrganizationUuid` — espelhamento na nuvem |
| `atis-latch` / `agent-name` | Identidade interna da sessão/agente |

### 3.8. Linhas do formato que podem não aparecer em toda sessão
- `type:"summary"` e linhas com `isCompactSummary:true` → compactação de contexto.
- `isSidechain:true` em `user`/`assistant` → turnos de subagente (spawn via `Task`/`Agent`).
- Prefixos `x-*` / `x-macos-*` → eventos específicos de plataforma.

## 4. `toolUseResult` por ferramenta

O formato do objeto depende da ferramenta que produziu o resultado:

| Ferramenta | Campos de `toolUseResult` |
|---|---|
| `Bash` | `stdout`, `stderr`, `interrupted`, `isImage`, `noOutputExpected` — **não há código de saída numérico** |
| `Edit` / `Write` | `filePath`, `content`, `structuredPatch[]` (`{oldStart, oldLines, newStart, newLines, lines[]}`), `originalFile`, `userModified`, `oldString`, `newString` |
| `Read` | `type`, `file` (`filePath`, `content`, `numLines`, `startLine`, `totalLines`) |
| `Grep` | `mode`, `matches` / `numFiles`, `content` |
| `Glob` | `filenames[]`, `numFiles` |
| `WebFetch` | `bytes`, `code`, `codeText`, `result`, `durationMs`, `url` |
| `WebSearch` | `query`, `results[]`, `durationSeconds`, `searchCount` |
| `ToolSearch` | `query`, `matches[]`, `total_deferred_tools` |
| `Task` / `Agent` | `content`, `totalDurationMs`, `totalTokens`, `totalToolUseCount` |

## 5. `message.usage` (linha `assistant`)

```jsonc
{
  "input_tokens": 2,                       // PLACEHOLDER — ver §2.1
  "cache_creation_input_tokens": 38404,
  "cache_read_input_tokens": 0,
  "output_tokens": 188,
  "output_tokens_details": { "thinking_tokens": 59 },
  "cache_creation": { "ephemeral_1h_input_tokens": 38404, "ephemeral_5m_input_tokens": 0 },
  "server_tool_use": { "web_search_requests": 0, "web_fetch_requests": 0 },
  "service_tier": "standard",
  "iterations": [ /* partials de streaming — não some ingenuamente */ ]
}
```

## 6. Tipos TypeScript de referência

Ponto de partida; ajuste conforme a versão do CLI encontrada no campo `version`.

```ts
type ISODate = string; // ISO-8601 UTC

interface BaseLine {
  type: string;
  sessionId?: string;
  uuid?: string;
  parentUuid?: string | null;
  timestamp?: ISODate;
  cwd?: string;
  gitBranch?: string;
  version?: string;
  isSidechain?: boolean;
  isMeta?: boolean;
}

interface UsageBlock {
  input_tokens: number;                 // placeholder de streaming
  cache_creation_input_tokens: number;
  cache_read_input_tokens: number;
  output_tokens: number;
  output_tokens_details?: { thinking_tokens?: number };
  server_tool_use?: { web_search_requests: number; web_fetch_requests: number };
  service_tier?: "standard" | "priority" | "batch";
}

type ContentBlock =
  | { type: "thinking"; thinking: string; signature: string }
  | { type: "text"; text: string }
  | { type: "tool_use"; id: string; name: string; input: unknown }
  | { type: "tool_result"; tool_use_id: string; content: unknown; is_error?: boolean };

interface AssistantLine extends BaseLine {
  type: "assistant";
  requestId: string;
  apiBlockIndex?: number;
  effort?: "low" | "medium" | "high";
  attributionSkill?: string;
  attributionMcpServer?: string;
  attributionMcpTool?: string;
  isApiErrorMessage?: boolean;
  isAbortedMidStream?: boolean;
  message: {
    model: string;
    id: string;
    role: "assistant";
    content: ContentBlock[];
    stop_reason: "end_turn" | "tool_use" | "max_tokens" | string | null;
    usage: UsageBlock;
  };
}

interface UserLine extends BaseLine {
  type: "user";
  promptId?: string;
  promptSource?: "typed" | "queued" | string;
  origin?: { kind: "human" | string };
  permissionMode?: string;
  toolDenialKind?: "user-rejected" | "permission-rule" | "automode-blocked";
  interruptedMessageId?: string;
  userFeedback?: unknown;
  toolUseResult?: Record<string, unknown>;
  sourceToolUseID?: string;
  message: { role: "user"; content: string | ContentBlock[] };
}

interface SystemLine extends BaseLine {
  type: "system";
  subtype: "turn_duration" | "local_command" | "away_summary"
         | "informational" | "scheduled_task_fire";
  durationMs?: number;
  messageCount?: number;
  content?: string;
}

interface CostStateLine {
  type: "cost-state";
  sessionId: string;
  totalCostUSD: number;
  totalDuration: number;
  totalAPIDuration: number;
  totalAPIDurationWithoutRetries: number;
  totalToolDuration: number;
  totalLinesAdded: number;
  totalLinesRemoved: number;
  startTime: number; // epoch ms
  hasUnknownModelCost: boolean;
  modelUsage: Record<string, {
    inputTokens: number; outputTokens: number;
    cacheReadInputTokens: number; cacheCreationInputTokens: number;
    webSearchRequests: number; costUSD: number;
  }>;
}

type TranscriptLine =
  | AssistantLine | UserLine | SystemLine | CostStateLine
  | (BaseLine & Record<string, unknown>); // demais tipos
```

## 7. Esqueleto de parser (browser, front-end only)

```ts
export function parseTranscript(raw: string): TranscriptLine[] {
  const lines: TranscriptLine[] = [];
  for (const l of raw.split(/\r?\n/)) {
    const s = l.trim();
    if (!s) continue;
    try { lines.push(JSON.parse(s)); }
    catch { /* linha truncada/corrompida — ignore */ }
  }
  return lines;
}

/** Deduplica turnos de streaming e devolve 1 AssistantLine por requestId. */
export function dedupeAssistants(lines: TranscriptLine[]): AssistantLine[] {
  const byReq = new Map<string, AssistantLine>();
  for (const l of lines) {
    if (l.type !== "assistant") continue;
    const a = l as AssistantLine;
    const prev = byReq.get(a.requestId);
    if (!prev || (a.apiBlockIndex ?? 0) >= (prev.apiBlockIndex ?? 0)) {
      byReq.set(a.requestId, a);
    }
  }
  return [...byReq.values()];
}

/** Caminho raiz -> folha de maior timestamp (a "conversa real"). */
export function mainPath(lines: TranscriptLine[]): TranscriptLine[] {
  const byUuid = new Map<string, TranscriptLine>();
  for (const l of lines) if (l.uuid) byUuid.set(l.uuid, l);
  let leaf: TranscriptLine | undefined;
  for (const l of lines) {
    if (!l.uuid || !l.timestamp) continue;
    if (!leaf || l.timestamp > leaf.timestamp!) leaf = l;
  }
  const path: TranscriptLine[] = [];
  let cur = leaf;
  while (cur) { path.unshift(cur); cur = cur.parentUuid ? byUuid.get(cur.parentUuid) : undefined; }
  return path;
}

export function inputReal(u: UsageBlock): number {
  return u.cache_read_input_tokens + u.cache_creation_input_tokens + u.input_tokens;
}

export function classifyUser(l: UserLine): "human" | "tool_result" | "meta" {
  if (l.isMeta) return "meta";
  if (l.toolUseResult || Array.isArray(l.message.content)) return "tool_result";
  return "human";
}
```

## 8. Como usar esta skill

1. Sempre construir o índice `uuid → linha` e o `mainPath` antes de qualquer métrica de conversa.
2. Sempre deduplicar `assistant` por `requestId` antes de somar tokens/custo.
3. Nunca somar `input_tokens` sozinho — usar `inputReal`.
4. Para custo, ler o último `cost-state`; recalcular só é necessário para granularidade fina.
5. Para *quais* métricas extrair e como classificá-las, ver a skill `claude-session-metrics`.
6. Aplicar as invariantes obrigatórias da rule `claude-session-invariants`.
