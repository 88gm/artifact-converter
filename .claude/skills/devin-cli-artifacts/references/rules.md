# Rules & AGENTS.md

DevIn's word of caution, direct from its docs: "To improve coding ability, speed of completion, and lower
cost, use Skills instead whenever possible. Skills are only injected when relevant; Rules should stay
minimal." Rules are for small, always-relevant standards (naming conventions, "never commit secrets");
anything workflow-shaped belongs in a skill instead — see [skills.md](skills.md).

## Native format

Plain markdown, no required frontmatter for the simple always-on files. Project-level, read automatically:

| File | Behavior |
|---|---|
| `AGENTS.md` | Recommended. Always-on, loaded at session start. |
| `AGENT.md` | Singular alternative name, same behavior. |
| `AGENTS.local.md` | Personal, gitignored — for individual preferences that shouldn't be shared. |
| `.devin/global_rules.md` | Single always-on file, alternative location. |
| `.devin/rules/*.md` | One rule per file, with trigger frontmatter (see below) — not always-on. |

Global (apply to every project): `~/.config/devin/AGENTS.md` on Linux/macOS,
`%APPDATA%\devin\AGENTS.md` on Windows.

### Frontmatter for `.devin/rules/*.md` (triggered rules)

Unlike the root `AGENTS.md`, files in `.devin/rules/` are not always-on by default — they declare a
trigger:

```markdown
---
trigger: glob
globs: ["**/*.sql"]
---

Always use parameterized queries. Never interpolate user input into SQL strings.
```

Valid `trigger` values: `always_on`, `manual`, `model_decision`, `agent`, `glob` (paired with a `globs`
list of file patterns). Use `glob` for rules that should only load when the agent touches matching files
(e.g., SQL conventions only when editing `.sql` files) — this keeps unrelated rules out of context.

### Loading behavior

- Root-level rules (`AGENTS.md`, `.devin/global_rules.md`) load at session start, every session.
- Subdirectory rules load lazily, only when the agent accesses files in that subdirectory.
- Project and global rules both load simultaneously — they're additive, not either/or.
- `.devin/` takes precedence over `.windsurf/` when both exist.

## Already compatible?

**Yes, largely.** DevIn CLI reads `CLAUDE.md` directly as a project rule file, and `~/.claude/CLAUDE.md`
as a global one — no translation needed for a plain CLAUDE.md to work in DevIn. This is controlled by
`~/.config/devin/config.json`:

```json
{"read_config_from": {"agents_standard": true, "cursor": true, "windsurf": true, "claude": true}}
```

If this is enabled (it's generally on by default per the docs), a user who just wants their existing
CLAUDE.md to work under DevIn doesn't need to do anything. Only translate to `AGENTS.md` if the user
specifically wants a DevIn-native file (e.g., to standardize a team on DevIn, or to stop relying on the
import path), or if they've disabled `read_config_from.claude`.

Note also: `.cursor/rules/` and `.windsurf/rules/` support the same trigger-frontmatter scheme
(`trigger`/`globs` for DevIn-style; Cursor itself uses `alwaysApply` boolean + `globs` — DevIn reads
Cursor's native format for imported Cursor rules, it doesn't require Cursor rules to use DevIn's schema).

## Translation steps (CLAUDE.md → AGENTS.md)

1. Copy `CLAUDE.md` content into `AGENTS.md` at the project root — the format is plain markdown in both,
   so this is usually a straight copy, not a rewrite.
2. If the CLAUDE.md mixes "always relevant" content with narrow, situational guidance (e.g., "when
   working with the payments module, do X"), consider splitting the situational parts into
   `.devin/rules/payments.md` with `trigger: glob` / `globs: ["**/payments/**"]` — this is an improvement
   DevIn's structure enables that flat CLAUDE.md files don't, worth offering even though it's not
   strictly required for compatibility.
3. Leave the original `CLAUDE.md` in place unless the user asks to remove it — Claude Code will still
   read it, and DevIn will too (per "Already compatible?" above), so keeping both isn't redundant if the
   team uses both tools.
