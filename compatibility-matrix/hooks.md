# Hooks

Status: **Copilot CLI ⚠️ reescrita completa · DevIn CLI ✅ reutilizável, com ressalva nos `matcher`.**

## Origem no Claude Code

`.claude/settings.json` → chave `"hooks"`. Eventos em PascalCase (`PreToolUse`, `PostToolUse`,
`UserPromptSubmit`, `Stop`, `SessionStart`, `SessionEnd`, `PostCompaction`, `PermissionRequest`), cada
evento mapeado para uma lista de grupos `{matcher, hooks}`, `matcher` sendo um regex contra o nome da
tool (`Bash`, `Read`, `Edit`...).

## Copilot CLI

**Não é compatível — precisa reescrita, não é rename de campo.** Copilot CLI não lê a chave `"hooks"`
de `.claude/settings.json` diretamente. Os schemas divergem estruturalmente:

| | Claude Code | Copilot CLI |
|---|---|---|
| Estrutura | Um config global, arrays `{matcher, hooks}` por evento | Um arquivo JSON por hook |
| Nome de evento | PascalCase (`PreToolUse`) | camelCase (`preToolUse`) |
| Escopo por tool | `matcher` regex | **Não existe** — todo hook do array roda em toda ocorrência do evento |
| Comando | `command` (shell string) | `bash` + `powershell` separados |
| Localização | `.claude/settings.json` | `.github/hooks/<nome>.json` (projeto) / `~/.copilot/hooks/` (pessoal) |

Mapeamento de eventos: `SessionStart`→`sessionStart`, `SessionEnd`→`sessionEnd`,
`UserPromptSubmit`→`userPromptSubmitted`, `PreToolUse`→`preToolUse`, `PostToolUse`→`postToolUse`,
`Stop`→`agentStop` (mais próximo, mas verificar semântica — "agent termina de responder" no Copilot vs.
"agent quer encerrar o turno" no Claude podem não ser idênticos). `PermissionRequest` e `PostCompaction`
**não têm equivalente documentado** — sinalizar ao usuário em vez de descartar silenciosamente.

Copilot não documenta o contrato de stdout do Claude (`{"decision": "block"}` /
`hookSpecificOutput`/`updatedInput`) — não assumir que é idêntico; verificar contra a doc atual do
Copilot ou testar ao vivo antes de depender de um hook para bloquear ação ou injetar contexto.

### Passos de tradução

1. Para cada `{matcher, hooks}`: se `matcher` casa com toda tool (vazio/omitido), o equivalente Copilot
   é um comando direto por evento. Se mira tools específicas (`"^Bash$"`), o Copilot não tem campo
   `matcher` — o próprio script do hook precisa inspecionar o contexto (se disponível), ou avisar o
   usuário que o escopo não pode ser replicado da mesma forma.
2. Mapear nome do evento (tabela acima).
3. Converter `command` em `bash` (e `powershell`, se suporte a Windows importa — não deixar usuário
   Windows com hook silenciosamente inerte).
4. Criar um `.github/hooks/<nome-descritivo>.json` por hook (ou grupo relacionado), com o wrapper
   `{"version": 1, "hooks": {...}}`.
5. Não portar o contrato de bloqueio/injeção de contexto do Claude assumindo que é idêntico — verificar
   antes de confiar nisso para alterar comportamento do agente.

## DevIn CLI

**Já é compatível, com uma ressalva importante.** DevIn lê a chave `"hooks"` de
`.claude/settings.json` diretamente, usando os mesmos nomes de evento e o mesmo shape JSON
(`matcher`/`hooks` arrays) — é um schema genuinamente compartilhado, não um importador.

**O que NÃO é herdado automaticamente:** regex de `matcher` escritos contra nomes de tool do Claude Code
(`Bash`, `Read`, `Edit`, `Write`, `WebFetch`...) **falham silenciosamente** no DevIn, porque os nomes de
tool são diferentes (`exec`, `read`, `edit`...). Um hook com `"^Bash$"` no Claude Code precisa virar
`"^exec$"` para disparar no DevIn. Este é o erro de porte mais comum — sempre revisar todo `matcher`,
mesmo quando o arquivo "já funciona" porque o DevIn faz parse sem erro.

### Passos de tradução (só necessários se não for confiar no passthrough de `.claude/settings.json`)

1. Criar `.devin/hooks.v1.json` com a mesma estrutura de evento no topo.
2. Reescrever todo `matcher` para os nomes de tool do DevIn.
3. Hooks `command`: scripts geralmente não precisam mudar, já que o JSON de stdin (`tool_name`,
   `tool_input`...) é o mesmo — mas se o script ramifica em cima do valor de `tool_name` (`"Bash"`,
   `"Read"`), essas comparações de string precisam do mesmo remapeamento.
4. Hooks `prompt`: sem mudanças além do nome do evento, já que são avaliados por LLM contra os mesmos
   campos de stdin.
5. Confirmar que hooks se **acumulam** em vez de sobrescrever entre níveis de config (projeto +
   projeto-local + usuário + o passthrough de `.claude/settings.json` disparam juntos) — escrever um
   `.devin/hooks.v1.json` não desativa silenciosamente os de `.claude/settings.json`; ambos rodam a
   menos que os hooks do Claude sejam removidos ou `read_config_from.claude` seja desligado.

### Localizações DevIn

Projeto: `.devin/hooks.v1.json` (recomendado), `.devin/config.json`/`.devin/config.local.json` sob
`"hooks"`. Global: `~/.config/devin/config.json`, `~/.claude.json`, `~/.claude/settings.json`.
</content>
