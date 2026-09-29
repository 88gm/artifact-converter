# Dossiê: Framework de Desenvolvimento Personalizado para Devin
### Orquestrador único, Adaptive Workflow com Router, Multiagentes em Paralelo e Conceito de State (estilo AWS AI-DLC)

Data da pesquisa: 2026-09-29
Fontes: documentação oficial `docs.devin.ai` (ver seção **Referências** no final)

---

## 0. Correção de premissa importante

A premissa do pedido era "no Devin só existe hooks, agent.md e skills". A pesquisa na documentação oficial mostra que isso **não é verdade** — o Devin atual (CLI + Cloud + API) expõe um conjunto de primitivos muito mais rico, e é justamente a combinação deles que resolve os 4 requisitos pedidos:

| Primitivo | O que é | Onde vive |
|---|---|---|
| `AGENTS.md` | Contexto/instruções sempre-on do repositório (16 KiB carregados automaticamente) | raiz do repo |
| `SKILL.md` (Skills) | Procedimentos reutilizáveis, formato aberto "Agent Skills" | `.agents/skills/<nome>/SKILL.md` |
| Hooks | Comandos shell/LLM disparados em eventos do ciclo de vida do agente | `.devin/hooks.v1.json` |
| **Playbooks** | Prompts/procedimentos reutilizáveis geridos no app web, anexáveis via macro (`!nome`), com escopo Org/Enterprise/System | Settings → Playbooks |
| **Knowledge** (em depreciação → migrando para Skills/Plugins) | Dicas/instruções recuperadas por relevância (RAG-like), não carregadas todas de uma vez | Settings → Knowledge |
| **Subagents** (Devin CLI) | Agentes-trabalhadores independentes, foreground ou background, com perfis customizáveis | `agents/*.md` (projeto ou global) |
| **Dynamic Workflows** | Script Python **determinístico** que orquestra um time de sessões Devin (`agent()`, `pipeline()`, `parallel()`, cache por hash de prompt) | script Python + Devin Cloud API |
| **Sessões gerenciadas em paralelo** (Advanced Capabilities) | Uma sessão "coordenadora" delega pedaços de um trabalho grande para N sessões Devin isoladas em VMs próprias | Devin MCP / API |
| **Devin API v3** | `POST /organizations/{org_id}/sessions` com `tags`, `playbook_id`, `knowledge_ids`, `structured_output_schema`, `session_links`, `resumable` | `api.devin.ai/v3` |
| **MCP** | Devin pode ser cliente (conectar ferramentas externas) e servidor (`Devin MCP`, expõe "Use Devin Sessions") | `.devin/mcp.json` / Devin MCP server |
| **Blueprints / Environment Management** | Definição declarativa do ambiente/VM (imagem, setup, snapshots) usada em toda sessão | Enterprise → Environment Management |

Ou seja: o framework pedido pelo usuário **já tem quase todos os blocos de construção nativos no Devin** — o trabalho de "criar um AI framework personalizado" é principalmente um trabalho de **arquitetura/composição** desses primitivos, não de inventar mecanismos que não existem.

---

## 1. Agente Orquestrador como único ponto de entrada

### Objetivo
Nenhuma outra "porta de entrada" deve executar trabalho diretamente — tudo passa por um orquestrador que classifica, roteia e delega.

### Como implementar no Devin

Existem duas formas de materializar isso, e elas **se complementam** (produção vs. interativo):

