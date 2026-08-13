# Instructions & Rules

## Native format

Plain markdown. Copilot CLI combines every applicable file into context rather than picking one winner.

| File | Scope | Notes |
|---|---|---|
| `.github/copilot-instructions.md` | Project, repo-wide | Primary repo instructions file. |
| `.github/instructions/**/*.instructions.md` | Project, path-specific | Modular; requires `applyTo` frontmatter. |
| `~/.copilot/copilot-instructions.md` | Personal, all projects | User-level primary instructions. |
| `~/.copilot/instructions/**/*.instructions.md` | Personal, path-specific | Modular personal instructions. |
| `AGENTS.md`, `CLAUDE.md`, `.claude/CLAUDE.md`, `GEMINI.md` | Project, agent-specific | Read directly — see "Already compatible?" below. |

`$HOME/.copilot` can be redirected with the `COPILOT_HOME` environment variable. Additional custom
directories for instructions can be registered via `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` (comma-separated).

Repository and agent instruction files are discovered in "standard locations": the repository root, the
current working directory, intermediate directories between them, and nested file paths — not just the
repo root.

### Frontmatter for modular `*.instructions.md` files

```yaml
---
applyTo: "src/**/*.ts,tests/**/*.ts"   # required — glob pattern(s), comma-separated for multiple
excludeAgent: "code-review"             # optional — "code-review" or "cloud-agent"
---

Instruction body in markdown.
```

`applyTo` glob examples: `*.py` (current dir only), `**/*.py` (recursive, all dirs), `src/**/*.py`
(recursive within `src/`), `**/subdir/**/*.py` (nested `subdir` at any depth). A modular file with no
matching `applyTo` pattern for the files in play simply doesn't activate — this is Copilot's equivalent of
a "situational" rule.

### Behavioral notes

- Copilot combines instructions from all applicable user-level and repo-wide files and removes exact
  duplicates, but establishes **no general precedence hierarchy** between them — don't assume repo beats
  personal or vice versa.
- `@relative/path` syntax inside an instructions file pulls in another file's content by reference. This
  does **not** work inside `GEMINI.md` or `*.instructions.md` files — only in the primary
  `copilot-instructions.md` files.
- Changes to instruction files require exiting and restarting (or resuming) the session to take effect.
- `/instructions` lists active instruction files and lets you toggle them on/off for the session.

## Already compatible?

**Yes, largely — this is Copilot's most generous compatibility surface for rules.** Copilot reads
`CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md`, and `GEMINI.md` directly as agent-specific instruction
files, discovered in the same standard locations as its own `copilot-instructions.md`. A project that
already has a `CLAUDE.md` generally needs **no translation** for Copilot CLI to pick it up.

The one thing that doesn't carry over: `@path` references inside a `CLAUDE.md` work fine (it's not
`GEMINI.md` or `*.instructions.md`), but confirm any relative paths still resolve correctly from wherever
Copilot considers the working root — this isn't guaranteed identical to how Claude Code resolves the same
reference.

## Translation steps (only if a Copilot-native file is actually wanted)

1. Copy `CLAUDE.md` content into `.github/copilot-instructions.md` — the format is plain markdown in both,
   so this is usually a straight copy, not a rewrite.
2. If the CLAUDE.md mixes always-relevant content with narrow, situational guidance (e.g., "when working
   with the payments module, do X"), consider splitting the situational part into
   `.github/instructions/payments.instructions.md` with `applyTo: "**/payments/**"` frontmatter — this is
   an improvement Copilot's structure enables that a flat CLAUDE.md doesn't, worth offering even though
   it's not required for compatibility.
3. Leave the original `CLAUDE.md` in place unless the user asks to remove it — Claude Code will still read
   it, and Copilot will too (per "Already compatible?" above), so keeping both isn't redundant if the team
   uses both tools.
4. If the user wants a file that Copilot reads but Claude Code should ignore, use `excludeAgent` on a
   modular `*.instructions.md` file rather than duplicating content with contradictory instructions in two
   places.
