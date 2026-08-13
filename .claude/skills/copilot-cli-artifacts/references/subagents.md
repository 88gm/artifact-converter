# Custom Agents (Subagents)

## Native format

Markdown files with `.agent.md` extension (required — not a bare `.md`). Filename derives from the agent
name (e.g., "Security expert" → `security-expert.agent.md`; lowercase letters and hyphens recommended for
programmatic use).

### Storage locations

| Scope | Location | Precedence |
|---|---|---|
| Project | `.github/agents/<name>.agent.md` | — |
| Personal | `~/.copilot/agents/<name>.agent.md` | **Wins** on a name collision with a project agent |

### Frontmatter / structure

Documented fields:

```yaml
---
name: security-expert          # display identifier; lowercase + hyphens recommended
description: >                  # states expertise and usage triggers, used for auto-inference
  Reviews code for security vulnerabilities and suggests fixes.
tools: [shell, edit]              # optional — restricts tool access; omit to allow everything by default
---

Behavioral guidelines, actions, and constraints — the agent's system prompt.
```

### Usage

Four ways to invoke: `/agent` (interactive picker), naming the agent explicitly in a prompt, automatic
inference from the `description`/trigger words, or programmatically via `--agent <name> --prompt "..."`.

Custom agents run with their own context window, populated independently of the main session — this lets
work happen without cluttering the primary agent's context, same underlying idea as Claude Code subagents.

## Already compatible?

**No.** Copilot CLI does not read `.claude/agents/` directly — there's no passthrough for subagent
definitions the way there is for `CLAUDE.md` or `.claude/skills/`. Every custom agent needs both a file
move and a rename (`.claude/agents/<name>.md` → `.github/agents/<name>.agent.md` or
`~/.copilot/agents/<name>.agent.md`).

## Translation steps (Claude Code subagent → Copilot custom agent)

1. Create `.github/agents/<name>.agent.md` (project) or `~/.copilot/agents/<name>.agent.md` (personal —
   remember this wins on a name collision with the project version, so don't create both unless the
   difference is intentional).
2. Frontmatter mapping:
   - `name`, `description` — copy as-is.
   - `tools` (Claude Code) → `tools` (Copilot) — same field name, but verify the value vocabulary: Claude
     Code uses capitalized built-in tool names (`Read`, `Edit`, `Bash`, ...); Copilot's tool vocabulary for
     this field isn't fully documented beyond examples like `shell`. Don't invent a full mapping table —
     carry over unambiguous entries (MCP tool names `mcp__server__tool` are stable across harnesses) and
     flag the rest for the user to confirm in a live session (`/agent` picker or docs).
   - Claude Code's `model` field has no documented Copilot custom-agent equivalent — omit it and mention
     that model selection may not be configurable per-agent in Copilot CLI the same way.
3. Copy the system-prompt body verbatim, adjusting any explicitly-named tools in the prose the same way as
   the frontmatter.
4. If other artifacts reference this subagent by name (a skill or hook that expects to invoke it), make
   sure the Copilot-side `name` matches exactly what those references expect, and update any invocation
   syntax to Copilot's (`/agent`, explicit naming, or `--agent name`) since Claude Code's `Agent` tool
   invocation doesn't carry over.
5. Validate with `/agent` in a Copilot CLI session, or `copilot --agent <name> --prompt "test"` from the
   shell.
