---
name: claude-session-metrics
description: >-
  Catálogo das métricas que podem ser extraídas de forma determinística de um único arquivo
  .jsonl de transcrição de sessão do Claude Code, sem chamar nenhum modelo. Use ao projetar
  ou implementar os cálculos de um dashboard de análise de sessão. Cada métrica traz a
  categoria (Direta / Calculada / Heurística / Estrutural / Fonte externa), a fórmula e os
  campos de origem. Depende da skill `claude-jsonl-format` para o schema.
---

# Métricas determinísticas de uma sessão do Claude Code

Pré-requisito: a skill `claude-jsonl-format` (schema, árvore, dedup, placeholder de tokens).
Todas as fórmulas abaixo assumem que você já:
- parseou o arquivo tolerando linhas quebradas,
- construiu o índice `uuid → linha` e o `mainPath` (raiz → folha de maior `timestamp`),
- deduplicou as linhas `assistant` por `requestId`,
- classificou cada linha `user` em `human` / `tool_result` / `meta`.

## Categorias

| Categoria | Definição | Como reportar |
|---|---|---|
| **Direta** | Lida de um campo único, sem transformação (no máx. um cast string→data) | valor cru |
| **Calculada** | Agregação/aritmética determinística sobre vários campos/linhas | valor + unidade |
| **Heurística** | Regra determinística no código, porém com limiar/dicionário escolhido por julgamento | valor **+ o limiar usado** |
| **Estrutural** | Derivada da topologia da árvore `parentUuid` (grafo, não aritmética) | valor |
| **Fonte externa** | Determinística só com uma referência fora do arquivo (tabela de preços, catálogo de janelas de contexto, tokenizer, corpus histórico) | valor + qual referência/versão |

**Fora de escopo (não determinístico):** resumo de tópico, extração de decisões, sentimento
semântico, rotulagem de intenção — qualquer coisa que precise de um LLM. Exceção: `ai-title` e
`system/away_summary` já vêm prontos no arquivo → lê-los é **Direta**.

---

## 1. Métricas DIRETAS

### Identidade & ambiente
| Métrica | Origem |
|---|---|
| `sessionId`, `cwd`, `gitBranch` | qualquer linha de conversa |
| Versão do CLI, `entrypoint`, `userType` | idem (pode variar no arquivo se houve `--resume` pós-upgrade) |
| Modelo por turno | `assistant.message.model` |
| Nível de esforço por turno | `assistant.effort` |
| Modo / modo de permissão atuais e transições | `mode.mode`, `permission-mode.permissionMode`, `user.permissionMode` |
| Título gerado | `ai-title.aiTitle` |
| Último prompt | `last-prompt.lastPrompt` |
| Recap de ausência | `system[subtype=away_summary].content` |
| Sessão espelhada na nuvem? | `bridge-session.bridgeSessionId` presente |

### Tokens & custo (brutos)
| Métrica | Origem |
|---|---|
| Tokens por turno (4 contadores + thinking) | `assistant.message.usage.*` |
| Chamadas de web search/fetch do servidor no turno | `assistant.message.usage.server_tool_use.*` |
| `service_tier`, `stop_reason` por turno | `assistant.message.usage.service_tier`, `assistant.message.stop_reason` |
| Custo total da sessão | `cost-state.totalCostUSD` (última linha) |
| Custo e tokens por modelo | `cost-state.modelUsage[model]` |
| Flag de custo incompleto | `cost-state.hasUnknownModelCost` |

### Tempo (medido pelo harness)
| Métrica | Origem |
|---|---|
| Duração de cada turno | `system[subtype=turn_duration].durationMs` |
| Duração total / API (com e sem retries) / ferramentas | `cost-state.total{Duration,APIDuration,APIDurationWithoutRetries,ToolDuration}` |
| `startTime` da sessão | `cost-state.startTime` |
| Latência de ferramentas web | `user.toolUseResult.{durationMs,durationSeconds}` |

### Arquivos & ferramentas
| Métrica | Origem |
|---|---|
| Linhas adicionadas/removidas na sessão | `cost-state.totalLinesAdded / totalLinesRemoved` |
| Patch por edição (hunks, `userModified`) | `user.toolUseResult.structuredPatch` |
| Arquivos versionados para rewind | `file-history-delta.trackingPath` |
| `stdout` / `stderr` de cada comando shell | `user.toolUseResult.{stdout,stderr,interrupted}` |
| Skill / servidor MCP / ferramenta MCP do turno | `assistant.attribution{Skill,McpServer,McpTool}` |
| Ferramenta barrada e motivo | `user.toolDenialKind` |
| Turno interrompido | `assistant.isAbortedMidStream`, `user.interruptedMessageId` |
| Erro de API | `assistant.isApiErrorMessage` + texto em `message.content` |
| Feedback explícito do usuário | `user.userFeedback` |
| Comando slash executado | `system[subtype=local_command]`, `attachment[type=command_permissions]` |
| Sessão disparada por agendamento | `system[subtype=scheduled_task_fire]` presente |

---

## 2. Métricas CALCULADAS

