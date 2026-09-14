# Métricas de Transcrições do Claude Code

> Deep research sobre o que os arquivos `.jsonl` de sessão do Claude Code registram e quais
> medidas podem ser extraídas deles de forma **programática e determinística**, classificadas
> por origem da informação.
>
> Base: inspeção direta de 91 arquivos de sessão em `"C:\Users\gabri\.claude\projects\"`
> (Claude Code CLI 2.1.235–2.1.251), cruzada com trabalho da comunidade.
> Versão artefato: https://claude.ai/code/artifact/9f0d54ef-1f31-43dc-acc7-81a91ecf18e9

---

## 1. Como a transcrição é armazenada

Cada sessão interativa é gravada como um arquivo **JSON Lines**: uma linha = um objeto JSON =
um evento, em ordem cronológica de escrita.

- **Local:** `"~/.claude/projects/<cwd-slug>/<sessionId>.jsonl"`
  O `<cwd-slug>` deriva do diretório de trabalho (barras e dois-pontos viram hífens).
- O arquivo é *append-only* durante a sessão e reaberto/anexado quando a sessão é retomada
  (`--resume` / `--continue`).
- Os eventos **não** formam uma lista linear e sim uma **árvore**: cada linha de conversa tem
  `uuid` e `parentUuid`. Editar um prompt anterior ou usar rewind cria um novo ramo a partir
  de um nó antigo — o arquivo mantém os dois. A "conversa real" é o caminho da raiz até a
  última folha.

### 1.1. Tipos de linha observados no formato atual (CLI 2.1.x)

| `type` | Papel | Campos-chave |
|---|---|---|
| `user` | Prompt humano **OU** resultado de ferramenta **OU** mensagem meta (injeção de skill/attachment) | `message.content`, `toolUseResult`, `promptSource`, `origin.kind`, `isMeta`, `isSidechain`, `toolDenialKind`, `interruptedMessageId`, `userFeedback` |
| `assistant` | Turno do modelo (texto, `thinking`, `tool_use`) | `message.model`, `message.usage`, `message.stop_reason`, `requestId`, `effort`, `apiBlockIndex`, `attributionSkill`, `attributionMcpServer/Tool`, `isApiErrorMessage`, `isAbortedMidStream` |
| `system` | Eventos do harness | `subtype` ∈ {`turn_duration`, `local_command`, `away_summary`, `informational`, `scheduled_task_fire`}, `durationMs`, `messageCount`, `content` |
| `attachment` | Contexto injetado no turno | `attachment.type` ∈ {`total_tokens_reminder`, `task_reminder`, `skill_listing`, `deferred_tools_delta`, `agent_listing_delta`, `mcp_instructions_delta`, `file`, `edited_text_file`, `plan_mode`/`plan_mode_exit`, `command_permissions`, `queued_command`, `hook_system_message`, …} |
| `cost-state` | Acumulador de custo/uso da sessão (snapshot periódico) | `totalCostUSD`, `totalAPIDuration(WithoutRetries)`, `totalToolDuration`, `totalLinesAdded/Removed`, `totalDuration`, `startTime`, `modelUsage{}` |
| `file-history-snapshot` / `file-history-delta` | Rastreio de arquivos tocados para backup/rewind | `trackingPath`, `backup.version`, `backupTime`, `snapshotMessageId` |
| `ai-title` / `last-prompt` | Título gerado e último prompt (cache de UI) | `aiTitle`, `lastPrompt` |
| `mode` / `permission-mode` | Estado atual e transições de modo | `mode`, `permissionMode` |
| `queue-operation` | Fila de prompts/notificações de tarefa | `operation`, `content` |
| `bridge-session` / `atis-latch` / `agent-name` | Metadados de sessão remota / identidade | `bridgeSessionId`, `ownerAccountUuid`, `ownerOrganizationUuid` |

Outras linhas existem no formato mas não apareceram neste corpus: `type:"summary"` e
`isCompactSummary:true` (compactação de contexto), `isSidechain:true` em turnos de subagente,
`type:"x-macos-*"`. **Um leitor robusto deve ignorar `type` desconhecido.**

---

## 2. Três armadilhas que tornam uma métrica "determinística" em errada

1. **`usage.input_tokens` é um placeholder de streaming.**
   Na maioria das linhas `assistant` ele vale `0`, `1` ou `2`. O tamanho real do prompt é
   `cache_read_input_tokens + cache_creation_input_tokens + input_tokens`. Somar só
   `input_tokens` subcontabiliza a entrada em 1–2 ordens de magnitude.

