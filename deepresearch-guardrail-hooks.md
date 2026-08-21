# Deep Research: Hooks de Guard-Rail para Agentes de IA (Claude CLI, Copilot CLI, DevIn CLI)

> Pesquisa de mercado + guia de implementação para construir, **do zero**, hooks determinísticos que protegem sessões de agentes de IA contra: (1) loops de erro, (2) payloads gigantes, (3) excesso de histórico, e (4) roteamento ruim de modelo/agente.
>
> Contexto de uso: ambiente corporativo estritamente seguro → **dependência mínima ou zero de bibliotecas externas**. As três ferramentas-alvo já expõem um mecanismo de hooks nativo; a estratégia recomendada é reusar esse mecanismo em vez de construir um proxy/gateway próprio.

---

## 1. Por que hooks e não apenas "instruções no prompt"

Instrução em `CLAUDE.md`/`AGENTS.md` é probabilística: o modelo pode ignorá-la sob pressão de contexto longo ou objetivo conflitante. Um **hook é código determinístico** que roda fora do modelo, em pontos fixos do ciclo de vida da ferramenta (antes/depois de uma tool call, no início/fim de sessão, antes de compactar contexto). Isso é o consenso de mercado atual:

> "Hooks like PreToolUse provide deterministic guardrails that can actively block or approve actions during execution... rules are guidance, hooks are enforcement." — padrão citado tanto na doc do DevIn quanto em análises independentes de guardrails para Claude Code/Copilot.

Regra prática adotada neste documento: **tudo que precisa ser garantido (não apenas sugerido) vira hook; tudo que é preferência de estilo continua em CLAUDE.md/AGENTS.md/instruções**.

---

## 2. O mecanismo de hooks nas três ferramentas (referência oficial)

As três CLIs convergiram para o mesmo desenho: eventos nomeados → processo externo recebe **JSON via stdin** → responde via **stdout (JSON) + exit code**. Isso é uma ótima notícia: um guard-rail pode ser escrito uma vez (ex.: em Python puro ou Node puro, ambos sem dependência externa) e adaptado por evento a cada ferramenta.

### 2.1 Claude Code

- Config: `.claude/settings.json` (projeto), `.claude/settings.local.json` (local, não versionado), `~/.claude/settings.json` (usuário).
- ~30 eventos. Os relevantes para este projeto:
  - `PreToolUse` — antes de qualquer tool call; pode **bloquear** (`permissionDecision: deny`), **reescrever input** (`updatedInput`), ou **escalar** para aprovação humana (`escalate`).
  - `PostToolUse` — depois que a tool rodou; não pode mais impedir a execução, mas pode **injetar `additionalContext`** e mostrar `systemMessage`. É o ponto certo para truncar payloads grandes.
  - `PostToolUseFailure` — depois que uma tool falhou; ponto certo para contar erros repetidos.
  - `PreCompact` / `PostCompact` — antes/depois da compactação automática de histórico; ponto certo para logar/auditar a compactação ou interceptar histórico antes de ele ser resumido.
  - `Stop` — quando Claude terminaria a resposta; pode retornar `continue: false` para forçar parada (útil como "kill switch" de loop).
  - `UserPromptSubmit` — antes de processar o prompt do usuário; bom lugar para um **roteador** (ex.: decidir se a tarefa é trivial e sinalizar isso via `additionalContext`).
- Exit codes: `0` = sucesso (stdout é parseado como JSON se começar com `{`); `2` = bloqueio (para eventos que suportam bloqueio: `PreToolUse`, `UserPromptSubmit`, `Stop`, `PreCompact`); outros códigos = erro não bloqueante (ação prossegue, mensagem some no transcript).
- Matcher: por nome de tool (`Bash`, `Edit|Write`, regex, `mcp__servidor__.*`).
- Timeout padrão de hook `command`: 600s (30s para `UserPromptSubmit`).

Fonte: [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks).

### 2.2 GitHub Copilot CLI

