# Subagents / Custom Agents

Status: **⚠️ Precisa port nos dois harnesses — nenhum lê `.claude/agents/` diretamente.**

## Origem no Claude Code

`.claude/agents/<nome>.md` — markdown com frontmatter YAML (`name`, `description`, `model`, `tools`) +
corpo como system prompt.

## Copilot CLI

**Não é compatível.** Sem passthrough para `.claude/agents/`. Precisa de **mover + renomear**:
`.claude/agents/<nome>.md` → `.github/agents/<nome>.agent.md` (projeto) ou
`~/.copilot/agents/<nome>.agent.md` (pessoal — este último **vence** em colisão de nome com a versão de
projeto, então não criar os dois sem intenção).

A extensão `.agent.md` é obrigatória — não é um `.md` simples.

### Mapeamento de frontmatter

- `name`, `description` — copiar como estão.
- `tools` (Claude Code) → `tools` (Copilot) — **mesmo nome de campo**, mas verificar o vocabulário de
  valores: Claude Code usa nomes capitalizados (`Read`, `Edit`, `Bash`...); o vocabulário do Copilot
  para este campo não é totalmente documentado além de exemplos como `shell`. Não inventar tabela de
  mapeamento completa — copiar entradas MCP inequívocas (`mcp__server__tool`, estáveis entre harnesses)
  e sinalizar o resto para o usuário confirmar (`/agent` picker ou docs).
- `model` do Claude Code **não tem equivalente documentado** em custom agent do Copilot — omitir e
  avisar que seleção de modelo por agente pode não ser configurável da mesma forma.

### Passos de tradução

1. Criar `.github/agents/<nome>.agent.md` (projeto) ou `~/.copilot/agents/<nome>.agent.md` (pessoal).
2. Aplicar mapeamento de frontmatter acima.
3. Copiar o corpo do system prompt literalmente, ajustando nomes de tool citados na prosa.
4. Se outros artefatos referenciam esse subagent pelo nome, garantir que o `name` do lado Copilot bate
   exatamente, e atualizar a sintaxe de invocação para a do Copilot (`/agent`, nome explícito, ou
   `--agent nome`) — a invocação via tool `Agent` do Claude Code não é herdada.
5. Validar com `/agent` numa sessão Copilot CLI, ou `copilot --agent <nome> --prompt "test"` no shell.

## DevIn CLI

**Não é compatível.** Sem passthrough para `.claude/agents/` — mesma situação do Copilot, mas com
localização e schema próprios.

Localização nativa: `.devin/agents/<nome>.md` (formato plano) ou `.devin/agents/<nome>/AGENT.md` (forma
de diretório, útil se o subagent precisa de recursos empacotados junto). Também reconhecido em
`.agents/agents/`. Global: `~/.config/devin/agents/` (`%APPDATA%\devin\agents\` no Windows).

### Mapeamento de frontmatter

- `name`, `description` — copiar como estão.
- `model` — copiar como está (nomes de modelo geralmente são vocabulário compartilhado, mas verificar
  se a string específica é reconhecida pelo DevIn em caso de dúvida).
- `tools` (Claude Code) → `allowed-tools` (DevIn) — **renomear o campo** e remapear os valores com a
  mesma tabela minúscula das skills: `Read`→`read`, `Edit`→`edit`, `Grep`→`grep`, `Glob`→`glob`,
  `Bash`→`exec`; nomes MCP inalterados.
- Claude Code não tem `max-nesting` — só adicionar se o prompt do subagent delega explicitamente para
  outros subagents e isso for intencional (DevIn não suporta nesting além de um nível por padrão).

### Passos de tradução

1. Criar `.devin/agents/<nome>.md` (ou a forma `<nome>/AGENT.md` se precisar de arquivos empacotados).
2. Aplicar mapeamento de frontmatter acima.
3. Copiar o corpo do system prompt literalmente, ajustando nomes de tool na prosa.
4. Se outras skills/rules do projeto invocam esse subagent pelo nome (ex.: uma skill com
   `agent: reviewer`), garantir que o `name` do lado DevIn bate exatamente com essas referências.

## Diferença-chave entre os dois destinos

| | Copilot CLI | DevIn CLI |
|---|---|---|
| Extensão de arquivo | `.agent.md` (obrigatória) | `.md` plano ou `AGENT.md` em subpasta |
| Campo de tools | `tools` (mesmo nome, vocabulário incerto) | `allowed-tools` (nome muda, valores minúsculos mapeados) |
| `model` | Sem equivalente documentado | Campo `model` existe, copiar direto |
| Nesting de subagents | Não documentado | `max-nesting` explícito, default 1 nível |
</content>
