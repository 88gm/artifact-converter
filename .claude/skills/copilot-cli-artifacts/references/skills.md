# Skills

## Native format

`SKILL.md` (case-sensitive) inside a lowercase, hyphenated directory — the directory name is the skill's
identifier.

```
.github/skills/github-actions-failure-debugging/
├── SKILL.md
├── script-file.sh          # optional
└── additional-resources/   # optional
```

### Storage locations

**Project skills** (repository-specific) — Copilot checks all three:
- `.github/skills/`
- `.claude/skills/`
- `.agents/skills/`

**Personal skills** (across projects):
- `~/.copilot/skills/`
- `~/.agents/skills/`

### Frontmatter schema

```yaml
---
name: my-skill                        # required, lowercase, hyphens for spaces
description: >                         # required — purpose + when Copilot should use it
  One or two sentences.
license: MIT                            # optional
allowed-tools: ["shell", "bash"]         # optional — pre-approve tools, skip confirmation prompts
---

Markdown body: instructions, examples, guidelines. Referenced scripts/resources become available once
the skill is loaded into context.
```

### Discovery & loading

Copilot discovers every file in a skill's directory when the skill is invoked; `SKILL.md` gets injected
into context and bundled scripts/resources become available for reference. Skills activate either by
explicit `/skill-name` invocation or by Copilot auto-selecting based on the `description`.

CLI/slash commands: `/skills list`, `/skills info <name>`, `/skills reload` (refresh mid-session),
`/skills remove <name>`; `copilot skill add` from the shell.

## Already compatible?

**Yes, largely — Copilot scans `.claude/skills/` directly as one of its three project skill locations.**
A Claude Code skill directory generally needs **zero file moves** to be discovered by Copilot CLI. Bundled
`scripts/`, `references/`, `assets/` subdirectories need no changes either — same directory convention in
both tools.

What genuinely differs is the frontmatter surface:
- Claude Code's `allowed-tools` uses capitalized tool names (`Read`, `Edit`, `Bash`, ...); Copilot's uses
  its own vocabulary (confirmed: `shell`/`bash`-style categories). The full Copilot tool-name list isn't
  documented in detail — don't invent a mapping table. If a skill has `allowed-tools` and needs to work
  identically under Copilot, flag the field for the user to verify with `/skills info` in a live session
  rather than guessing at exact values.
- Claude Code has no `license` field; Copilot has no `argument-hint`/`subagent`/`agent`/`permissions`/
  `triggers` fields that some other harnesses add — don't carry those over, they have no Copilot meaning.

## Translation steps (only needed for frontmatter fields, not file location)

1. If the skill already lives under `.claude/skills/`, tell the user it's already discoverable by Copilot
   CLI — stop here unless they specifically want a `.github/skills/` copy (e.g., to standardize a team on
   Copilot, or because `.claude/skills/` won't exist in a repo that drops Claude Code entirely).
2. If creating a Copilot-branded copy anyway: copy the directory to `.github/skills/<name>/` unchanged.
3. Frontmatter: copy `name` and `description` as-is. Leave `allowed-tools` values alone if reusing
   generic/MCP tool names (`mcp__server__tool` is unchanged across harnesses); flag any Claude-specific
   capitalized built-in tool name (`Bash`, `Read`, `Edit`, `Grep`, `Glob`, `WebFetch`, `WebSearch`) for the
   user to re-verify against Copilot's actual tool vocabulary rather than silently renaming to a guess.
4. Copy the markdown body verbatim; only touch prose that explicitly names a Claude Code tool (e.g., "use
   the Bash tool") if precision matters for the user's use case.
5. Validate with `/skills reload` then `/skill-name` in a Copilot CLI session.
