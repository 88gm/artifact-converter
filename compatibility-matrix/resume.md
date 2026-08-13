# Compatibilidade de Artefatos: Claude Code ↔ Copilot CLI ↔ DevIn CLI

Mapa consolidado de quais artefatos do harness do Claude Code (skills, rules, hooks, subagents,
plugins, MCP) funcionam sem alteração em GitHub Copilot CLI e DevIn CLI, e quais exigem port.
Fonte: `.claude/skills/copilot-cli-artifacts/` e `.claude/skills/devin-cli-artifacts/` deste repo.

Arquivos detalhados por artefato: [rules.md](rules.md) · [skills.md](skills.md) · [hooks.md](hooks.md) ·
[subagents.md](subagents.md) · [plugins-mcp.md](plugins-mcp.md)

## Tabela-resumo

| Artefato | Origem (Claude Code) | Copilot CLI | DevIn CLI |
|---|---|---|---|
| **Rules** | `CLAUDE.md`, `.claude/CLAUDE.md` | ✅ **Reutilizável** — lê `CLAUDE.md`/`AGENTS.md`/`GEMINI.md` direto, sem tradução | ✅ **Reutilizável** — lê `CLAUDE.md` direto via `read_config_from.claude` (ligado por padrão) |
| **Skills** | `.claude/skills/<nome>/SKILL.md` | ✅ **Reutilizável** — escaneia `.claude/skills/` nativamente (uma das 3 pastas de projeto) | ⚠️ **Precisa port** — sem passthrough; copiar para `.devin/skills/` e remapear `allowed-tools` |
| **Hooks** | `.claude/settings.json` → `"hooks"` | ⚠️ **Precisa port (reescrita)** — schema estruturalmente diferente (por evento, `bash`/`powershell`, sem `matcher`) | ✅ **Reutilizável, com ressalva** — lê `.claude/settings.json` direto, mas os `matcher` regex (`Bash`, `Read`...) não batem com os nomes de tool do DevIn (`exec`, `read`...) e falham silenciosamente |
| **Subagents** | `.claude/agents/<nome>.md` | ⚠️ **Precisa port** — mover + renomear para `.github/agents/<nome>.agent.md`, sem passthrough | ⚠️ **Precisa port** — copiar para `.devin/agents/<nome>.md`, sem passthrough |
| **Plugins** | `.claude-plugin/plugin.json` | ⚠️ **Precisa port** — sem passthrough documentado; manifesto precisa de `plugin.json` na raiz (não em subpasta) | ✅ **Reutilizável** — DevIn lê `.claude-plugin/plugin.json` nativamente, lado a lado com seu próprio formato |
| **MCP** | `.mcp.json` (`mcpServers`) | ✅ **Quase reutilizável** — mesmo schema-base; só falta campo `type` explícito (`local`/`http`/`sse`) | ✅ **Quase reutilizável** — copiar entradas para `.devin/mcp_config.json` (objeto direto, sem wrapper `mcpServers`) |

## Leitura rápida por harness

**Copilot CLI é mais generoso com Rules, Skills e MCP** (leitura nativa ou quase-cópia direta), mas
trata **Hooks e Subagents como reescrita completa** — schemas divergem de verdade, não é só rename de
campo.

**DevIn CLI é mais generoso com Rules, Hooks, Plugins e MCP** (lê os arquivos do Claude Code
literalmente, inclusive `.claude-plugin/plugin.json`), mas **Skills e Subagents sempre precisam de
cópia traduzida** — não existe passthrough para esses dois tipos.

## Regra geral de decisão

1. Antes de portar qualquer artefato, checar a seção "Já é compatível?" do arquivo correspondente —
   várias traduções são desnecessárias.
2. Preservar o arquivo original do Claude Code; a tradução é aditiva (arquivo novo ao lado), nunca
   substitutiva — outros harnesses continuam lendo o `.claude/...` original.
3. Quando existe passthrough mas com uma ressalva pontual (hooks no DevIn, MCP em ambos), resolver
   apenas o ponto de atrito citado — não recriar o artefato inteiro do zero.
4. Nomes de tools (`Read`/`Edit`/`Bash`/`Grep`/`Glob`/`WebFetch`/`WebSearch`) são o detalhe que mais
   quebra silenciosamente entre harnesses — sempre revisar `matcher`/`allowed-tools`/`tools` mesmo
   quando o restante do arquivo "já funciona".
</content>