**A) Orquestrador conversacional (uso interativo, dentro do Devin CLI/app)**
- Crie um `AGENTS.md` na raiz do repositório com uma seção obrigatória do tipo *"Papel do Agente"*, instruindo explicitamente: *"Você é um orquestrador. Nunca implemente diretamente. Classifique o pedido, decida a rota (ver Skill `router`), e delegue via subagents ou dynamic workflow."*
- Crie uma Skill `.agents/skills/orchestrator/SKILL.md` com `triggers: user` (evita auto-ativação indesejada) e argument-hint, que contém a rubrica de classificação (ver seção 2) e instruções de "nunca resolver, sempre delegar".
- Use um hook `SessionStart` (`.devin/hooks.v1.json`) para injetar sempre o mesmo contexto de "modo orquestrador" no início de toda sessão, garantindo que nenhuma sessão comece "sem chapéu de orquestrador" mesmo que alguém esqueça de mencionar a skill.
- Use um hook `UserPromptSubmit` para interceptar todo prompt do usuário e anexar metadados de roteamento (ex.: farejar palavras-chave de "bugfix", "migration", "research") antes que o agente decida.

**B) Orquestrador programático (uso automatizado/CI, produção)**
- O **verdadeiro** ponto de entrada único, em produção, deve ser um **Dynamic Workflow** (script Python) — ele é literalmente descrito na doc como "*um script Python determinístico que orquestra um time de agentes Devin*". Esse script:
  - É a única função/CLI invocada externamente (`devin workflow run entrypoint.py --prompt "..."` ou via API/CI).
  - Nunca deixa um chamador falar direto com uma sessão Devin "crua" — toda sessão nasce dentro de uma chamada `agent(...)` controlada pelo script.
  - Centraliza logging, `session_links` e tags de rastreabilidade.

**Recomendação prática:** trate (A) como a "camada de UX" para humanos usando Devin interativamente, e (B) como o "backend" real do framework, para automações e CI/CD. O `AGENTS.md` + Skill do orquestrador interativo deve, inclusive, poder **chamar o mesmo Dynamic Workflow** por trás quando o pedido exigir múltiplas sessões — reaproveitando a mesma lógica de roteamento sem duplicá-la.

---

## 2. Adaptive Workflow com Router

### Conceito
Um "router" decide, a partir do prompt, quais etapas (steps) ideais executar — não é um fluxo fixo, é decidido dinamicamente.

### Como o Devin já suporta isso nativamente
Dynamic Workflows foram desenhados exatamente para isso. Primitivos chave:

```python
# entrypoint.py — ponto de entrada único do framework
from devin_workflow import register_workflow, agent, pipeline, parallel, log

register_workflow(
    name="adaptive-dev-workflow",
    description="Router + execução adaptativa de tarefas de engenharia",
    phases=["route", "plan", "execute", "verify", "report"],
)

ROUTES = {
    "bugfix":    {"playbook_id": "pb_bugfix",    "parallel": False},
    "feature":   {"playbook_id": "pb_feature",   "parallel": True},
    "migration": {"playbook_id": "pb_migration", "parallel": True},
    "research":  {"playbook_id": "pb_research",  "parallel": False},
    "review":    {"playbook_id": "pb_review",    "parallel": False},
}

async def route(user_prompt: str) -> dict:
    log(f"Roteando prompt: {user_prompt[:80]}")
    # Chamada de classificação — barata, com schema estruturado
    decision = await agent(
        prompt=f"Classifique esta tarefa em uma das rotas {list(ROUTES)} "
               f"e estime a complexidade (low/med/high) e se é paralelizável: {user_prompt}",
        phase="route",
        schema={"route": "string", "complexity": "string", "parallelizable": "boolean"},
    )
    return decision

async def run(user_prompt: str):
    decision = await route(user_prompt)
    route_cfg = ROUTES[decision["route"]]

    if decision["complexity"] == "high" and route_cfg["parallel"]:
        # Adaptativo: decompõe em N subtarefas e roda em paralelo
        subtasks = await agent(
            prompt=f"Decomponha em subtarefas independentes: {user_prompt}",
            phase="plan",
            schema={"subtasks": "array"},
        )
        results = await parallel([
            agent(prompt=t, phase="execute", schema={"status": "string", "branch": "string"})
            for t in subtasks["subtasks"]
        ])
    else:
        # Rota simples: um único agente cuida de tudo
        results = await agent(
            prompt=user_prompt,
            phase="execute",
            schema={"status": "string", "branch": "string"},
        )

    report = await agent(
        prompt=f"Gere relatório final a partir de: {results}",
        phase="report",
        schema={"summary": "string"},
    )
    return report
```

