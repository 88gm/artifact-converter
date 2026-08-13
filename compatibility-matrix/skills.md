# Skills (SKILL.md)

Status: **Copilot CLI ✅ reutilizável (zero movimentação de arquivo) · DevIn CLI ⚠️ sempre precisa port.**

## Origem no Claude Code

`.claude/skills/<nome>/SKILL.md` — diretório nomeado com frontmatter YAML (`name`, `description`,
opcionalmente `allowed-tools`) + corpo em markdown, com subpastas opcionais `scripts/`, `references/`,
`assets/`.

## Copilot CLI

**Já é compatível.** Copilot escaneia `.claude/skills/` nativamente — é uma das três pastas de skill de
projeto que verifica (junto com `.github/skills/` e `.agents/skills/`). Uma skill do Claude Code
geralmente **não precisa mover arquivo nenhum** para ser descoberta pelo Copilot CLI. Subpastas
`scripts/`, `references/`, `assets/` também não precisam mudar — mesma convenção nos dois.

O que realmente difere é a superfície de frontmatter:
- `allowed-tools` do Claude Code usa nomes capitalizados (`Read`, `Edit`, `Bash`...); o vocabulário do
  Copilot é próprio (confirmado: categorias tipo `shell`/`bash`). O vocabulário completo não é
  documentado em detalhe — não inventar uma tabela de mapeamento; sinalizar o campo para o usuário
  confirmar com `/skills info` numa sessão ao vivo.
- Claude Code não tem campo `license`; Copilot não tem `argument-hint`/`subagent`/`agent`/
  `permissions`/`triggers` que outros harnesses adicionam — não copiar esses campos, não têm efeito no
  Copilot.

### Passos de tradução (só frontmatter, não localização)

1. Se a skill já está em `.claude/skills/`, ela já é descoberta pelo Copilot CLI — parar aqui, a menos
   que o usuário queira explicitamente uma cópia em `.github/skills/`.
2. Se for criar cópia Copilot-branded: copiar o diretório para `.github/skills/<nome>/` sem alteração.
3. Frontmatter: copiar `name` e `description` como estão. Deixar `allowed-tools` com nomes MCP
   (`mcp__server__tool`) intactos; sinalizar qualquer tool nativo capitalizado do Claude
   (`Bash`, `Read`, `Edit`, `Grep`, `Glob`, `WebFetch`, `WebSearch`) para o usuário reverificar em vez de
   renomear às cegas.
4. Copiar o corpo em markdown literalmente; só tocar em prosa que nomeia explicitamente uma tool do
   Claude Code (ex.: "use a tool Bash").
5. Validar com `/skills reload` e depois `/skill-name` numa sessão Copilot CLI.

## DevIn CLI

**Não é compatível — sempre precisa de tradução.** DevIn não lê `.claude/skills/` diretamente; não há
caminho de import para skills como existe para regras `CLAUDE.md`. Toda skill do Claude Code precisa de
uma cópia explícita para DevIn.

Boa notícia: o formato-núcleo (diretório com `SKILL.md` + YAML + corpo markdown, subpastas opcionais
`scripts/`, `references/`, `assets/`) é a mesma convenção nos dois — só frontmatter e localização mudam.

### Localização nativa

Projeto: `.devin/skills/<nome>/SKILL.md` (também aceito em `.windsurf/skills/` ou `.agents/skills/`).
Global: `~/.config/devin/skills/<nome>/SKILL.md` (`%APPDATA%\devin\skills\` no Windows).

### Passos de tradução

1. Criar `.devin/skills/<nome>/SKILL.md` espelhando a estrutura original — copiar `scripts/`,
   `references/`, `assets/` sem alteração.
2. Mapear frontmatter:
   - `name`, `description` — copiar como estão.
   - `allowed-tools` — remapear nomes capitalizados do Claude Code para minúsculos do DevIn:
     `Read`→`read`, `Edit`→`edit`, `Grep`→`grep`, `Glob`→`glob`, `Bash`→`exec`. Nomes MCP
     (`mcp__server__tool`) não mudam. `WebFetch`/`WebSearch` não têm equivalente documentado no DevIn —
     sinalizar ao usuário em vez de inventar mapeamento.
   - Claude Code não tem `argument-hint`, `subagent`, `agent`, `permissions`, `triggers` — são
     capacidades novas do DevIn, adicionar só se agregarem valor (ex.: `triggers: [user]` para uma skill
     destrutiva que não deveria disparar sozinha).
3. Copiar o corpo literalmente; só ajustar prosa que nomeia tools explicitamente (ex.: "use a tool Bash"
   → "use a tool exec").
4. Se a skill original delega para um subagent (`Task`/`Agent` do Claude Code), mapear para
   `subagent: true` ou `agent: <perfil>` no frontmatter do DevIn em vez de instrução em prosa — DevIn
   espera isso declarado, não narrado.
5. Validar invocando `/nome` numa sessão DevIn CLI.

### Campo exclusivo do DevIn: `triggers`

Não existe equivalente no Claude Code — controla se a skill pode ser invocada autonomamente pelo modelo
(`model`) além do comando explícito (`user`). Ao traduzir uma skill do Claude que nunca deveria disparar
sozinha, considerar `triggers: [user]` — isso é comportamento novo a decidir, não uma tradução literal.
</content>
