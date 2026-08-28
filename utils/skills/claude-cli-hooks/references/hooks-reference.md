# Claude Code Hooks — Full Reference

Verbatim-faithful condensation of https://code.claude.com/docs/en/hooks

## Table of contents

1. Overview & lifecycle
2. Configuration structure
3. Matcher patterns
4. Handler types (command, http, mcp_tool, prompt, agent)
5. Path placeholders & environment variables
6. Hook input JSON schema
7. Every event, with input additions and blocking behavior
8. Hook output — exit codes and JSON schema
9. Decision control per event
10. Example hooks
11. Security & permissions

---

## 1. Overview & lifecycle

Hooks execute automatically at specific points in Claude Code's lifecycle. They
receive JSON context via stdin (command hooks) or POST body (http hooks) and can
return decisions that block or modify actions.

Cadences:
- **Once per session**: `SessionStart`, `SessionEnd`
- **Once per turn**: `UserPromptSubmit`, `Stop`, `StopFailure`
- **Per tool call in the agentic loop**: `PreToolUse`, `PostToolUse`,
  `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`, `PostToolBatch`

Additional events: `UserPromptExpansion`, `SubagentStart`, `SubagentStop`,
`TaskCreated`, `TaskCompleted`, `TeammateIdle`, `PreCompact`, `PostCompact`,
`Notification`, `MessageDisplay`, `InstructionsLoaded`, `ConfigChange`,
`CwdChanged`, `DirectoryAdded`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`,
`Elicitation`, `ElicitationResult`, `Setup`.

---

## 2. Configuration structure

### Locations and scope

| Location | Scope | Shareable |
|---|---|---|
| `~/.claude/settings.json` | All projects | No |
| `.claude/settings.json` | Single project | Yes |
| `.claude/settings.local.json` | Single project | No |
| Managed policy settings | Organization-wide | Yes |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes |
| Skill/Subagent frontmatter | Current session/subagent | Yes |

### Schema

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolName|OtherTool",
        "hooks": [
          {
            "type": "command|http|mcp_tool|prompt|agent",
            "if": "Bash(rm *)",
            "timeout": 600,
            "statusMessage": "Custom message",
            "once": false
          }
        ]
      }
    ]
  },
  "disableAllHooks": false
}
```

---

## 3. Matcher patterns

| Matcher value | Evaluated as | Example |
|---|---|---|
| `"*"`, `""`, omitted | Match all | Fires on every occurrence |
| Letters, digits, `_`, `-`, spaces, `,`, `\|` | Exact string or list | `Bash`, `Edit\|Write`, `code-reviewer` |
| Other characters | JavaScript regex (unanchored) | `^Notebook`, `mcp__memory__.*` |

### Event-specific matchers

| Event | Filters on | Examples |
|---|---|---|
| `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied` | Tool name | `Bash`, `Edit\|Write`, `mcp__.*` |
| `SessionStart` | How session started | `startup`, `resume`, `clear`, `compact`, `fork` |
| `SessionEnd` | Why session ended | `clear`, `resume`, `logout`, `prompt_input_exit`, `other` |
| `Setup` | Which CLI flag triggered | `init`, `maintenance` |
| `Notification` | Notification type | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`, `agent_needs_input`, `agent_completed`, `quota_auto_resume_fired` |
| `SubagentStart`, `SubagentStop` | Agent type | `general-purpose`, `Explore`, `Plan`, custom agent names |
| `PreCompact`, `PostCompact` | What triggered compaction | `manual`, `auto` |
| `ConfigChange` | Configuration source | `user_settings`, `project_settings`, `local_settings`, `policy_settings`, `skills` |
| `DirectoryAdded` | How directory added | `slash_command`, `register_repo_root` |
| `FileChanged` | Literal filenames | `.envrc\|.env` |
| `StopFailure` | Error type | `rate_limit`, `overloaded`, `authentication_failed`, `billing_error`, `invalid_request`, `model_not_found`, `server_error`, `max_output_tokens` |
| `InstructionsLoaded` | Load reason | `session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact` |
| `UserPromptExpansion` | Command name | Skill or command names |
| `Elicitation`, `ElicitationResult` | MCP server name | Configured MCP server names |
| No matcher support | Always fires | `UserPromptSubmit`, `PostToolBatch`, `Stop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `WorktreeCreate`, `WorktreeRemove`, `MessageDisplay`, `CwdChanged` |

