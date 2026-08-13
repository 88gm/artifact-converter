---
name: claude-code-artifacts
description: Translate existing GitHub Copilot CLI or DevIn CLI harness artifacts (Copilot skills/custom instructions/hooks/custom agents/plugins/MCP config, DevIn skills/AGENTS.md rules/hooks/subagents/plugins/MCP config) into Claude Code equivalents (SKILL.md skills, CLAUDE.md, .claude/settings.json hooks, .claude/agents subagents, .claude-plugin plugins, .mcp.json). Use this whenever the user wants a Copilot or DevIn artifact to work with "Claude Code", ".claude/", "CLAUDE.md", or asks to make a skill/rule/hook/subagent/plugin "compatible with Claude Code", "portable across harnesses", or to "migrate", "port", or "convert" a Copilot CLI or DevIn CLI artifact to Claude Code. Also use when the user pastes a Copilot or DevIn artifact file and asks what its Claude Code equivalent looks like.
---

# Claude Code Artifact Translation (from Copilot CLI / DevIn CLI)

This skill is the mirror image of `copilot-cli-artifacts` and `devin-cli-artifacts`: those translate
*from* Claude Code *into* Copilot or DevIn; this one translates *from* Copilot or DevIn *into* Claude
Code. Some artifact types already have a passthrough in one direction but not the other — always check
this skill's own "Already compatible?" notes rather than assuming symmetry with the forward-direction
skills.

## Decision tree

1. **Identify the source harness and artifact type first.** Ask if ambiguous, or infer from the file:
   `.github/skills/`, `.github/copilot-instructions.md`, `.github/hooks/*.json`, `.github/agents/*.agent.md`
   → Copilot CLI. `.devin/skills/`, `AGENTS.md`, `.devin/hooks.v1.json`, `.devin/agents/`,
   `.devin-plugin/` → DevIn CLI.
2. **Check the matching reference file's "Already compatible?" section before writing anything.** Several
   artifact types need zero or near-zero translation because Claude Code already reads the source file
   directly, or the on-disk shape is close enough that only a field rename is needed.
3. **Read the full source file before touching anything** — don't guess at frontmatter fields from memory.
4. **Translate additively.** Write the new `.claude/...` file alongside the source; do not delete or
   rewrite the original — the source harness still needs it.

## Quick reference: artifact types and native Claude Code locations

| Artifact | Claude Code project location | Claude Code personal location | Details |
|---|---|---|---|
| Rules | `CLAUDE.md`, `.claude/CLAUDE.md` | `~/.claude/CLAUDE.md` | [references/rules.md](references/rules.md) |
| Skills | `.claude/skills/<name>/SKILL.md` | `~/.claude/skills/<name>/SKILL.md` | [references/skills.md](references/skills.md) |
| Hooks | `.claude/settings.json` → `"hooks"` key | `~/.claude/settings.json` | [references/hooks.md](references/hooks.md) |
| Subagents | `.claude/agents/<name>.md` | `~/.claude/agents/<name>.md` | [references/subagents.md](references/subagents.md) |
| Plugins & MCP | `.claude-plugin/plugin.json`, `.mcp.json` | `~/.claude/...` (plugin marketplaces), personal `.mcp.json` | [references/plugins-mcp.md](references/plugins-mcp.md) |

Every reference file covers **both** source harnesses in one place, each with the same three subsections:
**Already compatible?** (does Claude Code read the source file as-is, no work needed), **Translation
steps: from Copilot CLI**, and **Translation steps: from DevIn CLI**.

## Summary by artifact type (which source needs real work)

