# Rules (CLAUDE.md)

## Native format (Claude Code)

`CLAUDE.md` at the project root, or `.claude/CLAUDE.md`. Plain markdown, always loaded into the session —
no frontmatter, no trigger conditions. Personal/global version at `~/.claude/CLAUDE.md`.

## From Copilot CLI

Source files: `.github/copilot-instructions.md` (always-on), `.github/instructions/**/*.instructions.md`
(modular, frontmatter `applyTo: "<glob>"` scopes it to matching files, optional `excludeAgent` to hide it
from a specific agent).

### Already compatible?

**No — Claude Code does not read `.github/` instruction files directly.** Unlike the forward direction
(Copilot reads `CLAUDE.md` natively), there is no passthrough the other way. Every Copilot instructions
file needs an explicit `CLAUDE.md` copy.

### Translation steps

1. Copy the content of `.github/copilot-instructions.md` into `CLAUDE.md` (or `.claude/CLAUDE.md`) —
   direct copy, no reformatting needed since both are plain markdown always-on instructions.
2. For `.github/instructions/**/*.instructions.md` files: Claude Code has no native per-path/glob-scoped
   rule mechanism equivalent to `applyTo`. Two options, pick based on what the content actually is:
   - If it's always-relevant guidance regardless of which files are touched, fold it into `CLAUDE.md`
     directly (drop the `applyTo` scoping).
   - If it's genuinely workflow- or path-specific (e.g. "when touching `payments/**`, do X"), it ports
     better as a **Skill** — Claude Code skills are invoked by description-matching or explicit `/name`,
     which is a closer behavioral match to "only relevant sometimes" than a rule. See
     [skills.md](skills.md).
3. Drop `excludeAgent` — Claude Code has no multi-agent-audience concept for rule files; if the excluded
   content was Copilot-specific advice, don't carry it into `CLAUDE.md` at all.
4. Keep the original `.github/` files — Copilot continues reading them independently.

## From DevIn CLI

Source files: `AGENTS.md` (project root, DevIn-native rules), `.devin/rules/<name>.md` (frontmatter
`trigger: always_on | manual | model_decision | agent | glob`, plus `globs: [...]` when `trigger: glob`),
`.devin/global_rules.md`, `~/.config/devin/AGENTS.md` (personal).

### Already compatible?

**No — Claude Code does not read `AGENTS.md` or `.devin/rules/` directly.** DevIn reads Claude's
`CLAUDE.md` (via `read_config_from.claude`), but the reverse import path doesn't exist.

### Translation steps

1. Copy `AGENTS.md` content into `CLAUDE.md` (or `.claude/CLAUDE.md`) — direct copy; both are plain
   always-loaded markdown at the project root, so this is usually a clean 1:1 move.
2. For `.devin/rules/<name>.md` files, branch on `trigger`:
   - `always_on` — fold directly into `CLAUDE.md`, it was already always-relevant.
   - `manual` — Claude Code has no manual-invoke rule primitive; the closest equivalent is a Skill invoked
     explicitly via `/name` (see [skills.md](skills.md)) rather than a rule.
   - `model_decision` or `glob` — these are description/path-scoped, i.e. "sometimes relevant." Port as a
     Skill, not a rule — Claude Code rules are always-on by design, so a scoped DevIn rule folded straight
     into `CLAUDE.md` would over-apply. This mirrors DevIn's own guidance to prefer skills over rules for
     anything workflow-shaped.
   - `agent` — if the rule was scoped to a specific DevIn subagent profile, the Claude Code equivalent is
     putting that guidance in the corresponding `.claude/agents/<name>.md` system prompt instead of
     `CLAUDE.md` (see [subagents.md](subagents.md)).
3. Keep `AGENTS.md` and `.devin/rules/` in place — DevIn keeps reading them regardless of what's added to
   `CLAUDE.md`.

## Cross-cutting notes

- Neither source harness documents a repo-vs-personal precedence identical to Claude Code's — if content
  depends on exact precedence behavior, verify live rather than assuming it carries over.
- This translation is one-directional in effort: going *to* Claude Code from either harness always
  requires a real copy (no passthrough), even though both harnesses read `CLAUDE.md` natively in the
  other direction.
