# Skills

## Native format

`SKILL.md` inside a named directory — the directory name is the skill's invocation identifier
(`/<dirname>`).

**Project-scoped** (commit to git): `.devin/skills/<name>/SKILL.md`, `.windsurf/skills/<name>/SKILL.md`,
or `.agents/skills/<name>/SKILL.md` — DevIn checks all three, so which one to use depends on whether the
project already standardizes on one.

**Global** (all projects): `~/.config/devin/skills/<name>/SKILL.md` on Linux/macOS,
`%APPDATA%\devin\skills\<name>\SKILL.md` on Windows.

### Frontmatter schema

```yaml
---
name: my-skill              # optional, defaults to directory name
description: >              # what it does + when to use it, shown in completions
  One or two sentences.
argument-hint: "[filename]" # optional usage hint shown alongside the slash command
model: sonnet                # optional, overrides the active model for this skill's execution
subagent: true                # optional bool, run as an isolated worker (default: subagent_general profile)
agent: reviewer               # optional, use a specific custom subagent profile instead (see subagents.md)
allowed-tools:                 # optional, restricts tool access; omit to allow everything
  - read
  - edit
  - grep
  - glob
  - exec
  - mcp__github__list_issues
permissions:                    # optional, additive scope overrides — cannot override a higher-level deny
  allow: ["exec(git status)"]
  deny: ["exec(rm *)"]
  ask: ["exec(git push*)"]
triggers: [user, model]          # optional, default is both — set [user] to disable autonomous invocation
---

The markdown body is the prompt injected when the skill runs. Write it like instructions to the agent:
what to do, in what order, with what output format.
```

Key behavioral notes:

- `allowed-tools` values are lowercase generic categories (`read`, `edit`, `grep`, `glob`, `exec`) plus
  MCP tool names verbatim (`mcp__server__tool`) — **not** Claude Code's capitalized tool names.
- `subagent: true` vs `agent: <profile>`: use `subagent: true` for a quick isolated-context worker with
  default behavior; use `agent: <profile>` when a custom subagent profile (see subagents.md) with
  specific tool restrictions or a different model should run the skill instead. Don't nest subagents —
  DevIn's docs explicitly call out that orchestrator patterns run one level deep only.
- `triggers` controls *how* the skill can be invoked: `user` means only an explicit `/skill-name`
  command; `model` means the agent can decide to invoke it autonomously based on the description; default
  is both enabled.

### Best practices (per DevIn's own docs)

- Keep each skill focused on one task.
- Include output examples in the prompt body.
- Restrict `allowed-tools` for anything touching destructive or sensitive operations.
- Test iteratively with `/skill-name` before relying on autonomous (`model`) triggering.

## Already compatible?

**No — always needs translation.** DevIn does not read `.claude/skills/` directly; there's no import path
for skills the way there is for CLAUDE.md rules. Every Claude Code skill needs an explicit DevIn copy.

The good news: the core format (a directory containing `SKILL.md` with YAML frontmatter + markdown body,
optional `scripts/`, `references/`, `assets/` subdirectories) is the same convention in both tools, so
bundled resources need no changes — only the frontmatter and the file's new home directory.

## Translation steps (Claude Code skill → DevIn skill)

1. Create `.devin/skills/<name>/SKILL.md` (or the global equivalent) mirroring the source directory
   structure — copy any `scripts/`, `references/`, `assets/` subdirectories unchanged.
2. Frontmatter field mapping:
   - `name`, `description` — copy as-is.
   - `allowed-tools` — remap Claude Code's capitalized tool names to DevIn's lowercase ones:
     `Read`→`read`, `Edit`→`edit`, `Grep`→`grep`, `Glob`→`glob`, `Bash`→`exec`. MCP tool names
     (`mcp__server__tool`) carry over unchanged. Claude's `WebFetch`/`WebSearch` have no documented DevIn
     equivalent — flag this to the user rather than silently dropping or guessing at a mapping.
   - Claude Code has no `argument-hint`, `subagent`, `agent`, `permissions`, or `triggers` fields — these
     are new DevIn capabilities, not translations. Add them only if they add value (e.g., add
     `triggers: [user]` if the skill does something destructive enough that autonomous triggering would
     be unwelcome).
3. Copy the markdown body verbatim — the instructional content doesn't need to change; only the
   frontmatter and any tool-name references *within* the prose (e.g., "use the Bash tool" → "use the exec
   tool") if the skill explicitly names tools in its instructions.
4. If the original skill runs as a Claude Code subagent-type Task or references `Agent`/subagent
   delegation, map that to DevIn's `subagent: true` or `agent: <profile>` field instead of prose
   instructions to "spawn an agent" — DevIn expects this declared in frontmatter.
5. Validate by invoking `/name` in a DevIn CLI session.
