---
name: devin-cli-hooks
description: >-
  Authoritative reference for creating, configuring, and debugging Devin CLI hooks.
  Use whenever the user wants to add, edit, review, or troubleshoot a Devin CLI
  hook, write a hooks.v1.json / .devin/config.json hooks block, block or rewrite
  tool calls, inject context on prompt/session start, gate permissions, or control
  when the agent stops. Triggers: "Devin hook", "hooks.v1.json", ".devin/hooks",
  "PreToolUse", "PostToolUse", "PermissionRequest", "UserPromptSubmit", "Stop hook",
  "SessionStart hook", "PostCompaction".
---

# Devin CLI Hooks

Hooks run custom logic in response to events in the Devin CLI agent lifecycle.
Use them to enforce policies, inject context, log actions, modify permissions,
or integrate with external systems. Hooks configured for Claude Code in
`.claude/` are also picked up automatically.

Source: https://docs.devin.ai/cli/extensibility/hooks/overview

## Workflow for a hook request

1. Identify the **event** (see table below) and whether a **tool matcher** is needed.
2. Choose hook **type**: `command` (shell script, deterministic) or `prompt` (LLM evaluation).
3. Write config to `.devin/hooks.v1.json` (preferred) — see "Configuration".
4. For `command` hooks, write the script: read JSON from **stdin**, emit control JSON to **stdout**, use **exit codes** to signal outcome.
5. Tell the user to run `/hooks` in Devin CLI to verify the hook loaded and see its source file.

## Events

| Event | Fires | Key input fields | Can block? | Special output |
|---|---|---|---|---|
| `PreToolUse` | before a tool runs | `tool_name`, `tool_input` | yes (`block`/exit 2) | `updatedInput` (rewrite tool args) |
| `PostToolUse` | after a tool completes | `tool_name`, `tool_input`, `tool_response{success,output,error}` | no | `additionalContext` |
| `PermissionRequest` | agent needs a permission decision | `tool_name`, `tool_input` | yes | `decision: approve` / `deny` |
| `UserPromptSubmit` | user submits a message | `prompt` | yes | `additionalContext` (inject into context) |
| `Stop` | agent decides to end its turn | `stop_hook_active` (bool) | yes — `block` forces agent to continue | `reason` |
| `PostCompaction` | after context compaction succeeds | `summary` (may be null) | no | `additionalContext` |
| `SessionStart` | new session begins | `source` | no | `additionalContext` |
| `SessionEnd` | session terminates | `reason` | no | — |

⚠️ A `Stop` hook that blocks without an eventually-satisfiable condition will loop the agent. Always make the block condition converge.

## Configuration

`.devin/hooks.v1.json` — the hooks object **is the entire file**:

```json
{
  "PreToolUse": [
    {
      "matcher": "exec",
      "hooks": [
        { "type": "command", "command": "./scripts/check-command.sh", "timeout": 10 }
      ]
    }
  ]
}
```

Other locations nest the same object under a `"hooks"` key:

- **Project** (searched from working dir up to repo root):
  `.devin/hooks.v1.json`, `.devin/config.json`, `.devin/config.local.json` (gitignored local override),
  `.claude/settings.json`, `.claude/settings.local.json`
- **User / global**: `~/.config/devin/config.json` (Windows: `%APPDATA%\devin\config.json`),
  `~/.claude.json`, `~/.claude/settings.json`
  Note: the configuration reference states `hooks` is a **project-level** setting; prefer `.devin/hooks.v1.json`.

### Entry schema

| Field | Meaning |
|---|---|
| `matcher` | regex tested against `tool_name`; empty or omitted = match all. Non-tool events (`UserPromptSubmit`, `Stop`, `Session*`, `PostCompaction`) use `""` or omit. |
| `hooks[]` | list of hooks to run for the match |
| `hooks[].type` | `"command"` or `"prompt"` |
| `hooks[].command` | shell command (for `type: command`) |
| `hooks[].prompt` | LLM prompt (for `type: prompt`) |
| `hooks[].timeout` | optional, seconds |

Matcher examples: `"exec"` (contains exec), `"^exec$"` (exactly exec), `"^mcp__github__.*"` (GitHub MCP tools).

## Command hook I/O

**Stdin** (JSON): `hook_event_name`, `tool_name`, `tool_input`, `session_id`, `prompt_id` (rotates per user turn; absent before first prompt), plus event-specific fields.
Env var `DEVIN_PROJECT_DIR` points to the project root.

**Stdout** (JSON, all optional):

```json
{ "decision": "block", "reason": "shown to the agent" }
```

```json
{ "hookSpecificOutput": { "hookEventName": "UserPromptSubmit", "additionalContext": "..." } }
```

```json
{ "hookSpecificOutput": { "hookEventName": "PreToolUse", "updatedInput": { "command": "safe-cmd" } } }
```

| Output field | Applies to | Effect |
|---|---|---|
| `decision` | blocking events | `"approve"` allows, `"block"` denies |
| `reason` | blocking events | message shown to agent |
| `additionalContext` | UserPromptSubmit, SessionStart, PostToolUse, PostCompaction | injected into agent context |
| `updatedInput` | PreToolUse | merged into tool arguments |

**Exit codes**: `0` = success (stdout JSON honored) · `2` = block the action · anything else = error, logged but **not** blocking.

## More examples

See `references/examples.md` for ready-to-use hook scripts (block `rm -rf`, log shell
commands, auto-approve git, inject deploy policy, setup script on session start).