2. **Linhas duplicadas por streaming.**
   A mesma resposta pode aparecer em várias linhas com o mesmo `requestId` (e `apiBlockIndex`
   incremental). Para custo/tokens, deduplique por `(requestId, apiBlockIndex)` e use a última
   ocorrência; ou prefira o `cost-state` mais recente.

3. **`cost-state` vs. soma manual.**
   `cost-state.totalCostUSD` e `modelUsage[].costUSD` já são calculados pelo CLI com a tabela
   de preços vigente. Recalcular a partir de tokens exige uma tabela de preços externa
   (categoria própria abaixo) e vai divergir por arredondamento e por `hasUnknownModelCost`.

---

## 3. Categorias de métrica

A classificação pedida — *tiradas diretamente*, *calculadas*, *heurísticas* — cobre a maior
parte dos casos. A pesquisa nos logs revelou **dois grupos que merecem tipo próprio**, mais um
explicitamente fora de escopo:

| Tipo | Definição |
|---|---|
| **Direta** | Lida de um campo único de uma linha, sem transformação. No máximo um cast de tipo (string→data). Se o campo existe, a métrica existe. |
| **Calculada** | Agregação/aritmética determinística sobre vários campos ou linhas: somas, contagens, razões, diferenças de tempo, distribuições. Mesma entrada ⇒ mesma saída, sempre. |
| **Heurística** | Regra determinística no código, mas com *limiar ou dicionário arbitrário* escolhido por julgamento (ex.: "≥3 repetições = travado"). Reproduzível, porém a definição é opinativa e calibrável. |
| **Estrutural / de grafo** *(novo)* | Derivada da topologia da árvore `parentUuid`, não de aritmética: ramificações, profundidade, rewinds, ramos mortos, comprimento do caminho final. Determinística, mas é análise de grafo. |
| **De fonte externa** *(novo)* | Determinística *dada* uma referência que não está no arquivo: tabela de preços por modelo, catálogo de janelas de contexto, um tokenizer real, ou o corpus histórico do usuário (para percentis). |

### Fora de escopo: "Inferida por LLM"

Resumo de tópico, extração de decisões, classificação de sentimento semântico, rotulagem de
intenção. **Não é determinística** e depende de um modelo. Observação: `ai-title` e
`system/away_summary` **já vêm prontos** no arquivo — lê-los é métrica **Direta**; gerá-los, não.

---

## 4. Métricas — DIRETA (tiradas diretamente do documento)

Identificação, ambiente e valores que o CLI já persiste.

### Identidade & ambiente

| Métrica | Descrição | Origem |
|---|---|---|
| Session ID / projeto / branch | Identificação da sessão, diretório de trabalho e branch git no momento de cada evento | `user/assistant → sessionId · cwd · gitBranch` |
| Versão do CLI e ponto de entrada | Versão exata, `entrypoint` (`cli`, …), `userType`. Pode mudar no meio do arquivo se a sessão foi retomada após upgrade | `user/assistant → version · entrypoint · userType` |
| Modelo por turno | Modelo que respondeu cada turno (`claude-sonnet-5`, `claude-haiku-4-5-…`). Uma sessão mistura modelos | `assistant → message.model` |
| Nível de esforço de raciocínio | Campo `effort` (`low`/`medium`/`high`/…) associado ao turno | `assistant → effort` |
| Modo e modo de permissão | Estado corrente (`normal`/`plan`/…) e política (`auto`, `default`, `acceptEdits`, `bypassPermissions`, `plan`). Linhas dedicadas registram cada transição | `mode → mode` · `permission-mode → permissionMode` · `user → permissionMode` |
| Título gerado & último prompt | Resumo curto da sessão e texto do último prompt, já materializados pelo CLI | `ai-title → aiTitle` · `last-prompt → lastPrompt` |
| Recap de ausência ("away summary") | Texto em linguagem natural resumindo o que foi feito enquanto o usuário estava fora — pré-computado | `system[subtype=away_summary] → content` |
| Sessão remota / bridge | Se a sessão foi espelhada para a nuvem (app/web), com `bridgeSessionId`, conta e organização dona | `bridge-session → bridgeSessionId · ownerAccountUuid · ownerOrganizationUuid` |

### Tokens & custo (valores brutos por turno)

