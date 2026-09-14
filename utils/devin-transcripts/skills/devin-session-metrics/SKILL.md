---
name: devin-session-metrics
description: >-
  Catálogo das métricas que podem ser extraídas de sessões da Devin CLI a partir do
  `sessions.db` e/ou de um transcript ATIF exportado, classificadas por categoria (Direta /
  Calculada / Heurística / Estrutural / Fonte externa) na mesma taxonomia usada para o Claude
  Code. Use ao projetar ou implementar cálculos de dashboard sobre dados da Devin CLI. Depende
  das skills `devin-sessions-db-format` e `devin-atif-format` para o schema de origem.
---

# Métricas extraíveis de sessões da Devin CLI

Pré-requisito: skills `devin-sessions-db-format` (schema do `sessions.db`) e `devin-atif-format`
(schema do transcript exportado). Ambas carregam o aviso de confiança: schema reconstruído por
terceiros, não confirmado por inspeção direta de um arquivo real desta instalação.

**Ambiente corporativo:** este documento é referência congelada, compilada uma única vez fora
do ambiente corporativo (sem acesso à internet, sem novas dependências permitidas). Não repita
consultas externas ao trabalhar neste projeto — sinalize lacunas ao usuário. Note também que
`sessions.db` (SQLite binário) só é lido no dashboard front-end sem nova dependência com
esforço de engenharia significativo (ver `devin-sessions-db-format` §4); isso limita, na
prática, quais métricas abaixo são "baratas" de implementar hoje vs. quais dependem de uma
decisão explícita do usuário sobre dependências.

## Diferença estrutural chave vs. Claude Code

| Aspecto | Claude Code (`.jsonl`) | Devin CLI |
|---|---|---|
| Fonte sempre-disponível | o próprio `.jsonl` | `sessions.db` (SQLite) |
| Fonte opt-in, mais granular | não existe (tudo já está no `.jsonl`) | transcript ATIF (`--export`) |
| Unidade de custo nativa | USD (`cost-state.totalCostUSD`, já calculado) | **ACU** (Agent Compute Unit) — não há conversão para USD local; taxa é do contrato/conta e não fica salva na máquina |
| Modelo de árvore/rewind | `uuid`/`parentUuid` no `.jsonl` | `message_nodes` como "forest" (fork/revert) no `sessions.db` — colunas exatas de parentesco não confirmadas |
| Granularidade de tokens por turno | sempre presente (`message.usage`) | presente no ATIF (`metadata.metrics`) e possivelmente replicada em `message_nodes.chat_message` |

Isso muda o recorte de MVP: qualquer métrica de **custo em dólar** cai automaticamente na
categoria "Fonte externa" para a Devin CLI — nunca é "Direta" como no Claude, porque a taxa
ACU→USD não existe localmente. Ver `devin-session-invariants`.

## Categorias

Mesma definição usada para o Claude Code — ver skill `claude-session-metrics` se disponível no
seu ambiente, ou:

| Categoria | Definição |
|---|---|
| **Direta** | Lida de um campo/coluna único, sem transformação |
| **Calculada** | Agregação/aritmética determinística sobre vários campos/linhas |
| **Heurística** | Regra determinística com limiar/dicionário escolhido por julgamento — **exponha o limiar** |
| **Estrutural** | Derivada da topologia de `message_nodes` (fork/revert), não de aritmética |
| **Fonte externa** | Determinística só com uma referência fora do arquivo (taxa ACU→USD, tabela de contexto por modelo) |

---

## 1. Métricas DIRETAS

### De `sessions.db.sessions` (sempre disponível)

| Métrica | Origem |
|---|---|
| Título da sessão | `sessions.title` |
| Modelo (fallback de sessão inteira) | `sessions.model` |
| Diretório de trabalho / projeto | `sessions.working_directory` |
| Criada em / última atividade | `sessions.created_at`, `sessions.last_activity_at` |
| Sessão oculta | `sessions.hidden` |
| Custo ACU pré-agregado (se a coluna existir na versão do usuário) | `sessions.metadata.total_acu_cost` |

### De `sessions.db.message_nodes` / ATIF (por turno/step)

| Métrica | Origem |
|---|---|
| Papel da mensagem (`user`/`assistant`/`tool`/`system`) | `message_nodes.chat_message.role` |
| Modelo de geração do step | ATIF `metadata.generation_model` |
| É turno do usuário? | ATIF `metadata.is_user_input` |
| Tokens de entrada/saída/cache do step | ATIF `metadata.metrics.{input_tokens,output_tokens,cache_creation_tokens,cache_read_tokens}` (ou equivalente em `chat_message`, se presente) |
| Custo ACU do step | ATIF `metadata.committed_acu_cost` |
| Timestamp do step | ATIF `metadata.created_at` |
| Nome da função de tool call | ATIF `tool_calls[].function_name` / `chat_message.tool_calls` |
| ID de dedup do turno | ATIF `metadata.request_id` |

---

## 2. Métricas CALCULADAS

### Custo e uso (em ACU — nunca converter para USD sem taxa externa fornecida)

| Métrica | Fórmula |
|---|---|
| ACU total da sessão | `Σ step.metadata.committed_acu_cost` sobre todos os steps do ATIF, deduplicados por `request_id`; **ou** `sessions.metadata.total_acu_cost` se a coluna existir e for confiável — preferir essa quando presente (mesmo espírito do `cost-state` do Claude: fonte já agregada > recálculo manual) |
| Tokens totais (entrada/saída/cache) | `Σ metrics.*` sobre steps deduplicados, filtrando `is_user_input !== true` para "geração do agente" |
| Taxa de acerto de cache | `Σ cache_read_tokens / Σ (cache_read_tokens + cache_creation_tokens + input_tokens)` |
| ACU por step / por token | `total_acu / nº de steps do agente`, `total_acu / total_tokens` |