| Artifact | From Copilot CLI | From DevIn CLI |
|---|---|---|
| Rules | ⚠️ Copy content into `CLAUDE.md`; modular `*.instructions.md` with `applyTo` globs has no direct Claude Code equivalent | ⚠️ Copy `AGENTS.md` content into `CLAUDE.md`; `.devin/rules/*.md` with `trigger: glob`/`model_decision` ports better as a **Skill** than a rule |
| Skills | ⚠️ Same `SKILL.md` core format, but needs a directory move into `.claude/skills/` plus a tool-name check | ⚠️ Always needs translation — remap lowercase `allowed-tools` back to capitalized, drop DevIn-only fields |
| Hooks | ⚠️ Real rewrite — event names, schema shape, and matcher scoping all differ | ✅ Near-copy — same JSON shape, just remap `matcher` tool names back to capitalized |
| Subagents | ⚠️ Move + rename `.agent.md` → `.md`, into `.claude/agents/` | ⚠️ Move into `.claude/agents/`, rename `allowed-tools` → `tools`, remap casing |
| Plugins | ⚠️ Move manifest into `.claude-plugin/plugin.json` (nested, not root) | ✅ Often already dual-compatible — DevIn plugins may already ship `.claude-plugin/plugin.json` alongside `.devin-plugin/` |
| MCP | ✅ Near-copy — drop/ignore Copilot-only `type` and per-server `tools` fields | ⚠️ Near-copy but needs the `mcpServers` wrapper key added back (DevIn's file is a bare server object) |

## Workflow

1. Read the source artifact file(s) in full.
2. Open the matching reference file and check "Already compatible?" for the source harness. If Claude
   Code already reads the file (or the shape is close enough that only the caveat mentioned applies),
   say so and stop — don't manufacture a duplicate file the user didn't ask for.
3. If real translation is needed, apply "Translation steps: from Copilot CLI" or "Translation steps: from
   DevIn CLI" as appropriate — field renames, path moves, tool-name casing changes.
4. Preserve the source file; write the Claude Code version alongside it.
5. Validate: JSON files parse; `.claude/settings.json` hook `matcher` values target Claude Code's tool
   names (`Bash`, `Read`, `Edit`, `Grep`, `Glob`, `WebFetch`, `WebSearch`), not the source harness's; a new
   skill responds to `/<dirname>` in a Claude Code session; a new subagent shows up via the `Agent` tool.

## Cross-cutting gotchas worth flagging to the user

- **Tool-name casing is the single most common silent break.** Copilot's tool vocabulary for
  `tools`/`allowed-tools` is not fully documented — don't invent a mapping table, carry over unambiguous
  MCP tool names (`mcp__server__tool`) unchanged, and flag anything else for the user to confirm live.
  DevIn's vocabulary is well-documented and lowercase (`read`, `edit`, `grep`, `glob`, `exec`) — map it
  deterministically back to Claude Code's capitalized names (`Read`, `Edit`, `Grep`, `Glob`, `Bash`).
- **Fields that don't exist in Claude Code should be dropped, not silently reinterpreted.** DevIn's
  `argument-hint`, `subagent`, `agent`, `permissions`, `triggers` (skills) and `max-nesting` (subagents),
  and Copilot's per-server MCP `tools` allowlist, have no Claude Code equivalent. If a field is
  load-bearing for the source behavior (e.g. `triggers: [user]` preventing autonomous invocation), tell
  the user there's no direct translation rather than approximating it silently.
- **Hooks from Copilot need a real rewrite; hooks from DevIn need a light touch.** DevIn's hook file
  shares Claude Code's exact JSON shape (event name + `matcher`/`hooks` arrays); Copilot's is structurally
  different (per-event files, camelCase names, `bash`/`powershell` keys, no matcher concept at all). Don't
  apply the same translation effort to both — check [references/hooks.md](references/hooks.md) first.
- **MCP config from DevIn is a bare server object, not wrapped in `mcpServers`.** Forgetting to add the
  wrapper key when copying `.devin/mcp_config.json` into `.mcp.json` is the most common mistake there.
  Also check `.devin/mcp_config.local.json` for secrets that were split out — Claude Code doesn't document
  an equivalent gitignored split, so flag this to the user instead of committing credentials.
- **Don't assume Claude Code reads the source file directly.** Unlike Copilot and DevIn, which both have
  generous passthrough for Claude Code's own `CLAUDE.md`/`.claude/settings.json`/`.claude-plugin/`, the
  reverse is mostly *not* true — Claude Code does not natively scan `.github/`, `.devin/`, or `AGENTS.md`.
  Always verify against the reference file rather than assuming symmetry.