| Métrica | Descrição | Origem |
|---|---|---|
| Uso de tokens por turno | Os quatro contadores da API por resposta + `thinking_tokens` + split `ephemeral_1h`/`ephemeral_5m` do cache | `assistant → message.usage` |
| Chamadas de ferramenta de servidor | Contagem de `web_search` e `web_fetch` executadas do lado do servidor naquele turno | `assistant → message.usage.server_tool_use.{web_search_requests, web_fetch_requests}` |
| Tier de serviço & razão de parada | `service_tier` (`standard`/`priority`/`batch`) e `stop_reason` (`end_turn`, `tool_use`, `max_tokens`, …) | `assistant → message.usage.service_tier · message.stop_reason` |
| Acumuladores de custo da sessão | Snapshot já calculado: custo total USD, custo e tokens por modelo, flag `hasUnknownModelCost`. Use o `cost-state` de maior `timestamp` | `cost-state → totalCostUSD · modelUsage` |

Campos brutos de `usage`:

```
usage.input_tokens                  (placeholder — ver armadilha 1)
usage.cache_creation_input_tokens
usage.cache_read_input_tokens
usage.output_tokens
usage.output_tokens_details.thinking_tokens
```

Campos de `cost-state.modelUsage`:

```
modelUsage[model].{inputTokens, outputTokens,
                   cacheReadInputTokens, cacheCreationInputTokens,
                   webSearchRequests, costUSD}
```

### Tempo (medido pelo harness)

| Métrica | Descrição | Origem |
|---|---|---|
| Duração de cada turno | Tempo de parede de um turno completo, medido pelo CLI, com a contagem de mensagens do turno | `system[subtype=turn_duration] → durationMs · messageCount` |
| Durações agregadas da sessão | `totalDuration`, `totalAPIDuration` / `totalAPIDurationWithoutRetries`, `totalToolDuration`, `startTime` (epoch ms) | `cost-state` |
| Duração de ferramenta individual | Algumas ferramentas gravam sua própria latência (`WebFetch`, `WebSearch`, `ToolSearch`) | `user → toolUseResult.{durationMs, durationSeconds}` |

### Edições de arquivo & ferramentas

| Métrica | Descrição | Origem |
|---|---|---|
| Linhas adicionadas / removidas na sessão | Total acumulado mantido pelo CLI | `cost-state → totalLinesAdded · totalLinesRemoved` |
| Patch estruturado por edição | Para cada `Edit`/`Write`: caminho, hunks (`oldStart`/`oldLines`/`newStart`/`newLines` + linhas), e `userModified` | `user → toolUseResult.{filePath, structuredPatch, userModified, originalFile}` |
| Arquivos rastreados para rewind | Lista de todos os caminhos versionados durante a sessão, com número de versão e timestamp de cada backup | `file-history-delta → trackingPath · backup.version · backupTime` |
| Resultado bruto de comando shell | `stdout`, `stderr`, flag `interrupted`, `isImage` (não há código de saída numérico próprio para `Bash`) | `user → toolUseResult.{stdout, stderr, interrupted}` |
| Atribuição de skill / MCP no turno | Origem carimbada quando o turno foi produzido sob skill ativa ou ferramenta MCP | `assistant → attributionSkill · attributionMcpServer · attributionMcpTool` |
| Negação de permissão | Cada vez que o usuário ou uma regra barrou uma ferramenta, com motivo: `user-rejected`, `permission-rule`, `automode-blocked` | `user → toolDenialKind` |
| Interrupção de stream | Turno cortado pelo usuário no meio da geração | `assistant → isAbortedMidStream` · `user → interruptedMessageId` |
| Erro de API | Resposta que foi na verdade um erro (overload, context length, etc.), marcada explicitamente | `assistant → isApiErrorMessage · message.content · error` |
| Feedback explícito do usuário | Feedback estruturado sobre uma resposta (thumbs / comentário) | `user → userFeedback` |
| Comandos slash locais executados | Registro de `/comando` disparados e argumentos | `system[subtype=local_command]` · `attachment[type=command_permissions]` |
| Disparo de tarefa agendada | Se a sessão foi iniciada/retomada por um cron/routine | `system[subtype=scheduled_task_fire]` |

---

## 5. Métricas — CALCULADA (agregação determinística)