### MCP tool matching

MCP tools appear as `mcp__<server>__<tool>`:
- `mcp__memory__create_entities` — specific tool
- `mcp__memory__.*` — all tools from server
- `mcp__brave-search__.*` — server with hyphen
- `mcp__.*__write.*` — any write tool from any server
- Plugin-scoped: `mcp__plugin_<plugin-name>_<server-name>__<tool>`

---

## 4. Handler types

### Common fields (all types)

| Field | Required | Type | Description |
|---|---|---|---|
| `type` | Yes | String | `command`, `http`, `mcp_tool`, `prompt`, or `agent` |
| `if` | No | String | Permission rule filter (tool events only): `"Bash(rm *)"`, `"Edit(*.ts)"` |
| `timeout` | No | Number | Seconds before canceling (default: 600 command/http/mcp_tool, 30 prompt, 60 agent) |
| `statusMessage` | No | String | Custom spinner message |
| `once` | No | Boolean | Remove hook after first success (skill frontmatter only) |

### Command hooks

```json
{
  "type": "command",
  "command": "/path/to/script.sh",
  "args": [],
  "async": false,
  "asyncRewake": false,
  "shell": "bash"
}
```

| Field | Required | Description |
|---|---|---|
| `command` | Yes | Shell command or executable name |
| `args` | No | Argument list; when present, uses exec form (no shell) |
| `async` | No | Run in background without blocking |
| `asyncRewake` | No | Background run, wake Claude on exit code 2 |
| `shell` | No | `"bash"` or `"powershell"` (defaults to bash) |

- **Exec form** (`args` present): no shell; each arg is exactly one argument; path
  placeholders substituted as strings.
- **Shell form** (`args` absent): shell tokenizes, expands variables, interprets
  pipes / `&&` / globs.

### HTTP hooks

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/pre-tool-use",
  "headers": { "Authorization": "Bearer $MY_TOKEN" },
  "allowedEnvVars": ["MY_TOKEN"]
}
```

Sends hook JSON as POST body with `Content-Type: application/json`. `headers`
supports `$VAR` interpolation; `allowedEnvVars` whitelists which vars may appear.

### MCP tool hooks

```json
{
  "type": "mcp_tool",
  "server": "my_server",
  "tool": "security_scan",
  "input": { "file_path": "${tool_input.file_path}" }
}
```

`server` is a configured MCP server name or `plugin:<plugin-name>:<server-name>`.
`input` supports `${path}` substitution from the hook's JSON input.

### Prompt hooks

```json
{
  "type": "prompt",
  "prompt": "Analyze this command: $ARGUMENTS",
  "model": "fast-model"
}
```

`$ARGUMENTS` is replaced with the hook input JSON. `model` defaults to the fast model.

### Agent hooks

```json
{ "type": "agent", "prompt": "Verify this operation: $ARGUMENTS" }
```

Experimental subagent-based hooks with tool access.

---

## 5. Path placeholders & environment variables

| Placeholder | Description |
|---|---|
| `${CLAUDE_PROJECT_DIR}` | Project root |
| `${CLAUDE_PLUGIN_ROOT}` | Plugin installation directory |
| `${CLAUDE_PLUGIN_DATA}` | Plugin persistent data directory |

Also exported as environment variables on spawned processes. Additional env vars
available to hooks:
- `$CLAUDE_PROJECT_DIR`, `$CLAUDE_PLUGIN_ROOT`, `$CLAUDE_PLUGIN_DATA`
- `$CLAUDE_EFFORT` — current effort level
- `$CLAUDE_CODE_REMOTE` — `"true"` in web environments
- `$CLAUDE_CODE_BRIDGE_SESSION_ID` — Remote Control session ID (v2.1.199+)

---

## 6. Hook input JSON schema

### Common input fields (all events)

```json
{
  "session_id": "abc123",
  "prompt_id": "550e8400-e29b-41d4-a716-446655440000",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default|plan|acceptEdits|auto|dontAsk|bypassPermissions",
  "effort": { "level": "low|medium|high|xhigh|max" },
  "hook_event_name": "EventName",
  "agent_id": "subagent-id",
  "agent_type": "general-purpose"
}
```

| Field | Description |
|---|---|
| `session_id` | Current session identifier |
| `prompt_id` | UUID matching OpenTelemetry events for correlation |
| `transcript_path` | Path to conversation JSON (may lag current turn) |
| `cwd` | Current working directory |
| `permission_mode` | Current permission mode; `"default"` for Manual mode |
| `effort` | Object with `level` field |
| `hook_event_name` | Event name that fired |
| `agent_id` | Subagent identifier (subagent hooks only) |
| `agent_type` | Agent name (subagent or session `--agent`) |

### Tool event input additions

```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test",
    "description": "Run test suite",
    "timeout": 120000,
    "run_in_background": false
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

