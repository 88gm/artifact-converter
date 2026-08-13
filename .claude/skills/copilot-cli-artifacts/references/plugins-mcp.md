# Plugins & MCP Servers

## Plugins

### Native format

A plugin is a distributable, installable bundle of the other artifact types, sourced from a repository, a
marketplace, or a local path.

```
my-plugin/
├── plugin.json          # required manifest, at plugin root (not in a subdirectory)
├── agents/              # optional — *.agent.md custom agents
├── skills/              # optional — <name>/SKILL.md directories
├── hooks.json            # optional — lifecycle hooks
├── .mcp.json              # optional — MCP server configs
└── lsp.json                # optional — LSP server configs
```

The full `plugin.json` field spec lives in GitHub's "CLI plugin reference" doc — the fetched overview
confirms the manifest provides at minimum the plugin's name, but doesn't enumerate every optional field.
Don't invent fields; check the live reference (or an existing installed plugin's manifest) if the user
needs the complete schema.

### Distribution & installation

Three channels: **marketplaces** (registries defined by `marketplace.json`, hosting versioned plugins on
GitHub or elsewhere — e.g. copilot-plugins, awesome-copilot, claude-code-plugins), **repositories**
(direct install from a source repo), and **local paths**.

Install imperatively with `copilot plugin install` or `/plugin install`, or declaratively by adding to
`enabledPlugins` in `~/.copilot/settings.json` (personal) or `.github/copilot/settings.json` (repo — also
used by the cloud agent). Additional marketplaces register via `extraKnownMarketplaces`.

### Already compatible?

**Structurally close, but not a documented passthrough.** Copilot's plugin layout (`plugin.json` at root,
`agents/`, `skills/`, `hooks.json`, `.mcp.json`) closely mirrors Claude Code's plugin conventions
(`.claude-plugin/plugin.json`, `agents/`, `skills/`, hooks config, `.mcp.json`) — same component
directories, similar manifest idea. The one confirmed structural difference: **manifest location** —
Claude Code nests it in `.claude-plugin/plugin.json`; Copilot expects a bare `plugin.json` at the plugin
root. Nothing in the fetched docs confirms Copilot reads `.claude-plugin/` directly the way it reads
`.claude/skills/` or `CLAUDE.md`, so treat plugin manifests as needing an explicit Copilot-native copy
rather than assuming a passthrough.

### Translation steps

1. Create `plugin.json` at the plugin root (sibling to, not replacing, `.claude-plugin/plugin.json` if the
   goal is publishing to both ecosystems).
2. The internal `agents/`, `skills/`, hooks, and MCP config generally still need the per-artifact-type
   translation from the other reference files (skills.md often needs none if already scanning
   `.claude/skills/`-equivalent paths; hooks.md and subagents.md need real rewrites — see those files).
3. Confirm required vs. optional manifest fields against the live "CLI plugin reference" before publishing
   — don't guess at a schema beyond what's confirmed (`name` required; `version`, `description`, and the
   component directories are the only fields the fetched overview verifies).

## MCP Servers

### Native format

| Scope | File | Notes |
|---|---|---|
| Personal | `~/.copilot/mcp-config.json` | User-level, applies globally |
| Project (primary) | `.mcp.json` | Searched from working directory up to repo root |
| Project (secondary) | `.github/mcp.json` | Committed to repository |

Precedence: files closer to the working directory override files higher up; project configs override
personal configs. The built-in GitHub MCP server is available with no configuration.

### Schema

Server entries nest under a top-level `mcpServers` key (project files also accept bare top-level keys as a
shorthand — both are valid, `mcpServers` is the more explicit/portable form):

```json
{
  "mcpServers": {
    "github": {
      "type": "local",
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "GITHUB_PERSONAL_ACCESS_TOKEN", "ghcr.io/github/github-mcp-server"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "YOUR_PAT" },
      "tools": ["*"]
    },
    "stripe": {
      "type": "http",
      "url": "https://mcp.stripe.com",
      "headers": { "Authorization": "Bearer YOUR-TOKEN" },
      "tools": ["*"]
    }
  }
}
```

Fields: `type` (`"local"`, `"stdio"`, `"http"`, or `"sse"` — Copilot wants this explicit), `command`/`args`
(local), `env` (local), `url`/`headers` (remote), `tools` (array of tool names, `"*"` for all, `""`/empty
for none).

CLI: `copilot mcp add NAME -- COMMAND [ARGS...]` (local), `copilot mcp add --transport http NAME URL`
(remote), with `--env`, `--header`, `--transport`, `--tools`, `--timeout` options. Management:
`copilot mcp list [--json]`, `copilot mcp get NAME`, `copilot mcp remove NAME`. In-session: `/mcp add`,
`/mcp show [NAME]`, `/mcp edit NAME`, `/mcp delete NAME`, `/mcp enable|disable NAME`, and
`/mcp search [QUERY]` (needs `--experimental`).

### Already compatible?

**Yes, closely.** Claude Code's `.mcp.json` uses the same `mcpServers`-keyed object of
`{command,args,env}` or `{url,headers,...}` entries. The practical differences: Copilot wants an explicit
`type` field (`local`/`stdio`/`http`/`sse`) where Claude Code infers stdio-vs-remote from which keys are
present, and Copilot supports an optional `tools` allow-list per server that Claude Code's schema doesn't
have a documented equivalent for.

### Translation steps

1. Copy each entry from `.mcp.json`'s `mcpServers` object as-is into the Copilot-side file — same location
   name (`.mcp.json`) works for both tools simultaneously if the schemas stay compatible; only fork the
   file if `type`/`tools` fields need to differ per-tool.
2. Add an explicit `"type"` field to every entry if missing — `"local"` for anything with `command`/`args`,
   `"http"` or `"sse"` for anything with `url` (check which transport the server actually speaks; don't
   default-guess `"http"` for an `"sse"` server).
3. Leave secrets (tokens, keys in `env`/`headers`) where they already are; Copilot doesn't document a
   separate gitignored personal-override file the way some harnesses do — if the project `.mcp.json` is
   committed to git, keep secrets out of it and use `~/.copilot/mcp-config.json` (personal, not committed)
   for anything credential-bearing instead.
