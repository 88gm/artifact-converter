# Skills (SKILL.md)

## Native format (Claude Code)

`.claude/skills/<name>/SKILL.md` — directory named for the invocation identifier (`/<name>`), containing
a `SKILL.md` with YAML frontmatter (`name`, `description`, optionally `allowed-tools` using Claude Code's
capitalized tool names: `Read`, `Edit`, `Bash`, `Grep`, `Glob`, `WebFetch`, `WebSearch`, plus MCP tool
names `mcp__server__tool` verbatim) and a markdown body as the injected prompt. Optional `scripts/`,
`references/`, `assets/` subdirectories. Personal copy at `~/.claude/skills/<name>/SKILL.md`.

## From Copilot CLI

Source: `.github/skills/<name>/SKILL.md` (also `~/.copilot/skills/<name>/SKILL.md` personal, or
`.agents/skills/<name>/SKILL.md`).

### Already compatible?

**Structurally, yes — the core format is the same convention Claude Code originated** (named directory +
`SKILL.md` + YAML frontmatter + markdown body + optional `scripts/`/`references/`/`assets/`). But there is
no passthrough the other way: Claude Code only scans `.claude/skills/` (and its personal equivalent), not
`.github/skills/` or `.agents/skills/`. A directory move is still required, even though nothing inside it
needs a rewrite in most cases.

### Translation steps

1. Copy (or move) the directory to `.claude/skills/<name>/` — `scripts/`, `references/`, `assets/`
   subdirectories carry over unchanged.
2. Frontmatter: `name`, `description` copy as-is.
3. `tools`/`allowed-tools`: Copilot's tool vocabulary for this field isn't fully documented beyond
   examples like `shell`/`bash` categories. Don't invent a mapping table — carry over unambiguous MCP tool
   names (`mcp__server__tool`) unchanged, and flag any Copilot-specific category names for the user to
   confirm and translate to Claude Code's capitalized names (`Read`, `Edit`, `Bash`, `Grep`, `Glob`,
   `WebFetch`, `WebSearch`) themselves rather than guessing.
4. Copy the markdown body literally; only touch prose that explicitly names a Copilot tool.
5. Validate with `/name` in a Claude Code session.

## From DevIn CLI

Source: `.devin/skills/<name>/SKILL.md` (also `.windsurf/skills/`, `.agents/skills/`; personal
`~/.config/devin/skills/<name>/SKILL.md`). Frontmatter can include `argument-hint`, `model`, `subagent`,
`agent`, `allowed-tools` (lowercase: `read`, `edit`, `grep`, `glob`, `exec`, plus `mcp__server__tool`),
`permissions` (`allow`/`deny`/`ask`), `triggers` (`user`, `model`).

### Already compatible?

**No — always needs translation.** Claude Code has no passthrough for `.devin/skills/`.

The good news: the core directory/file convention is identical, so bundled resources need no changes —
only frontmatter and the file's new home directory.

### Translation steps

1. Create `.claude/skills/<name>/SKILL.md` mirroring the source directory structure — copy `scripts/`,
   `references/`, `assets/` unchanged.
2. Frontmatter field mapping:
   - `name`, `description` — copy as-is.
   - `allowed-tools` — remap DevIn's lowercase names to Claude Code's capitalized ones: `read`→`Read`,
     `edit`→`Edit`, `grep`→`Grep`, `glob`→`Glob`, `exec`→`Bash`. MCP tool names (`mcp__server__tool`)
     carry over unchanged.
   - `argument-hint`, `model`, `permissions`, `triggers` — **no Claude Code equivalent, drop them.** If any
     were load-bearing (e.g. `triggers: [user]` preventing autonomous invocation, or `permissions.deny`
     blocking a destructive action), tell the user explicitly that this protection isn't carried over
     rather than silently dropping it — Claude Code's closest lever is narrowing the `description` so the
     model is less likely to invoke it autonomously, which is a much weaker guarantee.
   - `subagent: true` / `agent: <profile>` — Claude Code has no frontmatter field for this. If the DevIn
     skill delegates to an isolated worker or a specific subagent profile, express that in the markdown
     body as an explicit instruction to use the `Agent` tool (optionally naming a `.claude/agents/<name>.md`
     subagent — see [subagents.md](subagents.md)) rather than declaring it in frontmatter.
3. Copy the markdown body literally; only adjust prose that explicitly names a DevIn tool (e.g. "use the
   `exec` tool" → "use the `Bash` tool").
4. Validate with `/name` in a Claude Code session.
