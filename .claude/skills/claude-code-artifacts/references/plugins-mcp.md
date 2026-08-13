# Plugins & MCP Servers

## Plugins

### Native format (Claude Code)

`.claude-plugin/plugin.json` as the manifest (nested in a `.claude-plugin/` subdirectory, not at plugin
root), plus optional `agents/`, `skills/` directories, hook config, and a `.mcp.json`.

### From Copilot CLI

Source: `plugin.json` loose at the plugin root (not nested), `agents/`, `skills/`, `hooks.json`,
`.mcp.json`. Distributed via marketplaces (`marketplace.json`), direct repos, or local paths.

#### Already compatible?

**No documented passthrough.** Nothing confirms Claude Code reads a root-level `plugin.json` the way
Copilot reads `.claude/skills/` — treat the manifest as needing an explicit nested copy.

#### Translation steps

1. Create `.claude-plugin/plugin.json` (nested subdirectory) alongside — not replacing — the root-level
   `plugin.json`, if the goal is to publish for both ecosystems.
2. Verify required vs. optional manifest fields against Claude Code's current plugin reference before
   publishing — don't assume the two schemas match field-for-field beyond `name`, `version`, `description`,
   and the component directories.
3. Internal `skills/` usually needs only the directory-location fix described in
   [skills.md](skills.md) (the `SKILL.md` core format already matches). Internal `agents/` and
   `hooks.json` need the real translations described in [subagents.md](subagents.md) and
   [hooks.md](hooks.md) respectively — don't treat those as light touch-ups just because the plugin
   wrapper is close.

### From DevIn CLI

Source: `.devin-plugin/plugin.json` (DevIn-native), or a plugin may already ship a root-level `plugin.json`
(open Agent Plugins 1.0.0 spec), or — notably — may already ship `.claude-plugin/plugin.json` directly,
since **DevIn reads Claude Code plugin manifests natively as one of three side-by-side formats it
recognizes.**

#### Already compatible?

**Check for an existing `.claude-plugin/plugin.json` first — it may already be there and require nothing.**
DevIn's generous compatibility here is specifically about DevIn reading Claude's format, not the other way
around: if the plugin was authored DevIn-first and only has `.devin-plugin/plugin.json`, Claude Code does
not read that natively.

#### Translation steps (only if no `.claude-plugin/plugin.json` exists yet)

1. Create `.claude-plugin/plugin.json` alongside `.devin-plugin/plugin.json`. Field shape is generally
   close (`name` required; `version`, `description`, component directories) but verify against Claude
   Code's current plugin reference rather than assuming a byte-for-byte match.
2. Internal skills and subagents inside a DevIn-authored plugin still need the real per-type translation
   from [skills.md](skills.md) and [subagents.md](subagents.md) — the plugin wrapper being close doesn't
   change that DevIn skills/subagents always need translating to reach Claude Code.
3. Internal hooks and MCP config are usually a near-copy per [hooks.md](hooks.md) and the MCP section
   below — DevIn's hook schema is Claude-shared, and its MCP schema is close.

## MCP Servers

### Native format (Claude Code)

`.mcp.json` at the project root, servers nested under `mcpServers`, entries shaped `{command,args,env}`
(stdio) or `{url,...}` (remote) — no explicit `type` field; Claude Code infers stdio-vs-remote from which
keys are present.

### From Copilot CLI

Source: `.mcp.json` / `.github/mcp.json` (project), `~/.copilot/mcp-config.json` (personal). Same
`mcpServers`-keyed base schema, but Copilot wants an explicit `type` field (`"local"`, `"stdio"`, `"http"`,
`"sse"`) and supports an optional per-server `tools` allow-list with no Claude Code equivalent.

#### Already compatible?

**Near-direct copy.** The base schema (`mcpServers`-keyed, `{command,args,env}` or `{url,...}`) is shared.

#### Translation steps

1. Copy each `mcpServers` entry as-is into (or directly reuse) `.mcp.json` at the project root — the same
   file generally works for both harnesses simultaneously.
2. Drop the `type` field — Claude Code doesn't use it and infers transport from the keys present. Leaving
   it in is likely harmless but unnecessary; strip it for a clean native file if publishing a
   Claude-specific copy.
3. Drop any per-server `tools` allow-list — no Claude Code equivalent. If it was scoping which tools a
   server exposes for safety reasons, flag that to the user rather than silently dropping the restriction.
4. Leave secrets (tokens, keys in `env`/`headers`) where they already are; if `.mcp.json` is committed,
   keep credentials out of it on the Claude Code side the same way the user already does for Copilot.

### From DevIn CLI

Source: `.devin/mcp_config.json` (shared project) / `.devin/mcp_config.local.json` (gitignored, personal
or secret-bearing) — **the file is the servers object directly, not wrapped in a `mcpServers` key** (older
DevIn versions wrapped it; migrated automatically on startup, don't port that old shape manually). Personal:
`~/.config/devin/mcp_config.json`.

#### Already compatible?

**Near-direct copy, but the wrapper key must be added.** This is the most common mistake in this
direction: forgetting that Claude Code's `.mcp.json` wraps servers under `"mcpServers"` while DevIn's file
does not.

#### Translation steps

1. Copy the server entries from `.devin/mcp_config.json` into `.mcp.json`, wrapping them under a
   top-level `"mcpServers"` key.
2. If `.devin/mcp_config.local.json` holds secrets split out from the shared file, decide how to handle
   that on the Claude Code side — Claude Code doesn't document an equivalent gitignored-split convention
   for `.mcp.json`, so flag this to the user rather than merging secrets into a file that might get
   committed.
3. Check DevIn-specific remote-server fields (e.g. `oauthClientId`) for a Claude Code equivalent before
   copying — carry over what has a documented match, don't invent values for fields Claude Code doesn't
   have.
4. Keep `.devin/mcp_config.json` in place — DevIn keeps reading it, and if the plugin format is
   `.claude-plugin/`, DevIn may already be reading a project's `.mcp.json` directly too.