### Pontos-chave da doc que sustentam o "adaptive"
- **Decisões são código Python puro** (`if`/`else`, `async def`) que leem os resultados estruturados (`schema=`) das etapas anteriores — não há limitação de "fluxo fixo".
- **Determinismo obrigatório**: a lógica do workflow e os prompts **não podem depender** de hora atual, aleatoriedade, IDs gerados, variáveis de ambiente, filesystem ou respostas de rede fora do controle do workflow. Isso é o que garante que o roteamento seja **replayable** e **auditável**.
- **Cache por hash de prompt**: cada chamada `agent()` é indexada por hash do prompt + schema + configurações de execução. Retomar um workflow interrompido reaproveita instantaneamente etapas já concluídas e só reexecuta o que falta — isso já é, na prática, uma forma de *state* (ver seção 4).

### Complemento no nível de sessão única (sem Dynamic Workflow)
Quando o roteamento precisa acontecer **dentro** de uma única sessão Devin interativa (não via script externo), use:
- Uma Skill "router" (`.agents/skills/router/SKILL.md`) que contém a rubrica de classificação e instrui o agente a, dependendo da rota, invocar Playbooks diferentes (`!bugfix-playbook`, `!migration-playbook` etc.) ou delegar a subagents específicos.
- Um hook `UserPromptSubmit` para pré-classificar e injetar `additionalContext` (ex.: "Esta tarefa parece ser uma migração — considere paralelizar por pacote") antes mesmo do agente formular seu plano.

---

## 3. Arquitetura multiagêntica (subagents em paralelo)

Há **duas camadas** de paralelismo no Devin, e o framework deve escolher a camada certa por rota:

### Camada 1 — Subagents dentro de uma única sessão (Devin CLI)
- Subagents são "trabalhadores" independentes que **não herdam o histórico do pai**, mas compartilham ferramentas e contexto de código.
- Dois modos:
  - **Foreground**: roda inline, pai pausa e espera; você aprova/nega tool calls.
  - **Background**: roda em paralelo ao pai; pai é notificado automaticamente ao concluir; tool calls não aprovadas são automaticamente negadas.
- Perfis nativos: `subagent_explore` (somente leitura, busca web) e `subagent_general` (acesso total, pode alterar código).
- **Perfis customizados**: arquivos markdown em `agents/` (projeto ou global) com frontmatter YAML definindo system prompt, restrição de ferramentas e `model:` (override de modelo).
- Controles: `subagents_enabled` em `config.json`; profundidade máxima de aninhamento é **zero por padrão** (só o agente raiz pode gerar subagents) — perfis customizados podem elevar isso via `max-nesting`.
- Custo: cada subagent consome créditos/contexto independentes — em planos por prompt, cada subagent = 1 cobrança adicional; aninhamento multiplica custo.

**Uso recomendado**: tarefas confinadas a **um repositório/uma sessão** — ex. "explore autenticação em paralelo enquanto implemento o endpoint", revisão de código concorrente à implementação, exploração de múltiplas hipóteses de bug.

### Camada 2 — Sessões paralelas em VMs isoladas (multi-sessão, cross-repo)
- Um "coordenador" (sessão Devin ou o próprio Dynamic Workflow) decompõe uma tarefa grande e delega pedaços para **N sessões Devin gerenciadas**, cada uma em sua **própria VM isolada**.
- O coordenador: define o escopo de cada pedaço, monitora progresso, resolve conflitos entre workstreams paralelos, compila e integra resultados; pode lançar sessões-filhas com prompt/playbook/tags/limites de recurso específicos, enviar instruções de acompanhamento, rastrear consumo de ACU por sessão, pausar/terminar sessões de baixo desempenho.
- Acesso via **Devin MCP** com a permissão "Use Devin Sessions" (incluída por padrão em `org_member`/`org_admin`).
- Exemplos documentados: migração de 50+ arquivos agrupados em pacotes independentes, cada pacote com sua sessão e playbook padronizado; ou análise de código identificando N módulos sem cobertura de teste, cada um gerando sua própria sessão + PR.
- **Handoff de código entre sessões em VMs separadas** acontece via **git branches**: cada sessão empurra sua branch e reporta o nome no output estruturado; etapas seguintes leem essas branches. (Sessões que compartilham a mesma VM compartilham a working tree direto, mas exigem salvaguardas de isolamento para escritores paralelos.)

