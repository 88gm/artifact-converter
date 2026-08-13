---
name: devin-cli-artifacts
description: Author new DevIn CLI harness artifacts (skills, AGENTS.md rules, hooks, subagents, plugins, MCP config) and translate existing Claude Code artifacts (SKILL.md skills, CLAUDE.md, .claude/settings.json hooks, .claude/agents subagents, .claude-plugin plugins, .mcp.json) into DevIn CLI equivalents. Use this whenever the user wants something to work with "DevIn", "DevIn CLI", ".devin/", "AGENTS.md", or asks to make a skill/rule/hook/subagent/plugin "compatible with DevIn", "portable across harnesses", or to "migrate" or "convert" a Claude Code skill/hook/subagent to DevIn. Also use when the user is building a new DevIn CLI extension from scratch and needs correct file locations, frontmatter schemas, or JSON formats.
---

# DevIn CLI Artifact Authoring & Translation

DevIn CLI's extensibility system deliberately mirrors Claude Code's (skills, rules, hooks, subagents,
plugins, MCP) closely enough that most artifacts translate with small, mechanical changes — mainly file
locations, frontmatter field names, and tool-name casing. It also natively *reads* several Claude Code
files directly (CLAUDE.md, `.claude/settings.json` hooks, `.claude-plugin/plugin.json` plugins), so the
first question to ask is always: **does this even need translating, or does DevIn already read it as-is?**

## Decision tree

1. **Is the user creating something new for DevIn?** → Jump to the relevant reference file below and
   author directly in DevIn's native format (`.devin/...`).
2. **Is the user translating an existing Claude Code artifact?** → Check the "Already compatible?" note
   in the matching reference file first. Several artifact types need zero or near-zero changes.
3. **Unsure which artifact type?** → Ask, or inspect what exists: `.claude/skills/` → skills,
   `CLAUDE.md`/`.claude/CLAUDE.md` → rules, `.claude/settings.json` "hooks" key → hooks,
   `.claude/agents/` → subagents, `.claude-plugin/` → plugins, `.mcp.json` → MCP servers.

## Quick reference: artifact types and native DevIn locations

| Artifact | DevIn project location | DevIn global location | Details |
|---|---|---|---|
| Rules | `AGENTS.md`, `.devin/rules/*.md`, `.devin/global_rules.md` | `~/.config/devin/AGENTS.md` (`%APPDATA%\devin\AGENTS.md` on Windows) | [references/rules.md](references/rules.md) |
| Skills | `.devin/skills/<name>/SKILL.md` | `~/.config/devin/skills/<name>/SKILL.md` (`%APPDATA%\devin\skills\` on Windows) | [references/skills.md](references/skills.md) |
| Hooks | `.devin/hooks.v1.json` | `~/.config/devin/config.json` (`"hooks"` key) | [references/hooks.md](references/hooks.md) |
| Subagents | `.devin/agents/<name>.md` or `.devin/agents/<name>/AGENT.md` | `~/.config/devin/agents/` | [references/subagents.md](references/subagents.md) |
| Plugins & MCP | `.devin-plugin/plugin.json`, `.devin/mcp_config.json` | `~/.config/devin/mcp_config.json` | [references/plugins-mcp.md](references/plugins-mcp.md) |

Every reference file has the same three sections: **Native format** (how to author it fresh for DevIn),
**Already compatible?** (what DevIn reads directly from Claude Code with no translation), and
**Translation steps** (the mechanical diff when a real conversion is needed).

## Workflow for translating a Claude Code artifact

1. Read the existing Claude Code file(s) in full before touching anything — don't guess at frontmatter
   fields from memory.
2. Check the relevant reference file's "Already compatible?" section. If DevIn reads the file natively,
   tell the user that and stop — don't manufacture unnecessary duplicate files. (Some users will still
   want a native `.devin/` copy for a team that standardizes on DevIn; that's fine, just don't imply it's
   *required* for DevIn to work.)
3. If translation is genuinely needed, apply the mechanical mapping from the reference file's
   "Translation steps" — field renames, path moves, tool-name casing changes. Preserve the original
   file; write the DevIn version alongside it (translation is additive, not destructive).
4. Validate JSON files parse and that any regex `matcher` fields in hooks target DevIn's tool names
   (`exec`, `read`, `edit`, ...), not Claude's (`Bash`, `Read`, `Edit`, ...) — this is the single most
   common mistake when porting hooks, since the event names match but the tool names don't.
5. If a directory structure needs multiple files (a plugin, a skill with bundled resources), create the
   full tree in one pass rather than iterating file-by-file.

## Cross-cutting gotchas worth flagging to the user

- **Tool name casing differs.** Claude Code: `Read`, `Edit`, `Grep`, `Glob`, `Bash`, `WebFetch`,
  `WebSearch`. DevIn: `read`, `edit`, `grep`, `glob`, `exec` (no direct `WebFetch`/`WebSearch`
  equivalent documented — flag this rather than inventing one). MCP tool names (`mcp__server__tool`)
  are the same in both.
- **Config precedence differs slightly.** DevIn's order (highest to lowest) is org/team → session →
  project-local (`.devin/config.local.json`) → project (`.devin/config.json`) → user. Hooks *accumulate*
  across all sources rather than overriding; permissions merge with the highest-priority denial always
  winning. See [references/rules.md](references/rules.md) for how this interacts with rule loading.
- **DevIn's skill `triggers` field has no Claude Code equivalent.** Claude Code controls autonomous
  invocation purely through the description; DevIn adds an explicit `triggers: [user, model]` field.
  When translating a Claude skill that should never auto-trigger, set `triggers: [user]` in the DevIn
  version — there's nothing to translate *from*, this is new behavior to consider.
  See [references/skills.md](references/skills.md).
- **Import instead of duplicate, when possible.** DevIn can be configured to read Cursor/Windsurf/Claude
  configs directly via `~/.config/devin/config.json` → `read_config_from`. If the user's goal is "make my
  existing Claude Code setup work in DevIn" rather than "produce a portable DevIn-native copy", enabling
  this is often less work and less drift than translating every file. Mention this option.
