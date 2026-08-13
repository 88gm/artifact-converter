# Subagents / Custom Agents

## Native format (Claude Code)

`.claude/agents/<name>.md` — markdown with YAML frontmatter (`name`, `description`, `model`, `tools` using
Claude Code's capitalized tool names) and a body that becomes the subagent's system prompt. Invoked via
the `Agent` tool, either automatically (based on `description`) or by explicit reference.

## From Copilot CLI

Source: `.github/agents/<name>.agent.md` (project) or `~/.copilot/agents/<name>.agent.md` (personal —
wins on name collision with the project version). The `.agent.md` extension is required on the Copilot
side, not a plain `.md`.

### Already compatible?

**No.** No passthrough for `.github/agents/`. Needs a move + rename in the opposite direction:
`<name>.agent.md` → `.claude/agents/<name>.md` (drop the `.agent` infix, plain `.md`).

### Translation steps

1. Create `.claude/agents/<name>.md`.
2. Frontmatter: `name`, `description` copy as-is.
3. `tools` (Copilot) → `tools` (Claude Code) — same field name, but Copilot's value vocabulary isn't fully
   documented beyond category examples like `shell`. Don't invent a full mapping — carry over unambiguous
   MCP tool names (`mcp__server__tool`) unchanged, and flag anything else for the user to confirm and
   convert to Claude Code's capitalized names (`Read`, `Edit`, `Bash`, `Grep`, `Glob`, `WebFetch`,
   `WebSearch`) themselves.
4. `model` — Copilot custom agents don't document a per-agent model override field, so there's nothing to
   carry over. If the user wants a specific model for the Claude Code subagent, that's a new choice to make,
   not a translation — ask rather than defaulting silently.
5. Copy the system-prompt body literally, adjusting any tool names named explicitly in the prose.
6. If other artifacts invoke this subagent by name, update the invocation syntax to Claude Code's (the
   `Agent` tool with this subagent's `name`) — Copilot's `/agent`, explicit name, or `--agent` invocation
   syntax isn't inherited.
7. Validate via the `Agent` tool in a Claude Code session.

## From DevIn CLI

Source: `.devin/agents/<name>.md` (flat form) or `.devin/agents/<name>/AGENT.md` (directory form, used
when the subagent needs bundled resources alongside it). Also recognized at `.agents/agents/`. Personal:
`~/.config/devin/agents/`.

### Already compatible?

**No.** No passthrough for `.devin/agents/`.

### Translation steps

1. Create `.claude/agents/<name>.md`. If the source used the directory form (`<name>/AGENT.md`) with
   bundled resource files alongside it, flag this to the user — Claude Code subagents are single files
   with no documented convention for packaged resources the way skills have `scripts/`/`references/`/
   `assets/`; bundled content usually needs to be inlined into the prompt body or moved to a skill instead.
2. Frontmatter field mapping:
   - `name`, `description` — copy as-is.
   - `model` — copy directly; model name vocabulary is generally shared, but verify the specific string is
     recognized by Claude Code if in doubt.
   - `allowed-tools` (DevIn) → `tools` (Claude Code) — **rename the field** and remap values from
     lowercase to capitalized: `read`→`Read`, `edit`→`Edit`, `grep`→`Grep`, `glob`→`Glob`, `exec`→`Bash`;
     MCP names unchanged.
   - `max-nesting` — no Claude Code equivalent; drop it. If the source subagent explicitly delegated to
     further subagents and relied on a nesting limit for safety, flag this to the user rather than
     assuming Claude Code's `Agent` tool nesting behaves the same way by default.
3. Copy the system-prompt body literally, adjusting tool names named explicitly in the prose.
4. If other project artifacts (skills, rules) invoke this subagent by name, ensure the Claude Code side's
   `name` matches those references exactly.
5. Validate via the `Agent` tool in a Claude Code session.

## Key difference between the two sources

| | From Copilot CLI | From DevIn CLI |
|---|---|---|
| Source file extension | `.agent.md` (drop the `.agent` infix on the way in) | Plain `.md` or `AGENT.md` in a subdirectory (may need flattening) |
| Tools field | `tools` (same name, vocabulary uncertain — flag for user) | `allowed-tools` (rename to `tools`, values deterministically mapped) |
| `model` field | Not present on the source side — new choice, not a translation | Present, copy directly |
| Nesting | Not documented on the source side | `max-nesting` present, no Claude Code equivalent — drop and flag if load-bearing |
