# Subagents

## Native format

Markdown files with YAML frontmatter, in one of two layouts:

**Project**: `.devin/agents/<name>.md` (flat) or `.devin/agents/<name>/AGENT.md` (directory form —
useful if the subagent needs bundled resources alongside it). Also recognized under `.agents/agents/`.

**Global**: `~/.config/devin/agents/` on Linux/macOS, `%APPDATA%\devin\agents\` on Windows.

### Frontmatter schema

```yaml
---
name: reviewer
description: Reviews code for correctness and style
model: sonnet              # optional, overrides default subagent model
allowed-tools:               # optional, lowercase DevIn tool names — see skills.md for the mapping
  - read
  - grep
  - glob
  - exec
max-nesting: 1                # optional, permits this subagent to itself spawn nested subagents
---

You are a code review subagent. Focus on correctness, security, style, and performance.
Cite specific file paths and line numbers in findings.
```

The body is the subagent's system prompt.

Subagents run **foreground** (parent pauses and waits) or **background** (parent continues in parallel),
share tools and codebase context with the parent, but do not inherit its conversation history. Per DevIn's
docs, nesting subagents beyond one level is not supported by default — `max-nesting` exists specifically
to opt a profile into deeper nesting when genuinely needed.

## Already compatible?

**No.** DevIn does not read `.claude/agents/` directly — there's no passthrough for subagent definitions
the way there is for CLAUDE.md rules or `.claude/settings.json` hooks. Every custom subagent needs an
explicit DevIn copy.

## Translation steps (Claude Code subagent → DevIn subagent)

1. Create `.devin/agents/<name>.md` (or the `<name>/AGENT.md` directory form if the subagent needs
   bundled files).
2. Frontmatter mapping:
   - `name`, `description` — copy as-is.
   - `model` — copy as-is (model identifiers are generally shared vocabulary across the docs, but verify
     the specific model string is one DevIn recognizes if unsure).
   - `tools` (Claude Code) → `allowed-tools` (DevIn) — rename the field *and* remap the values using the
     same lowercase mapping as skills (`Read`→`read`, `Edit`→`edit`, `Grep`→`grep`, `Glob`→`glob`,
     `Bash`→`exec`; MCP tool names unchanged).
   - Claude Code has no `max-nesting` equivalent — only add it if the subagent's prompt explicitly
     delegates to further subagents and that's an intentional design, not an accident of translation.
3. Copy the system-prompt body verbatim, adjusting any tool names mentioned in the prose the same way as
   the frontmatter.
4. If other skills or rules in the project invoke this subagent by name (e.g., a skill with
   `agent: reviewer`), make sure the DevIn-side `name` field matches exactly what those references expect.
