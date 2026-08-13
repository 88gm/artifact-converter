# Hooks

## Native format

Two hook types:

- **`command`** — a shell command that receives JSON event data on stdin and can block or modify
  behavior via stdout/exit code.
- **`prompt`** — an LLM evaluates a prompt against the event and makes the decision, instead of a script.

### File locations

**Project-level** (DevIn searches from the working directory upward):

| File | Notes |
|---|---|
| `.devin/hooks.v1.json` | Recommended — standalone, versioned hook file. |
| `.devin/config.json` under `"hooks"` key | Alternative, if hooks live alongside other project config. |
| `.devin/config.local.json` under `"hooks"` key | Gitignored, personal-only hooks. |
| `.claude/settings.json` under `"hooks"` key | Read directly — see "Already compatible?" below. |

**User-level** (global): `~/.config/devin/config.json` (or Windows `%APPDATA%\devin\config.json`),
`~/.claude.json`, `~/.claude/settings.json`.

### Config schema

```json
{
  "PreToolUse": [
    {
      "matcher": "^exec$",
      "hooks": [
        { "type": "command", "command": "./scripts/check-command.sh", "timeout": 10 }
      ]
    }
  ]
}
```

- `matcher`: regex against the tool name; empty or omitted matches every tool. Non-tool events
  (`UserPromptSubmit`, `Stop`, `SessionStart`, `SessionEnd`, `PostCompaction`) generally omit `matcher`.
- Each event maps to an array of `{matcher, hooks}` groups; each group's `hooks` array runs in order.
- `type: "command"` needs a `command` (shell command/path); `type: "prompt"` needs a `prompt` (LLM
  instruction text) instead.

### Events, stdin payload, and purpose

| Event | Fires | Key stdin fields |
|---|---|---|
| `PreToolUse` | Before tool execution | `tool_name`, `tool_input` |
| `PostToolUse` | After tool completes | `tool_name`, `tool_input`, `tool_response` |
| `PermissionRequest` | Agent needs permission | `tool_name`, `tool_input` |
| `UserPromptSubmit` | User sends a message | `prompt` |
| `Stop` | Agent wants to end its turn | `stop_hook_active` |
| `PostCompaction` | After context compaction | `summary` (may be null) |
| `SessionStart` | Session begins | `source` |
| `SessionEnd` | Session ends | `reason` |

All events also include `session_id` and `prompt_id`.

### Command hook output (stdout)

Block/allow decision:
```json
{"decision": "block", "reason": "Policy violation"}
```

Inject context (mainly for `UserPromptSubmit`, `SessionStart`, `PostCompaction`):
```json
{"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": "..."}}
```

Rewrite tool input (`PreToolUse` only):
```json
{"hookSpecificOutput": {"hookEventName": "PreToolUse", "updatedInput": {"command": "modified"}}}
```

Exit codes: `0` success, `2` block the action, anything else is logged as an error but non-blocking.

`Stop` hooks that repeatedly return `{"decision": "block"}` risk an infinite loop — mention this if the
user is writing a Stop hook that re-blocks based on unchanged conditions.

## Already compatible?

**Yes.** DevIn reads `.claude/settings.json`'s `"hooks"` key directly, using the same event names and
JSON shape (`PreToolUse`, `PostToolUse`, etc. with `matcher`/`hooks` arrays) — this is a genuinely shared
schema, not just an importer. A Claude Code hooks config generally works unchanged *for events and
structure*.

**The one thing that does NOT carry over:** `matcher` regexes written against Claude Code's tool names
(`Bash`, `Read`, `Edit`, `Write`, `WebFetch`, ...) will silently fail to match anything in DevIn, because
DevIn's tool names are different (`exec`, `read`, `edit`, ...) — see the mapping in skills.md. A hook that
matched `"^Bash$"` in Claude Code needs `"^exec$"` to actually fire under DevIn. This is the most common
silent breakage when porting hooks — always check every `matcher` value, even when the file "already
works" because DevIn parses it without error.

## Translation steps (only needed if not relying on the `.claude/settings.json` passthrough)

1. Create `.devin/hooks.v1.json` with the same top-level event structure.
2. Rewrite every `matcher` regex to target DevIn tool names instead of Claude Code ones.
3. `command` hooks: shell scripts themselves usually need no changes since the stdin JSON shape
   (`tool_name`, `tool_input`, etc.) matches — but if the script branches on `tool_name` values
   (`"Bash"`, `"Read"`), those string comparisons need the same tool-name remap as the matcher.
4. `prompt` hooks: no changes needed beyond the event name, since these are evaluated by an LLM against
   the same stdin fields.
5. Confirm hooks accumulate rather than override across config levels (project + project-local + user +
   the `.claude/settings.json` passthrough all fire together) — don't assume writing a `.devin/hooks.v1.json`
   silently disables the `.claude/settings.json` ones; both will run unless the Claude hooks are removed
   or `read_config_from.claude` is disabled.
