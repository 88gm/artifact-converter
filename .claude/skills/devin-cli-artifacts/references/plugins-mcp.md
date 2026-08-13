# Plugins & MCP Servers

## Plugins

### Native format

A plugin is a bundled, installable collection of the other artifact types, sourced from a GitHub repo,
git URL, or local folder. It's the unit of installation — users install a whole plugin, not individual
skills within it. Installed plugin skills become `/<plugin>:<skill>` slash commands.

Directory structure:

```
my-plugin/
├── .devin-plugin/
│   └── plugin.json          # required manifest
├── AGENTS.md                 # optional, always-on rule for every session with this plugin installed
├── rules/                     # optional, triggered rules (same trigger frontmatter as rules.md)
├── agents/                     # optional, custom subagents — same format as subagents.md
├── hooks.json                   # optional, lifecycle hooks — registers for every session, same schema as hooks.md
├── mcp_config.json               # optional, MCP servers that start with the session
└── skills/
    └── <name>/
        └── SKILL.md               # standard skill format, no plugin-specific changes
```

### Manifest (`plugin.json`)

Only `name` is required (lowercase alphanumeric, `-` or `.` separators). Optional metadata: `version`,
`description`, `author`, `homepage`, `repository`, `license`, `keywords`. Dependency management:
`requiredPlugins` (auto-installed; installation blocks if any is forbidden), `optionalPlugins`
(allow-listed but not auto-installed), `forbiddenPlugins` (deny-list, supports glob patterns). Dependency
entries accept shorthand (`"owner/repo"`), a git URL, or an object specifying source type/URL/subfolder.
Also supports `skills` (custom path override) and `mcpServers` (inline MCP declarations) fields directly
in the manifest.

### Already compatible? (this is the interesting one)

**Yes, largely — this is DevIn's most generous compatibility surface.** DevIn recognizes three plugin
manifest formats side by side, with no translation required to *install* any of them:

1. `.devin-plugin/plugin.json` — DevIn-native.
2. `.claude-plugin/plugin.json` — **Claude Code plugins work in DevIn as-is.**
3. Root `plugin.json` — the open Agent Plugins 1.0.0 spec.

DevIn also honors a Claude plugin's root `.mcp.json` and the manifest's `mcpServers` field directly. So a
plugin authored for Claude Code's plugin system generally does not need translation to install and run
under DevIn — only flag this to the user, don't manufacture a parallel `.devin-plugin/` copy unless they
specifically want a DevIn-branded manifest (e.g., different name/description for a DevIn-specific
marketplace listing) or the plugin uses Claude-specific features that don't map (rare).

### Translation steps (only if a DevIn-specific manifest is actually wanted)

1. Create `.devin-plugin/plugin.json` alongside (not replacing) `.claude-plugin/plugin.json` if the goal
   is to publish to both ecosystems with distinct metadata.
2. The internal skills/rules/hooks/agents/MCP config generally do not need duplicating — both manifest
   formats point at the same directory conventions, so translate those *contents* per the individual
   reference files (skills.md, rules.md, hooks.md, subagents.md) only where an artifact type genuinely
   isn't compatible (skills and subagents need translation; rules and hooks generally don't).

## MCP Servers

### Native format

Config file locations (DevIn v3000.3+):

| Scope | File |
|---|---|
| User | `~/.config/devin/mcp_config.json` (`%APPDATA%\devin\mcp_config.json` on Windows) |
| Project (shared, version-controlled) | `.devin/mcp_config.json` |
| Project (local override, gitignored — for personal API keys) | `.devin/mcp_config.local.json` |

Older DevIn versions stored servers under an `mcpServers` key in the main config file; these migrate
automatically on startup, so don't manually port them — just let DevIn start once.

### Schema

Local/stdio server:
```json
{
  "github": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": { "GITHUB_TOKEN": "ghp_..." },
    "disabled": false
  }
}
```

Remote server (HTTP/SSE):
```json
{
  "notion": {
    "url": "https://mcp.notion.com/mcp",
    "transport": "http",
    "headers": {},
    "oauthClientId": "...",
    "oauthClientSecret": "...",
    "oauthResource": "...",
    "disabled": false
  }
}
```

CLI shortcuts: `devin mcp add -s project <name> <url>` or `-s user <name> <url>` (default scope with no
flag is the local/gitignored file).

### Already compatible?

**Structurally, yes.** Claude Code's `.mcp.json` uses the same `mcpServers`-keyed object of
`{command,args,env}` or `{url,...}` entries. DevIn also honors a Claude plugin's root `.mcp.json`
directly when the plugin format is `.claude-plugin/`. For a plain (non-plugin) project's `.mcp.json`,
copying the server entries into `.devin/mcp_config.json` is close to a direct copy — check field names
match the schema above (DevIn's remote-server fields like `oauthClientId` may not have a 1:1 Claude Code
equivalent; carry over what exists, don't invent values for what doesn't).

### Translation steps

1. Copy each server entry from `.mcp.json`'s `mcpServers` object into `.devin/mcp_config.json` (note:
   DevIn's file *is* the servers object directly, not wrapped in an `mcpServers` key — check whichever
   DevIn version's docs the user is on, since older versions did wrap it).
2. Split anything containing secrets (API keys, tokens in `env`) into `.devin/mcp_config.local.json`
   instead, since that file is gitignored — don't commit credentials into the shared project file.
3. Leave the original `.mcp.json` in place; it keeps working for Claude Code and, in the plugin case,
   DevIn reads it directly anyway.