Contagens, somas, razões e diferenças de tempo. Todas exigem primeiro **reconstruir o caminho
da árvore** (raiz→folha final) e **deduplicar** linhas de streaming por `requestId`.

### Volume da conversa

| Métrica | Fórmula / definição |
|---|---|
| Nº de prompts humanos | `count(user WHERE origin.kind=="human" AND promptSource=="typed" AND NOT isMeta AND NOT isSidechain AND message.content é string)` |
| Nº de turnos do assistente / mensagens por turno | `turnos = count(assistant, dedup requestId)` ; `fan_out = turnos / nº_prompts_humanos` |
| Comprimento do prompt do usuário | Caracteres, palavras e (est.) tokens `~len/4` de cada prompt; distribuição média/mediana/máx |
| Verbosidade da resposta | `sum(len(b.text) for b in content if b.type=="text")` por turno (exclui `thinking` e `tool_use`) |

### Ferramentas

| Métrica | Fórmula / definição |
|---|---|
| Total de chamadas & histograma por nome | `Counter(b.name for assistant.content b WHERE b.type=="tool_use")` |
| Diversidade de ferramentas | Nº de ferramentas distintas; entropia de Shannon da distribuição de uso |
| Razão leitura : escrita | `(Read + Grep + Glob + ToolSearch) / (Edit + Write + NotebookEdit)` |
| Razão Edit : Write | Edição incremental vs. reescrita completa |
| Chamadas MCP vs. nativas | `count(name LIKE "mcp__%") / total_tool_calls` |
| Uso de subagentes | Nº de `Task`/`Agent`; nº de turnos `isSidechain`; tokens gastos em sidechains |
| Invocações de skill | Contagem e lista, via `tool_use` name `Skill` e via `attributionSkill` distintos |
| Web fetch / search total | `sum(server_tool_use.web_fetch_requests) + count(tool_use WebFetch)` |

### Tokens & custo (recomputados a partir dos brutos)

| Métrica | Fórmula |
|---|---|
| Tokens de entrada efetivos | `entrada = Σ_turnos (cache_read + cache_creation + input_tokens)` — após dedup por `requestId` |
| Tokens de saída totais & de raciocínio | `saida = Σ output_tokens` ; `raciocinio = Σ output_tokens_details.thinking_tokens` |
| Taxa de acerto de cache | `cache_read / (cache_read + cache_creation + input_tokens)` |
| Overhead de retries | `totalAPIDuration - totalAPIDurationWithoutRetries` |
| Distribuição de tokens por turno | Média, mediana, p95, máx de entrada e saída por turno |
| Curva de crescimento de contexto | `cache_read_input_tokens` ao longo do tempo ≈ ocupação da janela; pico e nº de quedas bruscas ≈ compactações |

### Tempo & ritmo

| Métrica | Fórmula |
|---|---|
| Duração de parede da sessão | `max(timestamp) - min(timestamp)` |
| Tempo ativo vs. ocioso | ativo = `Σ turn_duration.durationMs` ; ocioso = parede − ativo ; gap do usuário `gap[i] = ts(prompt_i) - ts(fim_resposta_{i-1})` |
| Throughput de saída | `output_tokens do turno / turn_duration.durationMs` → tok/s |
| Fração de tempo em ferramentas | `totalToolDuration / totalDuration` |
| Span temporal por dia / retomadas | Nº de dias-calendário distintos; nº de reabres (gap > N h entre linhas consecutivas, ou mudança de `version`) |

### Fluxo de trabalho

| Métrica | Fórmula / definição |
|---|---|
| Turnos até a primeira edição de código | Quantos turnos de assistente antes do primeiro `Edit`/`Write` |
| Arquivos distintos tocados / net de linhas | `distinct(structuredPatch.filePath)` ; `net = totalLinesAdded - totalLinesRemoved` |
| Contagem de negações e interrupções | `Σ toolDenialKind`, `Σ isAbortedMidStream`, `Σ interrupted:true` — normalizados por nº de turnos |
| Nº de compactações de contexto | `count(type=="summary" OR isCompactSummary==true)` |
| Nº de erros de API e de turnos vazios | `count(isApiErrorMessage)` ; `count(assistant sem text nem tool_use)` |
| Distribuição de modo de permissão | Fração de turnos sob cada `permissionMode` — proxy de autonomia concedida |
| Attachments injetados por turno | Contagem por `attachment.type` — mede quanto contexto o harness empurrou |

---

## 6. Métricas — ESTRUTURAL / DE GRAFO *(tipo sugerido)*

