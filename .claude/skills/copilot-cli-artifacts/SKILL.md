---
name: copilot-cli-artifacts
description: Author new GitHub Copilot CLI harness artifacts (skills, custom instructions, hooks, custom agents, plugins, MCP config) and translate existing Claude Code artifacts (SKILL.md skills, CLAUDE.md, .claude/settings.json hooks, .claude/agents subagents, .claude-plugin plugins, .mcp.json) into Copilot CLI equivalents. Use this whenever the user wants something to work with "Copilot", "Copilot CLI", "GitHub Copilot", ".github/", or asks to make a skill/rule/hook/subagent/plugin "compatible with Copilot", "portable across harnesses", or to "migrate" or "convert" a Claude Code skill/hook/subagent to Copilot. Also use when the user is building a new Copilot CLI extension from scratch and needs correct file locations, frontmatter schemas, or JSON formats.
---

# Copilot CLI Artifact Authoring & Translation

Copilot CLI's extensibility system overlaps with Claude Code's more than most harnesses: it literally
scans `.claude/skills/` and reads `CLAUDE.md`/`.claude/CLAUDE.md` as native instruction sources — no
importer, no shim, just a second location it checks. Hooks and custom agents are the opposite story: same
concepts, meaningfully different schemas. So the first question is always: **does this artifact type even
need translating, or does Copilot already read it as-is?**

## Decision tree

1. **Is the user creating something new for Copilot?** → Jump to the relevant reference file below and
   author directly in Copilot's native format (`.github/...` or `~/.copilot/...`).
2. **Is the user translating an existing Claude Code artifact?** → Check the "Already compatible?" note in
   the matching reference file first. Skills and rules need little to no work; hooks and subagents need a
   real rewrite.
3. **Unsure which artifact type?** → Ask, or inspect what exists: `.claude/skills/` → skills, `CLAUDE.md`/
   `.claude/CLAUDE.md` → instructions/rules, `.claude/settings.json` "hooks" key → hooks, `.claude/agents/`
   → subagents, `.claude-plugin/` → plugins, `.mcp.json` → MCP servers.

## Quick reference: artifact types and native Copilot locations

| Artifact | Copilot project location | Copilot personal location | Details |
|---|---|---|---|
| Instructions/Rules | `.github/copilot-instructions.md`, `.github/instructions/**/*.instructions.md` | `~/.copilot/copilot-instructions.md`, `~/.copilot/instructions/**/*.instructions.md` | [references/rules.md](references/rules.md) |
| Skills | `.github/skills/<name>/SKILL.md` (also reads `.claude/skills/`, `.agents/skills/`) | `~/.copilot/skills/<name>/SKILL.md` (also `~/.agents/skills/`) | [references/skills.md](references/skills.md) |
| Hooks | `.github/hooks/<name>.json` | `~/.copilot/hooks/` (`%USERPROFILE%\.copilot\hooks\` on Windows) | [references/hooks.md](references/hooks.md) |
| Custom agents | `.github/agents/<name>.agent.md` | `~/.copilot/agents/<name>.agent.md` (wins on name collision) | [references/subagents.md](references/subagents.md) |
| Plugins & MCP | `plugin.json` at plugin root; `.mcp.json` / `.github/mcp.json` | `~/.copilot/mcp-config.json`; `~/.copilot/settings.json` (`enabledPlugins`) | [references/plugins-mcp.md](references/plugins-mcp.md) |

Every reference file has the same three sections: **Native format** (how to author it fresh for Copilot),
**Already compatible?** (what Copilot reads directly from Claude Code with no translation), and
**Translation steps** (the mechanical diff when a real conversion is needed).

## Workflow for translating a Claude Code artifact

1. Read the existing Claude Code file(s) in full before touching anything — don't guess at frontmatter
   fields from memory.
2. Check the relevant reference file's "Already compatible?" section. If Copilot reads the file natively
   (skills in `.claude/skills/`, `CLAUDE.md` as instructions), tell the user that and stop — don't
   manufacture unnecessary duplicate files. Some users will still want a native `.github/` copy for a team
   that standardizes on Copilot; that's fine, just don't imply it's *required* for Copilot to work.
3. If translation is genuinely needed (hooks, custom agents), apply the mechanical mapping from the
   reference file's "Translation steps" — event names, schema shape, file locations all differ more than
   they do for DevIn-style harnesses, so treat these as closer to a rewrite than a find-and-replace.
4. Validate JSON files parse. For hooks specifically, Copilot's schema is structurally different from
   Claude's (per-event command objects with `bash`/`powershell` keys, not `matcher` + `hooks` arrays) — see
   [references/hooks.md](references/hooks.md) before assuming a light touch-up is enough.
5. If a directory structure needs multiple files (a plugin, a skill with bundled resources), create the
   full tree in one pass rather than iterating file-by-file.

## Cross-cutting gotchas worth flagging to the user

- **Copilot natively scans `.claude/skills/`.** This is the single biggest compatibility win: a Claude
  Code skill directory usually needs zero file moves to be discovered by Copilot CLI — only frontmatter
  field differences matter (see [references/skills.md](references/skills.md)). Don't reflexively copy
  skills into `.github/skills/` without checking whether the user actually wants a Copilot-branded
  location versus just confirming the existing one already works.
- **Copilot reads `CLAUDE.md` (and `AGENTS.md`, `GEMINI.md`) directly as agent-specific instructions**,
  alongside its own `.github/copilot-instructions.md`. A bare CLAUDE.md generally needs no translation —
  see [references/rules.md](references/rules.md) for the combination/precedence caveats.
- **Hooks do NOT share a schema.** Claude Code hooks are keyed by PascalCase event name with
  `matcher`/`hooks` arrays and a tool-name regex; Copilot hooks are keyed by camelCase event name
  (`sessionStart`, `preToolUse`, ...) with `bash`/`powershell` command strings and no matcher concept
  (one file = one hook, not a matcher-scoped list). Treat hook translation as a rewrite, not a field
  rename. See [references/hooks.md](references/hooks.md).
- **Custom agents use `.agent.md`, not a bare `.md`.** File name must end `.agent.md`; Claude Code's
  `.claude/agents/<name>.md` files need both a rename and a location change. Frontmatter fields are close
  enough (`name`, `description`, `tools`) that the body/prompt usually copies over verbatim.
- **Tool names in `allowed-tools`/`tools` fields are underdocumented for Copilot** — the docs only confirm
  categories like `shell`/`bash` exist for pre-approval. Don't invent a full Claude→Copilot tool-name
  mapping table; carry over what's unambiguous (e.g., leave MCP tool names `mcp__server__tool` unchanged)
  and flag anything else for the user to verify against `/skills info` or `--allow-tool` output in a live
  session.
- **MCP config is close to a direct copy.** Both tools use a `.mcp.json` with server entries nested under
  `mcpServers` (Copilot also accepts bare top-level keys as a project-level shorthand). See
  [references/plugins-mcp.md](references/plugins-mcp.md) for the one real schema difference (`type: local`
  vs Claude's implicit stdio detection).