### Volume da conversa
| Métrica | Fórmula |
|---|---|
| Nº de prompts humanos | `count(user where classify == "human")` |
| Nº de turnos do assistente | `dedupeAssistants(lines).length` |
| Fan-out por prompt | `turnosAssistente / promptsHumanos` |
| Comprimento do prompt (chars/palavras/≈tokens) | `s.length`, `s.split(/\s+/).length`, `Math.ceil(s.length/4)`; agregar média/mediana/p95/máx |
| Verbosidade da resposta | `Σ b.text.length` para blocos `type:"text"` do turno |

### Ferramentas
| Métrica | Fórmula |
|---|---|
| Histograma de chamadas por ferramenta | `Counter(b.name)` para `tool_use` no `mainPath` |
| Diversidade de ferramentas | nº de nomes distintos; opcional: entropia de Shannon |
| Razão leitura : escrita | `(Read+Grep+Glob+ToolSearch) / (Edit+Write+NotebookEdit)` |
| Razão Edit : Write | `count(Edit) / count(Write)` |
| Chamadas MCP vs. nativas | `count(name.startsWith("mcp__")) / totalToolCalls` |
| Uso de subagentes | `count(tool_use name in {Task,Agent})`; `count(lines where isSidechain)`; `Σ tokens` das linhas sidechain |
| Invocações de skill | `count(tool_use name == "Skill")` ∪ `distinct(attributionSkill)` |
| Total web fetch/search | `Σ server_tool_use.web_fetch_requests + count(tool_use WebFetch)` |

### Tokens & custo (recomputados dos brutos)
| Métrica | Fórmula |
|---|---|
| Tokens de entrada efetivos | `Σ (cache_read + cache_creation + input_tokens)` sobre `dedupeAssistants` |
| Tokens de saída totais | `Σ output_tokens` |
| Tokens de raciocínio | `Σ output_tokens_details.thinking_tokens` |
| Taxa de acerto de cache | `Σ cache_read / Σ (cache_read + cache_creation + input_tokens)` |
| Overhead de retries | `totalAPIDuration - totalAPIDurationWithoutRetries` |
| Distribuição de tokens/turno | média/mediana/p95/máx de entrada e de saída por turno |
| Curva de contexto | série `cache_read_input_tokens` vs. `timestamp`; pico = ocupação máx; quedas bruscas ≈ compactações |

### Tempo & ritmo
| Métrica | Fórmula |
|---|---|
| Duração de parede | `max(timestamp) - min(timestamp)` |
| Tempo ativo | `Σ turn_duration.durationMs` |
| Tempo ocioso | `parede - ativo` |
| Gaps de reflexão do usuário | `ts(promptHumano_i) - ts(fimResposta_{i-1})`; distribuição |
| Throughput de saída (tok/s) | `output_tokens do turno / (turn_duration.durationMs / 1000)` |
| Fração de tempo em ferramentas | `totalToolDuration / totalDuration` |
| Dias-calendário distintos | `distinct(timestamp.slice(0,10))` |
| Nº de retomadas | nº de gaps `> N h` entre linhas consecutivas **ou** mudanças de `version` |

### Fluxo de trabalho
| Métrica | Fórmula |
|---|---|
| Turnos até a 1ª edição | índice do 1º `tool_use` `Edit`/`Write` na sequência de turnos |
| Arquivos distintos tocados | `distinct(structuredPatch[].filePath)` (∪ `file-history-delta.trackingPath`) |
| Net de linhas | `totalLinesAdded - totalLinesRemoved` |
| Negações / interrupções por turno | `Σ toolDenialKind`, `Σ isAbortedMidStream`, `Σ interrupted` ÷ nº de turnos |
| Nº de compactações | `count(type=="summary" || isCompactSummary)` |
| Nº de erros de API | `count(isApiErrorMessage)` |
| Turnos vazios | `count(assistant sem text e sem tool_use)` |
| Distribuição de `permissionMode` | fração de turnos em cada modo |
| Attachments por tipo | `Counter(attachment.type)` |

---

## 3. Métricas ESTRUTURAIS (grafo `parentUuid`)

Constrói-se o mapa `parentUuid → filhos[]` uma vez.

| Métrica | Fórmula |
|---|---|
| Nº de branch points | `count(uuid u where children(u).length > 1)` |
| Nº de rewinds / prompts editados | branch points onde o filho mais novo tem `timestamp` bem posterior ao filho original |
| Trabalho descartado | `totalNós - mainPath.length`; `Σ output_tokens` das linhas fora do `mainPath` |
| Profundidade do caminho final | `mainPath.length` (≠ nº de linhas do arquivo quando há ramos) |
| Razão de linearidade | `mainPath.length / totalNósDeConversa` (1.0 = sem rewind) |
| Comprimento das cadeias de ferramenta | nº de `tool_use` consecutivos entre dois blocos `text` |
| Integridade `tool_use` ↔ `tool_result` | todo `tool_use.id` tem `tool_result.tool_use_id`? contar órfãos |
| Árvore de subagentes | agrupar linhas `isSidechain` pelo `Task` pai; profundidade de aninhamento; tokens por subárvore |

---