Determinísticas, mas obtidas por travessia do DAG `uuid ← parentUuid`, não por aritmética.
Merecem tipo próprio porque a maioria dos parsers de transcrição trata o arquivo como lista e
**perde** essa informação.

| Métrica | Fórmula / definição |
|---|---|
| Nº de ramos (branch points) | `count(uuid u WHERE |{n : n.parentUuid == u}| > 1)` — pontos de edição de prompt / rewind |
| Nº de rewinds / prompts editados | Ramos que partem de nó não-folha cujo novo filho tem timestamp muito posterior ao filho original |
| Trabalho descartado (dead branches) | `nós_totais - nós_no_caminho_final` ; `Σ output_tokens dos nós fora do caminho` |
| Profundidade do caminho final | Comprimento da cadeia efetiva de conversa (≠ nº de linhas do arquivo quando há ramos) |
| Razão de linearidade | `nós_no_caminho_final / nós_totais_da_árvore` (1.0 = sem rewind) |
| Encadeamento prompt→tool→tool | Comprimento das sequências ininterruptas de `tool_use` entre dois turnos de texto |
| Emparelhamento tool_use ↔ tool_result | Todo `tool_use.id` tem um `tool_result.tool_use_id`? Órfãos indicam interrupção ou truncamento |
| Árvore de subagentes | Agrupar turnos `isSidechain` por `Task` pai; profundidade de aninhamento; tokens por subárvore |

---

## 7. Métricas — HEURÍSTICA (regra determinística, limiar opinativo)

Código reproduzível, mas cada uma embute um número ou lista escolhida à mão.
**Reporte sempre o limiar junto do valor.**

| Métrica | Regra |
|---|---|
| Taxa de erro de ferramenta | `é_erro = stderr não vazio OR regex /(error|exception|traceback|fatal|not found|cannot|failed|ENOENT|command not found)/i em stdout+stderr OR toolUseResult.is_error==true` ; `taxa = Σ é_erro / Σ tool_results` |
| Detecção de "travado" (thrash) | `≥ 3 tool_use com mesmo (name, hash(input)) numa janela de 5 turnos` **ou** `≥ 2 Bash com é_erro consecutivos` |
| Turno de correção do usuário | `len(prompt) < 200` **e** `regex /^(n[aã]o|no|stop|para|errado|wrong|isso n[aã]o|actually|na verdade|volta|undo|revert|de novo|again)/i` **e** o turno anterior do assistente não pediu confirmação |
| Score de frustração / sentimento léxico | `+1` por palavrão / "ainda não" / "já falei" / "pela última vez" ; `+1` por CAPS LOCK em >50% das palavras ; `+1` por `>= 3` exclamações ; `+1` por turno de correção |
| Rework no mesmo arquivo | `K = 3` edições no mesmo path **ou** `Edit(path)` num intervalo de 2 turnos após `Bash` com `é_erro` que mencione esse path |
| Sucesso "one-shot" | Prompt → assistente edita → próximo evento humano é positivo ou é o fim da sessão, sem turno de correção no meio |
| Segmentação em tarefas | Nova fronteira quando: gap ocioso `> T` min **e** Jaccard de tokens `< 0,2` entre o novo prompt e o anterior |
| Fase exploração → execução | Janela deslizante sobre o mix de ferramentas: leitura/busca = "explorando" ; Edit/Write/Bash = "executando" ; marca o turno de virada |
| Profundidade de raciocínio (proxy) | `thinking` vazio (redigido) → tamanho da `signature` como proxy ; `thinking` presente → `thinking_tokens / output_tokens` |
| Sinal de TDD / verificação | Comandos de teste no `Bash` (`pytest`, `npm test`, `go test`, `cargo test`, `jest`, …) ; se a última execução de teste passou |
| Higiene de git | `git commit` no `Bash`? Razão commits / linhas alteradas ; houve `git push` / `--no-verify`? |
| Abandono | Sessão termina em ≤ 2 turnos após um erro não resolvido (ferramenta ou API sem turno de recuperação) |
| Uso de planejamento | Ocorrência de `attachment/plan_mode` + `plan_mode_exit` ; fração da sessão em plan mode ; plano aprovado ou descartado |
| Hedging / confiança do assistente | Frequência de expressões de incerteza ("acho que", "talvez", "pode ser", "not sure", "I think") por 1000 palavras de texto visível |
| Scope creep | Razão entre nº de arquivos distintos tocados e o nº de arquivos citados no prompt inicial |

