# Rules (CLAUDE.md / AGENTS.md / copilot-instructions.md)

Status: **✅ Reutilizável em ambos os harnesses, sem tradução na maioria dos casos.**

## Origem no Claude Code

- `CLAUDE.md` na raiz do projeto, `.claude/CLAUDE.md`.
- Markdown puro, sempre carregado na sessão.

## Copilot CLI

**Já é compatível.** Copilot lê `CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md` e `GEMINI.md`
diretamente como arquivos de instrução "agent-specific", nas mesmas localizações-padrão onde procura seu
próprio `copilot-instructions.md`. Um projeto que já tem `CLAUDE.md` não precisa de nenhuma tradução.

Ressalvas:
- Copilot combina todas as instruções aplicáveis (projeto + pessoal) e remove duplicatas exatas, mas
  **não define hierarquia de precedência** entre elas — não assumir que repo > pessoal.
- `@relative/path` dentro de um `CLAUDE.md` funciona (não é `GEMINI.md` nem `*.instructions.md`), mas
  confirmar se caminhos relativos resolvem a partir da raiz que o Copilot considera — não é garantido
  ser idêntico à resolução do Claude Code.
- Mudanças em arquivos de instrução exigem reiniciar/retomar a sessão para valer.

### Quando criar versão nativa (`.github/copilot-instructions.md`)

Só se o time quiser padronizar em Copilot ou parar de depender do import. Passos:
1. Copiar o conteúdo de `CLAUDE.md` para `.github/copilot-instructions.md` (cópia direta, não reescrita).
2. Se o `CLAUDE.md` mistura conteúdo sempre-relevante com orientação situacional (ex.: "ao mexer no
   módulo de pagamentos, faça X"), considerar dividir a parte situacional em
   `.github/instructions/payments.instructions.md` com `applyTo: "**/payments/**"`.
3. Manter o `CLAUDE.md` original — Claude Code e Copilot continuam lendo ambos.
4. Para um arquivo que só o Copilot deve ler (Claude Code deve ignorar), usar `excludeAgent` num
   `*.instructions.md` modular em vez de duplicar conteúdo contraditório.

## DevIn CLI

**Já é compatível.** DevIn lê `CLAUDE.md` diretamente como regra de projeto, e `~/.claude/CLAUDE.md`
como regra global — controlado por `~/.config/devin/config.json`:

```json
{"read_config_from": {"agents_standard": true, "cursor": true, "windsurf": true, "claude": true}}
```

Geralmente ligado por padrão. Se estiver ligado, nada precisa ser feito para um `CLAUDE.md` existente
funcionar sob DevIn.

DevIn recomenda usar **Skills** em vez de Rules sempre que possível ("Rules should stay minimal") —
regras devem ser pequenas e sempre-relevantes (convenções de nomenclatura, "nunca commitar segredos");
qualquer coisa em formato de workflow pertence a uma skill (ver [skills.md](skills.md)).

### Quando criar versão nativa (`AGENTS.md`)

Só se o usuário quiser um arquivo DevIn-nativo (padronizar time, ou se `read_config_from.claude`
estiver desligado). Passos:
1. Copiar `CLAUDE.md` para `AGENTS.md` na raiz — cópia direta na maioria dos casos.
2. Dividir orientação situacional em `.devin/rules/<nome>.md` com frontmatter `trigger: glob` +
   `globs: [...]` — regras nessa pasta não são always-on por padrão (`trigger` pode ser `always_on`,
   `manual`, `model_decision`, `agent` ou `glob`).
3. Manter `CLAUDE.md` original — ambos continuam lendo.

## Cuidados cruzados

- Precedência: no DevIn, `.devin/` tem prioridade sobre `.windsurf/` quando ambos existem; regras de
  projeto e globais carregam simultaneamente (aditivo, não either/or).
- Nenhum dos dois harnesses documenta uma hierarquia repo-vs-pessoal idêntica à do Claude Code —
  verificar comportamento se o conteúdo depende de precedência exata.
</content>