## 4. Métricas HEURÍSTICAS (regra fixa, limiar opinativo — **exponha o limiar na UI**)

| Métrica | Regra (limiar entre `[ ]`) |
|---|---|
| Taxa de erro de ferramenta | `erro = stderr não vazio` OU `regex /(error|exception|traceback|fatal|not found|cannot|failed|ENOENT|command not found)/i` em `stdout+stderr` OU `toolUseResult.is_error`. `taxa = Σ erro / Σ tool_results` |
| "Travado" (thrash) | `[≥3]` `tool_use` com mesmo `(name, JSON.stringify(input))` numa janela de `[5]` turnos, OU `[≥2]` `Bash` com erro consecutivos |
| Turno de correção do usuário | `len(prompt) < [200]` E `regex /^(n[aã]o|no|stop|para|errado|wrong|isso n[aã]o|actually|na verdade|volta|undo|revert|de novo|again)/i` E o turno anterior do assistente não pediu confirmação |
| Score de frustração | `+1` por palavrão / "já falei" / "de novo" / "pela última vez"; `+1` se `[>50%]` das palavras em CAPS; `+1` se `[≥3]` "!"; `+1` por turno de correção |
| Rework no mesmo arquivo | `[≥3]` edições no mesmo `filePath`, OU `Edit(path)` até `[2]` turnos depois de um `Bash` com erro que cite `path` |
| Sucesso "one-shot" | prompt → assistente edita → próximo evento humano é positivo ou é o fim da sessão, sem turno de correção no meio |
| Segmentação em tarefas | nova tarefa quando gap ocioso `> [10] min` E Jaccard de tokens `< [0.2]` entre o novo prompt e o anterior |
| Fase exploração → execução | janela deslizante de `[5]` turnos: maioria leitura/busca = "explorando"; maioria Edit/Write/Bash = "executando"; marcar o turno de virada |
| Profundidade de raciocínio (proxy) | `thinking` vazio → `signature.length` como proxy; `thinking` presente → `thinking_tokens / output_tokens` |
| Sinal de TDD | `regex /(pytest|npm (run )?test|yarn test|go test|cargo test|jest|vitest|mvn test|dotnet test)/` em comandos `Bash`; e se a última execução passou |
| Higiene de git | `git commit` apareceu no `Bash`? razão commits / linhas alteradas; houve `git push` / `--no-verify`? |
| Abandono | sessão termina em `[≤2]` turnos após um erro (ferramenta ou API) não seguido de turno de recuperação |
| Uso de planejamento | `attachment[type=plan_mode]` + `plan_mode_exit`; fração da sessão em plan mode; plano aprovado vs. descartado |
| Hedging do assistente | frequência de "acho que" / "talvez" / "pode ser" / "not sure" / "I think" por `[1000]` palavras de texto visível |
| Scope creep | `arquivosDistintosTocados / arquivosCitadosNoPromptInicial` |

---

## 5. Métricas DE FONTE EXTERNA

Determinísticas **dado** um artefato que não está no `.jsonl`. Documente a versão da referência.

| Métrica | Fórmula | Referência |
|---|---|---|
| Custo recomputado por turno | `Σ (tok_input × preço_in + tok_output × preço_out + cache_write × preço_cw + cache_read × preço_cr)` por modelo | tabela de preços Anthropic por modelo/tipo de token |
| % de ocupação da janela de contexto | `(cache_read + cache_creation + input_tokens) / limiteDoModelo` | catálogo de janelas (200k / 1M …) por `model` |
| Contagem exata de tokens do prompt | tokenizar `message.content` dos prompts humanos | tokenizer local ou endpoint `count_tokens` |
| Custo por linha de código | `totalCostUSD / netLines` | tabela de preços (via `cost-state`) |
| Custo por tarefa | `totalCostUSD / nºTarefas` (tarefas via segmentação heurística) | idem |
| Percentil vs. histórico | posição desta sessão na distribuição de todas as sessões em `~/.claude/projects/` (custo, duração, taxa de erro, tokens) | o corpus histórico completo do usuário |
| Economia real do cache | `Σ cache_read × (preço_in − preço_cr) − Σ cache_creation × (preço_cw − preço_in)` | tabela de preços |

---

## 6. Recomendações para o dashboard (front-end only)

- **Pipeline puro**: `File → text → parseTranscript → { índice, mainPath, assistantsDedup } → métricas`.
  Nenhuma métrica desta skill precisa de rede (exceto as da §5, que precisam de um JSON de
  referência que você pode empacotar no bundle).
- **Camadas de confiança na UI**: agrupe visualmente por categoria. Métricas Heurísticas devem
  mostrar o limiar e permitir ajuste (slider) recalculando ao vivo.
- **Sempre exiba os avisos de qualidade**: se `hasUnknownModelCost`, se houver linhas
  descartadas no parse, se `mainPath.length < totalNós` (houve rewind), se faltar `cost-state`.
- **Nunca** derive número de "tokens de entrada" de `input_tokens` sozinho (ver
  `claude-jsonl-format` §2.1).
- Teste com arquivos de várias versões de CLI (campo `version`) — o schema evolui.