**Uso recomendado**: tarefas que atravessam **muitos arquivos, módulos ou repositórios** — exatamente o caso de "migration" e "feature" grandes no router da seção 2.

### Como o router decide qual camada usar
Regra prática a codificar na etapa `route()` do Dynamic Workflow:

| Sinal do router | Camada escolhida |
|---|---|
| Tarefa cabe em 1 repo, complexidade baixa/média | Sessão única, sem subagents |
| Tarefa cabe em 1 repo, mas se beneficia de exploração paralela (ex. pesquisa + implementação simultânea) | Subagents (Camada 1), background mode |
| Tarefa cross-file/cross-module/cross-repo, decomponível em N pacotes independentes | `parallel([agent(...) for ...])` no Dynamic Workflow → N sessões em VMs isoladas (Camada 2) |
| Tarefa monolítica que só pode ser paralelizada em fases sequenciais dependentes | `pipeline(items, stage1, stage2, ...)` (paralelismo por pipeline, sem barreira de sincronização forçada) |

---

## 4. Conceito de State (equivalente ao AWS AI-DLC)

### O que o AI-DLC da AWS tem que queremos replicar
O AI-DLC organiza o trabalho em **fases** (Inception → Construction → Operations), com **unidades de trabalho** que carregam **estado explícito** através das fases, checkpoints humanos ("mob elaboration") e rastreabilidade de qual fase cada unidade está.

### O que o Devin já oferece nativamente (mais perto do que parece)
1. **Fases nomeadas no Dynamic Workflow**: `register_workflow(phases=["route", "plan", "execute", "verify", "report"])` — cada `agent()` é etiquetado com `phase=`, dando rastreabilidade nativa de "em que fase da esteira este resultado nasceu".
2. **Cache determinístico por hash** = *state store* implícito: cada etapa é resumível — um workflow interrompido no meio retoma reaproveitando etapas concluídas (cache hit) e refazendo só o resto. Isso é, na prática, uma máquina de estados persistente e transparente, sem você precisar construir nada extra.
3. **API v3 de sessões** expõe primitivos de estado explícito por sessão:
   - `resumable: true` — permite pausar/retomar uma sessão.
   - `session_links` — permite modelar relações pai/filho entre sessões (unidade de trabalho → subunidades).
   - `tags` — permite marcar cada sessão com fase/rota/id de unidade de trabalho.
   - `structured_output_required` + `structured_output_schema` — força que cada sessão retorne um objeto de estado tipado e validável (equivalente ao "contrato de saída" de uma unidade de trabalho no AI-DLC).
4. **Blueprints / Environment Management** (Enterprise) — snapshots declarativos do ambiente garantem que o "estado do mundo" (imagem de VM, setup) seja reproduzível entre fases/sessões, análogo a fixar o ambiente de uma unidade de trabalho do AI-DLC.

### O que falta nativamente e precisa ser construído por você
O Devin **não tem** um painel/objeto de "unidade de trabalho com estado visível e editável" como o AI-DLC. Para emular isso de forma equivalente:

**a) State Store externo e versionado**
- Mantenha um arquivo `workflow_state.json` (ou uma tabela em um DB leve) **committado no repositório** ou em um serviço à parte, com uma entrada por "unidade de trabalho":
  ```json
  {
    "unit_id": "feat-2026-09-29-001",
    "phase": "execute",
    "route": "feature",
    "status": "in_progress",
    "sessions": [
      {"session_id": "sess_abc", "phase": "plan", "branch": "wip/feat-001-plan"},
      {"session_id": "sess_def", "phase": "execute", "branch": "wip/feat-001-impl"}
    ],
    "history": ["route", "plan", "execute"]
  }
  ```