---

## 8. Métricas — DE FONTE EXTERNA *(tipo sugerido)*

Totalmente determinísticas, mas só calculáveis com um artefato que **não está no arquivo**.
Separadas porque a saída muda quando a referência muda (preços novos, tokenizer diferente) sem
que a transcrição mude.

| Métrica | Descrição | Referência necessária |
|---|---|---|
| Custo recomputado por turno | Reconstrói o custo a partir dos tokens (input / output / cache-write / cache-read). Necessário quando `hasUnknownModelCost==true` ou para reprecificar em outra moeda | Tabela de preços Anthropic por modelo e tipo de token |
| % de ocupação da janela de contexto | `(cache_read + cache_creation + input_tokens) / limite_do_modelo` | Catálogo de janelas de contexto (200k, 1M, …) por `model` |
| Contagem de tokens exata do prompt | O arquivo guarda o texto mas não os tokens dos prompts humanos | Tokenizer real ou endpoint `count_tokens` |
| Custo por linha de código / por tarefa | `totalCostUSD / net_lines` e `totalCostUSD / nº_tarefas` (tarefas via segmentação heurística) | Tabela de preços |
| Percentil vs. histórico do usuário | Onde esta sessão cai na distribuição de todas as sessões em `~/.claude/projects/`: custo, duração, taxa de erro, tokens | O corpus histórico completo |
| Preço efetivo do cache | `tokens_lidos_do_cache × (preço_input_normal − preço_cache_read)` menos o custo de escrita de cache | Tabela de preços |

---

## 9. Roteiro de extração (ordem recomendada)

1. **Parse tolerante:** uma linha por vez, `try/except`, ignore `type` desconhecido e JSON
   quebrado (a última linha pode estar truncada).
2. **Índice por `uuid`** e construção do mapa `parentUuid → filhos`.
3. **Caminho final:** a partir da folha de maior `timestamp`, suba por `parentUuid` até a raiz.
   Só esse caminho conta para métricas de conversa; o resto alimenta as métricas Estruturais.
4. **Dedup de streaming:** agrupe `assistant` por `requestId`; mantenha a linha com
   `message.usage` mais completo (maior `apiBlockIndex`).
5. **Classifique cada `user`:** prompt humano · resultado de ferramenta (`toolUseResult`
   presente) · meta (`isMeta`). Nunca conte os dois últimos como "mensagem do usuário".
6. **Custo/tokens:** prefira o último `cost-state`; só recomponha dos brutos se precisar de
   granularidade por turno — e então use a fórmula de entrada efetiva.
7. **Tempo:** some `turn_duration` para tempo ativo; use min/max de `timestamp` (ISO-8601 UTC)
   para o parede.
8. **Emita o limiar** de toda métrica heurística junto do valor, para a análise ser auditável.

---

## 10. Ferramentas e trabalhos relacionados

- **claude-session-analyzer** — https://github.com/lucemia/claude-session-analyzer
  Replica a metodologia da issue `anthropics/claude-code#42796`: profundidade de raciocínio,
  razão Read:Edit, sinais comportamentais, sentimento, custo.
- **ccusage** — https://ccusage.com/guide/cost-modes
  Agrega os JSONL em resumos diário/mensal/por sessão; documenta os modos de custo (dado vs.
  recomputado) e a necessidade de dedup.
- **"Claude Code's Token Counts Are Wrong"** — https://gille.ai/en/blog/claude-code-jsonl-logs-undercount-tokens/
  Análise do placeholder de `input_tokens` no streaming.
- **claude-code-analytics** — https://deepwiki.com/spences10/claude-code-analytics/3.3-jsonl-transcript-processing
  Pipeline de ingestão de JSONL para SQLite com busca e analytics.
- **"Inside Claude Code: The Session File Format"** — https://databunny.medium.com/inside-claude-code-the-session-file-format-and-how-to-inspect-it-b9998e66d56b
  Passo a passo do formato de sessão.
- **claude-code-transcripts** — https://github.com/simonw/claude-code-transcripts ·
  **Claude Logs viewer** — https://claudelogs.com/tools/log-viewer/
  Renderização legível das transcrições.

---

*O formato JSONL não é documentado oficialmente e muda entre versões — trate nomes de campo
como estáveis por minor, não por major.*