### Volume da conversa

| Métrica | Fórmula |
|---|---|
| Nº de turnos do usuário vs. do agente | `count(is_user_input == true)` vs. `count(is_user_input == false)`, deduplicados por `request_id` |
| Nº de sessões por diretório de trabalho | `GROUP BY working_directory` em `sessions` |
| Duração de parede da sessão | `last_activity_at - created_at` (de `sessions`) |
| Nº de tool calls | `Σ len(tool_calls)` por step/mensagem |
| Histograma de tool calls por `function_name` | `Counter(function_name)` |

### Multi-sessão (só possível via `sessions.db`, ATIF é por-sessão)

| Métrica | Fórmula |
|---|---|
| Sessões por dia/semana | `GROUP BY date(created_at)` |
| Projetos mais ativos | `GROUP BY working_directory ORDER BY count(*) DESC` |
| Modelo mais usado | `GROUP BY model ORDER BY count(*) DESC` |
| ACU total acumulado (todas as sessões visíveis) | `Σ sessions.metadata.total_acu_cost WHERE hidden = 0` (se a coluna existir) |

---

## 3. Métricas ESTRUTURAIS (árvore de `message_nodes`)

Análogo direto do grafo `parentUuid` do Claude, mas com uma ressalva importante: **as colunas
exatas de parentesco em `message_nodes` não foram confirmadas** por nenhuma fonte consultada.
Antes de implementar qualquer métrica desta seção, rode `PRAGMA table_info(message_nodes)` no
arquivo real do usuário e identifique as colunas de ligação (provavelmente algo como
`parent_id`/`session_id`/`step_index`, mas confirme).

| Métrica | Fórmula (uma vez identificadas as colunas de parentesco) |
|---|---|
| Nº de forks (`/fork`) | nós com mais de um filho |
| Nº de reverts (`/revert`) | ramos abandonados após um ponto de revert — heurística: filho mais novo de um nó tem timestamp muito posterior ao(s) irmão(s) mais antigo(s) |
| Trabalho descartado | nós fora do caminho final da conversa |
| Razão de linearidade | `tamanho do caminho final / total de nós` (1.0 = sem fork/revert) |

## 4. Métricas HEURÍSTICAS

Menos maduras aqui do que na skill equivalente do Claude, porque a granularidade de conteúdo
textual (para detectar frustração, correção, thrash) depende de `chat_message.content`, cujo
formato de texto livre não foi documentado por nenhuma fonte consultada nesta pesquisa. Ao
implementar, trate como **v2**, atrás de validação com dados reais — não hardcode regex/limiares
copiados 1:1 da skill do Claude sem confirmar que o texto de `content` segue um formato
comparável.

| Métrica | Regra proposta (limiar entre `[ ]`, a validar) |
|---|---|
| Taxa de erro de tool call | resultado de `tool` com indicação de erro (campo exato não confirmado) |
| Sessão "travada" | `[≥3]` tool calls idênticas em janela de `[5]` steps |
| Retrabalho por diretório | `[≥N]` sessões distintas no mesmo `working_directory` em `[X horas]` |

---

## 5. Métricas DE FONTE EXTERNA

| Métrica | Fórmula | Referência necessária |
|---|---|---|
| Custo em USD | `total_acu * taxa_acu_usd` | taxa fornecida pelo usuário — **não existe default, não existe fonte local**. Ferramentas de terceiros que fazem essa conversão exigem que o usuário informe a taxa manualmente, sem valor padrão, justamente por isso. |
| % de janela de contexto usada | `tokens_do_step / limite_do_modelo` | catálogo de janelas de contexto por modelo (empacotar no bundle, como no dashboard do Claude) |
| Percentil vs. histórico do usuário | posição desta sessão na distribuição de todas as sessões do `sessions.db` | o próprio `sessions.db` completo (multi-sessão), já disponível localmente — mais fácil que no Claude, que exige ler vários arquivos |

---

## 6. Recomendações para o dashboard (front-end only, sem novas dependências)

- **MVP fica em Direta + Calculada, com ACU (não USD) como unidade de custo.** Mostrar "USD"
  sem taxa configurada pelo usuário seria inventar um número — inaceitável mesmo em MVP.
- **Priorizar ATIF como fonte principal do MVP**, não `sessions.db` — ATIF é JSON puro
  (`JSON.parse` nativo, zero dependências), enquanto ler `sessions.db` no browser sem
  dependência nova exige um parser SQLite escrito à mão (esforço alto) ou aprovação explícita
  para incorporar uma biblioteca de terceiros (dependência nova, precisa de decisão explícita
  do usuário). Ver `devin-sessions-db-format` §4 e `relatorio.md` §2.
- **Sinalizar degradação de dados**: sessão sem ATIF (métricas por-step indisponíveis, e sem
  suporte a `sessions.db` implementado ainda, não há nenhuma métrica disponível — avisar
  claramente na UI em vez de mostrar zeros); ATIF com `schema_version` desconhecida.
- Ver `relatorio.md` para a análise completa de viabilidade técnica de ler `sessions.db`
  (SQLite binário) num app front-end only **sob a restrição de não adicionar dependências**.