- Cada chamada `agent()` no Dynamic Workflow lê o estado atual antes de rodar e escreve o novo estado depois (via `structured_output_schema` retornando o próximo estado), replicando a transição de fase do AI-DLC.

**b) Hooks para persistir e auditar transições de estado**
- `SessionStart`: carrega o `workflow_state.json` relevante e injeta como contexto (*"você está retomando a unidade feat-2026-09-29-001 na fase execute"*).
- `SessionEnd` / `Stop`: persiste o novo estado (fase concluída, branch produzida, status) de volta no state store, com o `session_id` e `prompt_id` estáveis fornecidos pelo próprio payload do hook para rastreabilidade.
- `PostCompaction`: reinjeta o resumo do estado que poderia ter sido perdido na compactação de contexto — importante em sessões longas multi-fase.

**c) Checkpoints humanos (equivalente ao "mob elaboration" do AI-DLC)**
- Use `bypass_approval: false` na criação de sessões críticas (API v3) para forçar aprovação humana antes de fases sensíveis (ex. antes de `execute` em produção).
- Combine com hook `PermissionRequest` para gates automáticos (ex. bloquear automaticamente qualquer tentativa de `execute` sem que a fase `plan` tenha sido marcada como `approved` no state store).

### Resumo da equivalência

| Conceito AI-DLC | Equivalente no Devin |
|---|---|
| Fase (Inception/Construction/Ops) | `phase=` em `agent()` + fases nomeadas em `register_workflow` |
| Unidade de trabalho | Registro no state store externo (`unit_id`), amarrado via `tags`/`session_links` |
| Estado persistente entre fases | Cache por hash de prompt (nativo) + `workflow_state.json` (construído por você) |
| Retomada de trabalho interrompido | `resumable: true` + replay de cache do Dynamic Workflow |
| Checkpoint humano / mob elaboration | `bypass_approval` + hook `PermissionRequest` + gates no state store |
| Contrato de saída da unidade | `structured_output_schema` |

---

## 5. Estrutura de repositório proposta

```
meu-projeto/
├── AGENTS.md                          # papel de orquestrador, regras sempre-on
├── .devin/
│   ├── hooks.v1.json                  # SessionStart, UserPromptSubmit, Stop, PermissionRequest...
│   └── mcp.json                       # conexão com Devin MCP / ferramentas externas
├── .agents/
│   └── skills/
│       ├── orchestrator/SKILL.md      # "nunca resolva direto, delegue"
│       ├── router/SKILL.md            # rubrica de classificação de rota
│       └── <skills específicas>/SKILL.md
├── agents/                             # perfis customizados de subagents (CLI)
│   ├── explorer.md
│   └── implementer.md
├── workflows/
│   ├── entrypoint.py                  # Dynamic Workflow = ponto de entrada único real
│   └── routes/
│       ├── bugfix.py
│       ├── feature.py
│       └── migration.py
└── state/
    └── workflow_state.json            # (ou aponta para DB externo) — state store das unidades de trabalho
```

---

## 6. Roadmap de implementação sugerido

1. **Semana 1** — `AGENTS.md` + Skill `orchestrator` + Skill `router` com rubrica simples (regras determinísticas, sem LLM ainda). Validar que sessões interativas nunca "pulam" o roteamento.
2. **Semana 2** — Primeiro `entrypoint.py` com Dynamic Workflow: `register_workflow` + `route()` com classificação por LLM + schema. Rodar 2-3 rotas simples sem paralelismo.
3. **Semana 3** — Introduzir `parallel()`/`pipeline()` para a rota `feature`/`migration`; testar handoff via git branches entre sessões em VMs separadas.
4. **Semana 4** — Subagents customizados (`agents/*.md`) para exploração paralela dentro de sessão única; comparar custo/benefício vs. sessões separadas.
5. **Semana 5** — State store externo (`workflow_state.json`) + hooks `SessionStart`/`SessionEnd`/`PermissionRequest` para persistência e checkpoints humanos.
6. **Semana 6** — Hardening: `bypass_approval` em fases sensíveis, tags/`session_links` para auditoria completa, dashboards a partir do state store.