---

## 7. Events

### PreToolUse
Before a tool call executes. Can block it.
Input adds: `tool_name`, `tool_input`, `tool_use_id`.
JSON decision control:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "...",
    "updatedInput": { "command": "..." }
  }
}
```
Exit code 2: blocks the tool call.

### PostToolUse
After a tool call succeeds. Input adds: `tool_name`, `tool_input`, `tool_use_id`,
`tool_output`. Exit code 2: shows stderr to Claude; tool already ran.

### PostToolUseFailure
After a tool call fails. Input adds: `tool_name`, `tool_input`, `tool_use_id`,
`tool_error`. Exit code 2: shows stderr to Claude; tool already failed.

### PostToolBatch
After a full batch of parallel tool calls resolves, before the next model call.
Input adds `tool_calls` array of `{tool_name, tool_input, tool_use_id}`.
Exit code 2: stops agentic loop before next model call.

### PermissionRequest
When a tool call needs a permission decision. Input adds `tool_name`,
`tool_input`, `tool_use_id`, `permission_rules`, `matching_rule`.
JSON: `{"hookSpecificOutput": {"hookEventName": "PermissionRequest", "decision": "allow|deny|ask"}}`.
Exit code 2 ignored — use the JSON `decision`.

### PermissionDenied
When auto mode denies a tool call. Input adds `tool_name`, `tool_input`,
`tool_use_id`, `denial_reason`, `classifier_verdict` (`deny|unknown`).
JSON: `{"hookSpecificOutput": {"hookEventName": "PermissionDenied", "retry": true}}`.
Exit code and stderr ignored.

### UserPromptSubmit
When user submits a prompt, before Claude processes it. Input adds `prompt`.
Exit code 2: blocks prompt processing and erases the prompt. stdout is added as
context Claude sees.

### UserPromptExpansion
When a user-typed command expands into a prompt. Input adds `command_name`,
`command_input`, `expanded_prompt`. Exit code 2: blocks the expansion.

### Stop
When Claude finishes responding. Input adds `last_assistant_message`,
`stop_reason` (`end_turn|max_tokens`). Exit code 2: prevents Claude from
stopping; continues the conversation.

### SubagentStop
When a subagent finishes. Input adds `last_assistant_message`, `stop_reason`
(`end_turn|max_tokens|user_cancelled`). Exit code 2: prevents subagent stopping.

### SessionStart
Session begins or resumes. Matchers: `startup`, `resume`, `clear`, `compact`,
`fork`. Exit code 2: shows stderr to user only; session proceeds. stdout adds
context.

### SessionEnd
Session terminates. Matchers: `clear`, `resume`, `logout`, `prompt_input_exit`,
`other`. Input adds `end_reason`. Exit code and stderr shown to user only.

### Setup
Starting with `--init-only`, `--init`, or `--maintenance` in `-p` mode. Matchers:
`init`, `maintenance`. Exit code and output ignored (one-time setup only).

### Notification
When Claude Code sends a notification. Matchers: `permission_prompt`,
`idle_prompt`, `auth_success`, `elicitation_dialog`, `elicitation_url_dialog`,
`elicitation_complete`, `elicitation_response`, `agent_needs_input`,
`agent_completed`, `quota_auto_resume_fired`, `quota_auto_resume_stale`,
`quota_auto_resume_disabled`. Input adds `notification_type`, `message`. Exit
code and output ignored.

### SubagentStart
When a subagent is spawned. Matchers: agent type. Input adds `agent_type`.
Exit code 2 shows stderr to user only; subagent starts.

### TaskCreated
Task created via `TaskCreate`. Exit code 2: rolls back task creation.

### TaskCompleted
Task marked completed. Exit code 2: prevents marking as completed.

### TeammateIdle
Agent team teammate about to go idle. Exit code 2: prevents going idle.

### PreCompact
Before context compaction. Matchers: `manual`, `auto`. Exit code 2: blocks
compaction.

### PostCompact
After context compaction completes. Exit code 2: shows stderr to user only.

### CwdChanged
Working directory changes (e.g. `cd`). No matcher support.

### DirectoryAdded
Directory added mid-session via `/add-dir` or SDK `register_repo_root`. Matchers:
`slash_command`, `register_repo_root`.

### FileChanged
Watched file changes on disk. Matcher: literal filenames (`.envrc|.env`).

### InstructionsLoaded
`CLAUDE.md` or `.claude/rules/*.md` loaded into context. Matchers:
`session_start`, `nested_traversal`, `path_glob_match`, `include`, `compact`.

### ConfigChange
Configuration file changes during session. Matchers: `user_settings`,
`project_settings`, `local_settings`, `policy_settings`, `skills`. Exit code 2:
blocks the change from taking effect (except `policy_settings`).

### WorktreeCreate
Worktree created via `--worktree`, `isolation: "worktree"`, or background
session. Any non-zero exit aborts worktree creation.

### WorktreeRemove
Worktree removed at session exit, subagent finish, or background session delete.
Failures logged in debug mode only.

### Elicitation
MCP server requests user input during a tool call. Matcher: MCP server name.
Exit code 2: denies the elicitation.

### ElicitationResult
After user responds to an MCP elicitation, before response sent to server.
Matcher: MCP server name. Exit code 2: blocks the response (action becomes
decline).

### MessageDisplay
While assistant message text is displayed. Display-only. No matcher support. Exit
code and output ignored.

---

## 8. Hook output — exit codes and JSON

### Exit code behavior

| Code | Meaning | Blocks? | Reads JSON? |
|---|---|---|---|
| 0 | Success | No | Yes, if valid |
| 1 | Non-blocking error | No | Yes, if valid |
| 2 | Blocking error | Yes* | Yes |
| Other | Non-blocking error | No | Yes, if valid |

\* Exit 2 blocks only on events that support blocking (see each event above).

### JSON output schema

```json
{
  "hookSpecificOutput": {
    "hookEventName": "EventName",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Human-readable reason",
    "decision": "allow|deny|ask",
    "continue": true,
    "stopReason": "reason",
    "additionalContext": "Text for Claude",
    "retry": true,
    "updatedInput": { "field": "value" }
  },
  "systemMessage": "Message for Claude",
  "terminalSequence": "[2J"
}
```

| Field | Events | Description |
|---|---|---|
| `hookEventName` | All | Event that fired |
| `permissionDecision` | Tool events | `allow`, `deny`, `ask` |
| `permissionDecisionReason` | Tool events | Reason for decision |
| `decision` | `PermissionRequest` | `allow`, `deny`, `ask` |
| `continue` | `Stop`, `SubagentStop` | Override stopping |
| `stopReason` | `Stop`, `SubagentStop` | Why continuing |
| `additionalContext` | Multiple | Context for Claude |
| `retry` | `PermissionDenied` | Allow model to retry |
| `updatedInput` | `PreToolUse` | Modified tool input |
| `systemMessage` | Most events | Message to Claude (varies by event) |
| `terminalSequence` | Most events | Terminal escape sequences |

---

## 9. Decision control per event

| Event | Fields honored |
|---|---|
| `PreToolUse` | `permissionDecision`, `permissionDecisionReason`, `updatedInput`, `additionalContext`, `systemMessage` |
| `PermissionRequest` | `decision` |
| `PermissionDenied` | `retry` |
| `UserPromptSubmit` | `additionalContext`, `systemMessage` |
| `UserPromptExpansion` | `additionalContext`, `systemMessage` |
| `PostToolUse`, `PostToolUseFailure` | `additionalContext`, `systemMessage` |
| `Stop`, `SubagentStop` | `continue`, `stopReason`, `additionalContext`, `systemMessage` |
| `PostToolBatch` | `continue`, `additionalContext`, `systemMessage` |
| Other blocking events | Event-specific fields |

---

## 10. Example hooks

### Block destructive bash commands

```bash
#!/bin/bash
# .claude/hooks/block-rm.sh
COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked by hook"
    }
  }'
else
  exit 0
fi
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

### HTTP hook with authentication

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "http",
            "url": "http://localhost:8080/hooks/pre-tool-use",
            "timeout": 30,
            "headers": { "Authorization": "Bearer $MY_TOKEN" },
            "allowedEnvVars": ["MY_TOKEN"]
          }
        ]
      }
    ]
  }
}
```

### MCP tool hook for security scanning

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "mcp_tool",
            "server": "my_server",
            "tool": "security_scan",
            "input": { "file_path": "${tool_input.file_path}" }
          }
        ]
      }
    ]
  }
}
```

### Prompt-based hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review this command for safety: $ARGUMENTS",
            "model": "fast-model",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### Linting after file writes

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [ { "type": "command", "command": "/path/to/lint-check.sh" } ] }
    ]
  }
}
```

### Skill with hooks (frontmatter)

```yaml
---
name: secure-operations
description: Perform operations with security checks
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

### PowerShell / Windows note

Set `"shell": "powershell"` on the handler, or point `command` at a `.ps1` /
`.cmd` file. Default shell is bash. `jq` may not be present on Windows — parse the
stdin JSON in PowerShell with `$input | ConvertFrom-Json` or `ConvertFrom-Json (
[Console]::In.ReadToEnd() )`.

---

## 11. Security & permissions

- **Workspace trust**: hooks in `.claude/settings.json` require workspace trust
  before running.
- **Managed hooks**: administrators can restrict hooks to managed policy settings
  only via `allowManagedHooksOnly`.
- **HTTP allowlists**: `allowedHttpHookUrls` (whitelist URLs),
  `httpHookAllowedEnvVars` (whitelist env vars).
- **Disable hooks**: `"disableAllHooks": true` in settings, or
  `--settings '{"disableAllHooks": true}'`.
- Hooks run with your user's full permissions. Review any hook command before
  adding it; validate and quote shell inputs; use absolute paths; avoid running
  on sensitive paths (`.env`, `.git/`, keys).

## Hooks menu

Type `/hooks` in Claude Code to browse configured hooks read-only. Shows event,
matcher, type, source, and handler details.