- Config: `.github/hooks/*.json` (repositório, aplica a todo uso de agentes Copilot no repo) ou `~/.copilot/hooks/*.json` (pessoal, todo uso da CLI). Também existe camada de política admin em `/etc/github-copilot/policy.d/*.json`, que roda antes de tudo e não pode ser desativada.
- Formato do hook: `{"type": "command", "bash": "...", "powershell": "...", "command": "fallback cross-platform", "timeoutSec": 30, "matcher": "regex"}` — **já nativo para Windows** via chave `powershell`, o que é diretamente relevante já que o ambiente do usuário é Windows 11.
- Eventos relevantes: `preToolUse`, `postToolUse`, `postToolUseFailure`, `userPromptSubmitted`, `sessionStart`, `sessionEnd`, `errorOccurred`, `preCompact`, `permissionRequest`.
- `preToolUse` retorna `permissionDecision: allow|deny|ask` + `modifiedArgs`.
- `postToolUse` retorna `modifiedResult` (pode **reescrever o resultado da tool**, ideal para truncar payload) + `additionalContext`.
- Exit codes: **assimétrico e importante** — para `preToolUse`/`permissionRequest`, exit 2 ou qualquer erro/crash é **fail-closed** (bloqueia por segurança); para a maioria dos outros eventos é **fail-open** (ignora erro e segue); **timeout é sempre fail-open em qualquer evento**, inclusive `preToolUse` — ou seja, um hook travado NÃO impede a ação, então hooks de segurança devem ser rápidos e não podem depender de uma chamada de rede lenta.