---

## 7. Riscos e limitações a considerar

- **Custo**: subagents e sessões paralelas multiplicam consumo de créditos/ACU rapidamente — o router deve ter um "orçamento" (ex. limite de sessões paralelas por unidade de trabalho) codificado explicitamente.
- **Skills: apenas uma ativa por vez** (documentado) e composição sequencial de skills ainda "em progresso" segundo a doc — não desenhe o roteamento assumindo múltiplas skills encadeadas automaticamente; prefira que o Dynamic Workflow (Python) faça a orquestração de alto nível, e as Skills fiquem para procedimentos pontuais dentro de uma sessão.
- **Knowledge está sendo depreciado** em favor de Skills/Plugins — não construa nenhuma parte crítica do framework sobre Knowledge; se já existir, planeje migração.
- **Determinismo é um requisito rígido dos Dynamic Workflows** — nada de `datetime.now()`, `random`, IDs gerados ou chamadas de rede fora de controle dentro da lógica de roteamento, sob pena de perder a resumibilidade (o "state" gratuito que a seção 4 descreve).
- **Nesting de subagents é zero por padrão** — se o design depender de subagents que geram subagents, isso precisa ser explicitamente habilitado via `max-nesting` em perfis customizados, e o custo cresce multiplicativamente.

---

## 8. Referências (documentação oficial consultada)

- [Devin Skills](https://docs.devin.ai/product-guides/skills)
- [Devin CLI Skills — overview](https://docs.devin.ai/cli/extensibility/skills/overview)
- [Devin CLI Skills — creating skills](https://docs.devin.ai/cli/extensibility/skills/creating-skills)
- [Creating Playbooks](https://docs.devin.ai/product-guides/creating-playbooks)
- [Using Playbooks](https://docs.devin.ai/product-guides/using-playbooks)
- [AGENTS.md](https://docs.devin.ai/onboard-devin/agents-md)
- [CLI Extensibility — Rules](https://docs.devin.ai/cli/extensibility/rules)
- [Knowledge (product guide)](https://docs.devin.ai/product-guides/knowledge)
- [Knowledge onboarding](https://docs.devin.ai/onboard-devin/knowledge-onboarding)
- [Hooks — Overview](https://docs.devin.ai/cli/extensibility/hooks/overview)
- [Hooks — Lifecycle Hooks](https://docs.devin.ai/cli/extensibility/hooks/lifecycle-hooks)
- [Dynamic Workflows](https://docs.devin.ai/work-with-devin/dynamic-workflows)
- [Advanced Capabilities (sessões paralelas)](https://docs.devin.ai/work-with-devin/advanced-capabilities)
- [CLI Subagents](https://docs.devin.ai/cli/subagents)
- [MCP — Connect Devin to external tools](https://docs.devin.ai/work-with-devin/mcp)
- [Devin MCP server](https://docs.devin.ai/work-with-devin/devin-mcp)
- [Devin API — Create Session (v3)](https://docs.devin.ai/api-reference/v3/sessions/post-organizations-sessions)
- [Devin API — Create Session (v1)](https://docs.devin.ai/api-reference/v1/sessions/create-a-new-devin-session)
- [Devin API — List Sessions](https://docs.devin.ai/api-reference/sessions/list-sessions)
- [Devin API — Overview](https://docs.devin.ai/api-reference/overview)
- [Environment / Blueprints](https://docs.devin.ai/onboard-devin/environment/blueprints)
- [Enterprise — Environment Management Overview](https://docs.devin.ai/enterprise/environment-management/overview)