Fonte: [GitHub Copilot hooks reference](https://docs.github.com/en/copilot/reference/hooks-reference).

### 2.3 DevIn CLI

- Config: `.devin/hooks.v1.json` no projeto (o próprio arquivo já é o objeto de hooks, sem wrapper), ou aninhado sob a chave `"hooks"` em `.devin/config.json`.
- Eventos: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `UserPromptSubmit`, `Stop`, `PostCompaction`, `SessionStart`, `SessionEnd`.
- Saída: `{"decision": "block", "reason": "..."}` para bloquear; `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "updatedInput": {...}}}` para reescrever argumentos; `additionalContext` para injetar texto.
- Exit codes: `0` ok, `2` bloqueia, outro = erro não bloqueante.
- `AGENTS.md` (na raiz, `.devin/rules/*.md`, ou `.devin/global_rules.md`) continua valendo para instruções não determinísticas — mas **não substitui hooks** para os quatro problemas deste projeto.

Fonte: [Devin CLI Hooks Reference](https://docs.devin.ai/cli/extensibility/hooks/overview).

### 2.4 Tabela-resumo de compatibilidade

| Conceito | Claude Code | Copilot CLI | DevIn CLI |
|---|---|---|---|
| Formato de I/O | JSON stdin → JSON stdout + exit code | idem | idem |
| Bloquear tool call | `PreToolUse` + exit 2 ou `permissionDecision: deny` | `preToolUse` + `permissionDecision: deny` | `PreToolUse` + `decision: block` |
| Reescrever args da tool | `updatedInput` | `modifiedArgs` | `updatedInput` (dentro de `hookSpecificOutput`) |
| Reescrever resultado da tool | `additionalContext` (PostToolUse não sobrescreve o resultado bruto, só adiciona contexto) | `modifiedResult` (sobrescreve de fato) | `additionalContext` |
| Ponto de hook em falha de tool | `PostToolUseFailure` | `postToolUseFailure` | *(tratar dentro de `PostToolUse` verificando status)* |
| Hook em compactação | `PreCompact`/`PostCompact` | `preCompact` | `PostCompaction` |
| Fail-open vs fail-closed | depende do evento (ver acima) | assimétrico, documentado explicitamente | não documentado — tratar como fail-open por padrão e testar |
| Suporte nativo Windows/PowerShell | script chamado via shell padrão do SO | chave `powershell` dedicada | script chamado via shell padrão do SO |

**Implicação de design:** dá para manter **um único script de lógica por guard-rail** (ex. `loop_guard.py`) e um **adaptador fino por ferramenta** que só traduz o schema de entrada/saída. Isso evita duplicar a lógica de detecção em 3 lugares.

---

## 3. Princípio de "zero/mínima dependência externa"

Ambiente corporativo estritamente seguro normalmente proíbe ou dificulta:
- instalar pacotes via `pip`/`npm`/`choco` em runtime;
- baixar binários de terceiros (`jq`, `yq`, etc.) — **`jq` não vem por padrão no Windows**, e muitos exemplos de mercado (inclusive a doc oficial do Claude Code) usam `jq`, o que **não deve ser copiado literalmente** neste ambiente;
- rodar processos que fazem chamadas de rede para SaaS de terceiros (LLM-as-judge de guardrail, serviços tipo Invariant/Guardrails AI/NeMo Guardrails) — a maioria das soluções "prontas" de mercado (NeMo Guardrails, Guardrails AI, Invariant, LangChain callbacks) depende de bibliotecas Python pesadas e, em vários casos, de um LLM adicional para julgar o conteúdo. Isso é exatamente o que o usuário quer evitar.

**Decisão de arquitetura recomendada:** escrever os hooks em **linguagem já presente no runtime**, sem pacotes de terceiros:

| Linguagem | Por que serve | Cuidado |
|---|---|---|
| **Python 3** (`json`, `sys`, `re`, `hashlib`, `pathlib`, `sqlite3` — tudo stdlib) | Já costuma estar disponível em máquinas dev; stdlib cobre 100% do que estes hooks precisam (parse JSON, regex, hash, fila persistida em SQLite/arquivo) | usar `#!/usr/bin/env python3` e testar caminho do interpretador no Windows (`py` vs `python`) |
| **PowerShell 7+** (`ConvertFrom-Json`, `ConvertTo-Json`, `Get-Content`) | Nativo no Windows, zero instalação, e é literalmente a chave `powershell` do Copilot CLI | evitar `Invoke-WebRequest` para qualquer endpoint externo dentro do hook |
| **Node.js** (`JSON.parse`, `fs`, `crypto`) | Se o time já tem Node no PATH (comum em times de front/full-stack) | idem, sem `npm install` |

Recomendação final: **PowerShell para os hooks "leves" (regex, contagem, truncamento) porque é 100% nativo no Windows do usuário**, e **Python para os hooks com estado persistente** (ex.: contador de loop entre chamadas, cache de histórico) porque manipular JSON/SQLite em PowerShell é mais verboso. Ambos evitam pacotes externos.

Evitar explicitamente:
- `jq` (não nativo no Windows);
- frameworks de guardrail de mercado (NeMo Guardrails, Guardrails AI, Invariant, LlamaFirewall) — bons como **referência de padrão de projeto**, não como dependência;
- qualquer hook que faça uma chamada de API para "julgar" com outro LLM se algo é seguro — isso adiciona latência, custo, superfície de rede e uma dependência externa contrária ao requisito.

---

## 4. Guard-rail 1 — Loop de erro (mesma ferramenta falhando repetidamente)

### O problema, segundo a pesquisa de mercado
- Um agente entra em loop quando repete a mesma tool call (mesmo `tool_name` + `tool_input` ou mesmo padrão de erro) sem progresso. Sintoma clássico em produção: spans repetidos, custo de token subindo, sem mudança de estado.
- Causa raiz mais citada: **a tool não devolve um sinal de sucesso/falha claro**, então o modelo não sabe que já tentou aquilo e tenta de novo.
- Padrão de mercado consolidado ("no-progress guard"): fazer **hash da tupla `(tool, argumentos normalizados, resultado/erro)`** das últimas N chamadas e, se a mesma tupla aparecer **2-3 vezes seguidas**, interromper — não esperar até um limite alto tipo 20-25 passos.

### Design proposto (determinístico, sem LLM extra)

1. **Estado**: arquivo JSON local por sessão, ex. `.claude/state/loop-guard-<session_id>.json` (ou `%TEMP%` se preferir não versionar), contendo uma lista circular das últimas N tuplas `{tool_name, input_hash, outcome, timestamp}`.
2. **Hook `PostToolUse` / `PostToolUseFailure`** (Claude/Copilot) ou `PostToolUse` (DevIn):
   - normaliza `tool_input` (remove campos voláteis tipo timestamp) e calcula hash (`sha256` da stdlib);
   - registra `{hash, sucesso/falha}` no estado;
   - conta repetições consecutivas do mesmo hash com o mesmo resultado (falha).
3. **Gatilho**: 3 repetições idênticas consecutivas com falha → no próximo `PreToolUse` que bater o mesmo hash, **bloquear** (`deny`/`decision: block`) com mensagem clara: *"Esta chamada já falhou 3x de forma idêntica. Pare e reavalie a abordagem antes de tentar de novo."* Isso empurra o modelo a mudar de estratégia em vez de travar a sessão via `Stop`.
4. **Escalonamento opcional**: se o mesmo hash aparecer uma 4ª vez (ou seja, o modelo ignorou a mensagem de erro do bloqueio anterior de outra forma), o hook `Stop`/`SubagentStop` retorna `continue: false` como kill-switch final.
5. **Reset de estado**: limpar o arquivo em `SessionStart`/`sessionStart`/`SessionStart` (todas as 3 ferramentas têm esse evento) para não vazar estado entre sessões.

### Por que isso é melhor que "contar N tentativas totais"
Contar tentativas totais também pune tentativas legítimas de retry com input diferente (ex.: tentar 5 comandos de build diferentes não é loop). O diferencial é comparar a **tupla completa**, não só o nome da tool.

### Pseudocódigo (PowerShell, sem dependências)

```powershell
# loop-guard-pretooluse.ps1
$input_json = [Console]::In.ReadToEnd() | ConvertFrom-Json
$toolName   = $input_json.tool_name
$toolInput  = $input_json.tool_input | ConvertTo-Json -Compress
$hash       = [BitConverter]::ToString(
                [System.Security.Cryptography.SHA256]::Create().ComputeHash(
                  [Text.Encoding]::UTF8.GetBytes("$toolName|$toolInput")
                )
              ) -replace '-',''

$stateFile = Join-Path $env:TEMP "loop-guard-$($input_json.session_id).json"
$state = if (Test-Path $stateFile) { Get-Content $stateFile -Raw | ConvertFrom-Json } else { @{ failures = @{} } }

$count = [int]($state.failures.$hash)
if ($count -ge 3) {
  @{ hookSpecificOutput = @{
       hookEventName = "PreToolUse"
       permissionDecision = "deny"
       permissionDecisionReason = "Chamada idêntica falhou $count vezes seguidas. Troque de abordagem."
     } } | ConvertTo-Json -Depth 5
  exit 0
}
exit 0
```

(O incremento do contador acontece no hook `PostToolUseFailure`/equivalente, de forma análoga, e é zerado quando a mesma tupla tem sucesso.)

Fontes de padrão: [Invariant Labs — Loop Detection](https://explorer.invariantlabs.ai/docs/guardrails/loops/), [Particula — Stop AI Agents Looping](https://particula.tech/blog/stop-ai-agents-looping-same-tool-call-no-progress), [FutureAGI — Infinite Loop glossary](https://futureagi.com/glossary/infinite-loop/), [MatrixTrak — Loop guardrails](https://matrixtrak.com/blog/agents-loop-forever-how-to-stop).

---

## 5. Guard-rail 2 — Payload gigante (retorno de tool estoura o contexto)

### O problema, segundo a pesquisa de mercado
- Tool que devolve conteúdo de arquivo, log ou resultado de busca sem limite pode sozinha consumir a maior parte da janela de contexto.
- Prática de mercado: **truncar na origem**, não depois. Exemplos citados: Trae Agent trunca cada resposta de tool nos primeiros 16 KB; outras implementações usam ~10 KB por resposta como teto.
- Truncamento ingênuo (cortar e pronto) pode remover a parte relevante — por isso a técnica mais robusta combina truncamento com um **"ponteiro" para o conteúdo completo em disco**, para que o agente possa pedir mais se precisar (padrão "Memory Pointer" / "file-based pointer offloading", citado inclusive em produtos como Deep Agents da LangChain).

### Design proposto

1. **Hook `PostToolUse`**: mede o tamanho de `tool_result` (Claude/DevIn) ou `toolResult.textResultForLlm` (Copilot).
2. Se exceder um limite configurável (ex. **8.000 caracteres**, ajustável por tipo de tool — `Read`/`Bash` podem ter limites diferentes de `Grep`):
   - grava o conteúdo completo em `.claude/state/payload-cache/<hash>.txt` (fora do contexto);
   - devolve ao modelo apenas os primeiros N caracteres + últimos M caracteres (cabeça e cauda costumam ser mais informativos que só a cabeça, especialmente em logs onde o erro está no final) + uma nota: *"Saída truncada (12.400 → 8.000 chars). Conteúdo completo salvo em `<path>`. Use Read/Grep nesse arquivo se precisar do restante."*
   - via `additionalContext` (Claude/DevIn) ou `modifiedResult` (Copilot, que de fato substitui o resultado — a opção mais limpa nesta ferramenta).
3. **Sinalização de custo**: opcionalmente, logar em um arquivo `.claude/state/metrics.jsonl` cada truncamento, para depois auditar quais tools/quais tarefas mais geram payload grande — dado útil para revisar prompts/instruções da equipe.

### Por que não usar apenas o parâmetro nativo de "output limit" de cada ferramenta
Algumas CLIs já truncam automaticamente (ex. Bash tool do Claude Code tem limite de output), mas isso é **por tool individual e não configurável globalmente por política corporativa**, e não gera o ponteiro para recuperação — por isso um hook central complementa, aplicando a mesma política a **todas** as tools, inclusive MCP de terceiros que não truncam nada.

Fontes: [arXiv 2511.22729 — Solving Context Window Overflow in AI Agents](https://arxiv.org/html/2511.22729v1), [AWS DEV.to — AI Context Window Overflow: Memory Pointer Fix](https://dev.to/aws/ai-context-window-overflow-memory-pointer-fix-3akc), [arXiv 2602.07092 — Lemon Agent (compressão hierárquica em 3 camadas)](https://arxiv.org/pdf/2602.07092).

---

## 6. Guard-rail 3 — Excesso de histórico (conversas longas encarecendo requisições)

### O problema, segundo a pesquisa de mercado
- Cada nova requisição reenvia o histórico inteiro; sem gestão, custo cresce quadraticamente ao longo de uma sessão longa.
- Padrão de produção mais citado (2026): **pipeline combinando as 4 técnicas**, na ordem de agressividade crescente:
  1. **Collapse de tool-calls antigas** (gentle) — substitui o corpo de tool results antigos (não os mais recentes) por um resumo de uma linha;
  2. **Sumarização de trechos antigos da conversa** (moderate) — LLM resume blocos antigos de turnos;
  3. **Janela deslizante mantendo só os últimos N turnos de usuário** (aggressive);
  4. **Drop de grupos mais antigos como último recurso** (emergency backstop), se ainda estourar o orçamento.
- Regra de disparo citada: monitorar quando a sessão cruza **~80-85% da janela do modelo** e, a partir daí, truncar/resumir automaticamente (ou escalar para um modelo com janela maior).

### O que já existe nativamente
Claude Code já tem compactação automática (evento `PreCompact`/`PostCompact`); Copilot tem `preCompact`; DevIn tem `PostCompaction`. Ou seja, **a ferramenta já faz sumarização de histórico por padrão** — o guard-rail aqui não é reinventar a compactação, é:
1. **Auditar/logar** quando compactação acontece e quanto contexto foi liberado (hook `PostCompact`/`PostCompaction`), para ter visibilidade de custo;
2. **Aplicar o collapse de tool-calls antigas *antes* de a compactação nativa precisar entrar em ação** — via `PostToolUse`, sempre que uma nova tool call acontecer, verificar se há resultados de tools de N passos atrás (ex. mais de 20 tool calls no passado) que ainda estão "crus" no contexto e substituí-los proativamente por uma versão resumida de 1-2 linhas + o ponteiro do Guard-rail 2. Isso é o mesmo mecanismo do payload gigante, só que disparado por **idade**, não por **tamanho**.
3. **Bloquear em `PreCompact`/`preCompact`** apenas se você quiser controle manual total (ex. exigir que um humano aprove a compactação em sessões de auditoria) — não recomendado por padrão, pois adiciona fricção sem ganho de segurança real.

### Design proposto (complementar ao nativo, não substituto)
- Hook leve em `PostToolUse`: mantém um contador de "quantas tool calls atrás" cada resultado está. Quando um resultado passa da idade configurada (ex. 20 chamadas), ele já foi coberto pelo Guard-rail 2 (truncamento por tamanho) — então este guard-rail é essencialmente **Guard-rail 2 com gatilho por idade em vez de por tamanho**, reusando a mesma função de "gravar em disco + devolver ponteiro".
- Hook em `PostCompact`/`PostCompaction`: grava métrica `{session_id, tokens_antes, tokens_depois, timestamp}` em `.claude/state/metrics.jsonl` — visibilidade de custo sem nenhuma dependência externa (nem telemetria de terceiros).

Fontes: [Microsoft Learn — Compaction (Agent Framework)](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/compaction), [Augment Code — AI Agent Loop Token Costs](https://www.augmentcode.com/guides/ai-agent-loop-token-cost-context-constraints), [ExplainX — Conversation History Management for AI Agents 2026](https://explainx.ai/blog/conversation-history-management-ai-agents-2026), [GetMaxim — Context Window Management Strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/).

---

## 7. Guard-rail 4 — Roteamento ruim (agente caro para pergunta trivial)

### O problema, segundo a pesquisa de mercado
- Ferramentas de LLM routing de mercado (RouteLLM, NeuralTrust, Requesty, Martian) usam classificadores treinados ou um modelo pequeno "tentando primeiro" para decidir se escala para um modelo caro. Ganhos citados: até 85% de redução de custo mantendo ~95% da qualidade do modelo grande.
- Isso normalmente depende de infraestrutura própria (proxy de API, classificador treinado) — **inviável e desnecessário** dentro do escopo "hook local, sem dependência externa" pedido aqui, porque Claude CLI/Copilot CLI/DevIn CLI não expõem troca de modelo por requisição individual da mesma forma que uma API direta.

### O que É viável como hook local, sem ML e sem dependência externa
Regra baseada em **heurística determinística** (regex + comprimento + palavras-chave), não em classificador treinado — o padrão "rules-based routing" citado nas pesquisas como abordagem válida quando não se quer treinar/hospedar um modelo:

1. **Hook `UserPromptSubmit`** (Claude/DevIn) ou `userPromptSubmitted` (Copilot):
   - Aplica heurísticas simples sobre o prompt do usuário e sobre o `permission_mode`/tipo de agente que está prestes a ser invocado:
     - prompt muito curto (< N palavras) **e** sem menção a arquivos/código/comandos → provável pergunta trivial;
     - prompt bate em uma lista de padrões conhecidos como triviais (saudação, "o que é X", "explique Y em uma frase") vs. padrões que indicam tarefa complexa (menção a múltiplos arquivos, "refatore", "implemente", stack trace colado, etc.);
   - Se o prompt parece trivial mas o usuário está prestes a acionar um subagente pesado/orquestração multi-agente (ex. o `Workflow`/`Agent` tool deste próprio ambinete, ou um "planner" caro em outra ferramenta), o hook **não pode trocar o modelo por conta própria** (isso é decisão do host), mas pode:
     - injetar `additionalContext`: *"Este prompt parece simples; considere responder diretamente em vez de acionar um subagente."* — um "nudge" determinístico e barato, sem custo de token de um LLM julgador;
     - ou, em ferramentas que suportam `permissionDecision: deny`/`ask` em `PreToolUse` para a *tool de orquestração específica* (ex. bloquear a chamada a `Agent`/`Task`/subagente pesado quando o prompt de origem é trivial e pedir confirmação humana) — isso sim é enforcement real, não apenas sugestão.
2. **Hook `PreToolUse` com matcher no nome da tool de orquestração** (ex. `Agent`, `Workflow`, `Task`, ou o nome do subagente caro): antes de permitir a chamada, checa um sinal barato (tamanho do prompt original, se já existe um plano, se é a primeira tool call da sessão) e, se o sinal indicar "tarefa trivial acionando ferramenta cara", retorna `escalate`/`ask` em vez de `deny` — dá controle ao humano sem travar automação legítima.

### Por que não implementar um "classificador" de fato
Um classificador de complexidade minimamente confiável é ML (mesmo que leve, tipo regressão logística) e precisaria de dataset rotulado + retraining — isso é dependência de manutenção incompatível com "hook simples, sem lib externa, ambiente corporativo travado". A heurística regex/palavra-chave é o que a própria pesquisa de mercado chama de "rules-based routing", reconhecida como estratégia legítima (não é gambiarra), só que com recall menor que ML — trade-off aceitável dado o requisito do usuário.

Fontes: [NeuralTrust — LLM Model Routing](https://neuraltrust.ai/blog/llm-model-routing), [Decagon — What is an LLM Router](https://decagon.ai/glossary/what-is-an-llm-router), [Requesty — Intelligent LLM Routing in Enterprise AI](https://www.requesty.ai/blog/intelligent-llm-routing-in-enterprise-ai-uptime-cost-efficiency-and-model), [Braintrust — Best LLM routers 2026](https://www.braintrust.dev/articles/best-llm-routers-2026).

---

## 8. Soluções prontas de mercado (referência, não para instalar)

| Solução | O que faz | Por que **não** usar diretamente aqui |
|---|---|---|
| **NeMo Guardrails** (NVIDIA) | DSL (Colang) para regras de conversação/segurança | Dependência Python pesada, motor próprio, overkill para 4 regras determinísticas |
| **Guardrails AI** | Validação estruturada de output de LLM (schemas, PII, etc.) | Foco em validação de *conteúdo* gerado, não em loop/payload/histórico/routing de agente; pacote externo |
| **Invariant / Explorer (Invariant Labs)** | Guardrails de trace de agente, incluindo detecção de loop | Bom como **referência de algoritmo** (seção 4); é um produto/serviço externo |
| **agent-guardrails (roboticforce, GitHub)** | Hooks prontos para bloquear comandos destrutivos (terraform, k8s, db) em Claude Code | Bom como referência de *padrão de hook de segurança de comando*; não cobre os 4 casos deste projeto — mas vale olhar a estrutura do repo como exemplo de organização de `.claude/hooks/` |
| **RouteLLM / Martian / NeuralTrust** | Roteamento de modelo via proxy/classificador | Requer infraestrutura de proxy de API própria; não se aplica dentro do modelo de hooks locais das 3 CLIs-alvo |
| **LangChain callbacks / Deep Agents (LangChain)** | Context engineering, memory pointer pattern | Referência conceitual excelente (seção 5-6); é um framework Python completo, não um hook isolado |

Conclusão da pesquisa: **nenhuma solução de mercado se encaixa "out of the box"** no requisito (zero dependência, dentro do mecanismo nativo de hooks de 3 CLIs específicas). A rota correta é implementar os 4 guard-rails como scripts pequenos e independentes, usando os padrões de mercado só como referência de algoritmo — que é exatamente a abordagem detalhada nas seções 4-7.

---

## 9. Plano de implementação sugerido (passo a passo)

1. **Fase 0 — Esqueleto comum**
   - Criar `hooks/common/` com funções puras (Python stdlib) reaproveitáveis por todos os guard-rails: `read_stdin_json()`, `write_stdout_json()`, `hash_tool_call()`, `state_path(session_id)`.
   - Criar `hooks/adapters/` com um adaptador por ferramenta (`claude/`, `copilot/`, `devin/`) que só faz o "de-para" de schema (nomes de campo) e chama a lógica comum.
2. **Fase 1 — Guard-rail 1 (loop de erro)**: menor risco, maior valor imediato. Testar manualmente forçando um comando que falha repetidamente.
3. **Fase 2 — Guard-rail 2 (payload gigante)**: aplicar primeiro em tools de leitura de arquivo/bash, medir com um caso real (ex. `cat` de um log grande).
4. **Fase 3 — Guard-rail 3 (histórico)**: começar só com o hook de auditoria em `PostCompact`/`PostCompaction` (observabilidade, zero risco), depois avaliar se o collapse proativo (idade) é necessário além da compactação nativa.
5. **Fase 4 — Guard-rail 4 (roteamento)**: começar apenas com o `additionalContext` de "nudge" (sem bloquear nada), validar em uso real antes de evoluir para `escalate`/`ask` em tools de orquestração específicas.
6. **Fase 5 — Rollout multi-ferramenta**: validar cada guard-rail isoladamente no Claude CLI primeiro (documentação mais madura), depois portar para Copilot CLI (atenção ao fail-open/fail-closed assimétrico) e DevIn CLI (schema mais enxuto, sem `modifiedResult`).

### Convenções recomendadas de projeto
- Um diretório de estado por guard-rail, sempre com `session_id` no nome do arquivo, sempre limpo em `SessionStart`.
- Limites (tamanho de payload, nº de repetições, idade de tool call) em **um único arquivo de config** (`hooks/config.json`) lido por todos os adaptadores — facilita auditoria de política de segurança sem tocar em código.
- Logar toda decisão de bloqueio/truncamento em `hooks/logs/*.jsonl` (append-only, sem envio externo) para revisão posterior pelo time de segurança.

---

## 10. Fontes consultadas

- [Claude Code Hooks Reference (oficial)](https://code.claude.com/docs/en/hooks)
- [Claude Code Hooks (2026) — morphllm](https://www.morphllm.com/claude-code-hooks)
- [Claude Code Hooks: 6 Production Patterns — Pixelmojo](https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns)
- [GitHub Copilot hooks reference (oficial)](https://docs.github.com/en/copilot/reference/hooks-reference)
- [Using hooks with GitHub Copilot CLI (oficial)](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/use-hooks)
- [Devin CLI — Hooks Reference (oficial)](https://docs.devin.ai/cli/extensibility/hooks/overview)
- [Devin CLI — Rules & AGENTS.md (oficial)](https://docs.devin.ai/cli/extensibility/rules)
- [Zarar.dev — Don't rely on instructions, use Agent Hooks](https://zarar.dev/agent-hooks-deterministic-guardrails-for-ai-generated-code/)
- [RanTheBuilder — Agentic Coding Hooks: Deterministic AI Guardrails](https://ranthebuilder.cloud/blog/agentic-coding-hooks-deterministic-ai-guardrails/)
- [FutureAGI — What Is an Infinite Loop (Agent Failure)?](https://futureagi.com/glossary/infinite-loop/)
- [Invariant Labs — Loop Detection docs](https://explorer.invariantlabs.ai/docs/guardrails/loops/)
- [DEV.to (AWS) — How to Prevent AI Agent Reasoning Loops from Wasting Tokens](https://dev.to/aws/how-to-prevent-ai-agent-reasoning-loops-from-wasting-tokens-2652)
- [MatrixTrak — How to Stop AI Agents from Looping Forever](https://matrixtrak.com/blog/agents-loop-forever-how-to-stop)
- [Particula — Stop AI Agents Looping on the Same Failed Tool Call](https://particula.tech/blog/stop-ai-agents-looping-same-tool-call-no-progress)
- [arXiv 2511.22729 — Solving Context Window Overflow in AI Agents](https://arxiv.org/html/2511.22729v1)
- [DEV.to (AWS) — AI Context Window Overflow: Memory Pointer Fix](https://dev.to/aws/ai-context-window-overflow-memory-pointer-fix-3akc)
- [arXiv 2509.23586 — Reducing Cost of LLM Agents with Trajectory Reduction](https://arxiv.org/pdf/2509.23586)
- [Agenta — Top 6 techniques to manage context length in LLMs](https://agenta.ai/blog/top-6-techniques-to-manage-context-length-in-llms)
- [LangChain — Context Management for Deep Agents](https://www.langchain.com/blog/context-management-for-deepagents)
- [arXiv 2602.07092 — Lemon Agent Technical Report](https://arxiv.org/pdf/2602.07092)
- [Microsoft Learn — Compaction (Agent Framework)](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/compaction)
- [Augment Code — AI Agent Loop Token Costs](https://www.augmentcode.com/guides/ai-agent-loop-token-cost-context-constraints)
- [ExplainX — Conversation History Management for AI Agents: 2026 Guide](https://explainx.ai/blog/conversation-history-management-ai-agents-2026)
- [GetMaxim — Context Window Management Strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/)
- [NeuralTrust — LLM Model Routing](https://neuraltrust.ai/blog/llm-model-routing)
- [Decagon — What is an LLM Router?](https://decagon.ai/glossary/what-is-an-llm-router)
- [Requesty — Intelligent LLM Routing in Enterprise AI](https://www.requesty.ai/blog/intelligent-llm-routing-in-enterprise-ai-uptime-cost-efficiency-and-model)
- [Braintrust — Best LLM routers and model routing platforms in 2026](https://www.braintrust.dev/articles/best-llm-routers-2026)
- [Fiddler AI — AI Coding Agent Security: Threat Models and Controls](https://www.fiddler.ai/blog/ai-coding-agent-security)
- [GitHub — roboticforce/agent-guardrails](https://github.com/roboticforce/agent-guardrails)
- [Galileo — 8 Best AI Agent Guardrails Solutions in 2026](https://galileo.ai/blog/best-ai-agent-guardrails-solutions)
- [DEV.to — AI Coding Agent Security: Practical Guardrails for Claude Code, Copilot, and Codex](https://dev.to/maxkrivich/ai-coding-agent-security-practical-guardrails-for-claude-code-copilot-and-codex-och)

---

## 11. Próximos passos sugeridos

- Confirmar com o time de segurança se scripts em Python/PowerShell locais (sem instalar pacotes) são aceitáveis, ou se há restrição adicional (ex. só PowerShell assinado digitalmente).
- Decidir o limite numérico de cada guard-rail (nº de repetições, tamanho de payload, idade de tool call, tamanho de janela de histórico) — os valores neste documento são pontos de partida da pesquisa de mercado, não requisitos fixos.
- Começar a implementação pela Fase 1 (loop de erro) no Claude CLI, por ser o ambiente de desenvolvimento atual e ter a documentação de hooks mais completa das três ferramentas.
