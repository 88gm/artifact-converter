---
name: claude-cli-hooks
description: >-
  Authoritative reference for creating, configuring, and debugging Claude Code
  (Claude CLI) hooks. Use this skill whenever the user wants to add, edit,
  review, or troubleshoot a Claude Code hook; write a `hooks` block in
  settings.json / settings.local.json / plugin hooks.json / skill frontmatter;
  block, allow, or rewrite tool calls; inject context on prompt or session
  start; auto-run linters/formatters/tests after edits; gate permissions; or
  control when Claude stops. Trigger even if the user does not say the word
  "hook" but describes automated behavior like "every time Claude edits a file,
  run X", "before running any bash command, check Y", "when the session starts,
  load Z". Triggers include: "Claude Code hook", "PreToolUse", "PostToolUse",
  "UserPromptSubmit", "SessionStart hook", "Stop hook", "PreCompact",
  "Notification hook", "hookSpecificOutput", "permissionDecision",
  "CLAUDE_PROJECT_DIR", `/hooks`.
---

# Claude Code Hooks

Hooks are user-defined shell commands, HTTP endpoints, MCP tools, LLM prompts, or
agents that run automatically at points in Claude Code's lifecycle. They receive
JSON on stdin (command hooks) or as a POST body (http hooks) and can allow,
block, or modify what Claude does.

Official source: https://code.claude.com/docs/en/hooks

## How to use this skill

1. Clarify **which lifecycle moment** the behavior belongs to (see event table
   below). Most requests map to `PreToolUse` (gate/modify before), `PostToolUse`
   (react after, e.g. lint/format), `UserPromptSubmit` (inject context / block),
   `SessionStart` (load context), or `Stop` (keep working / validate).
2. Choose the **config location** by scope (see table). Default to
   `.claude/settings.json` for project-shared, `.claude/settings.local.json` for
   personal/untracked, `~/.claude/settings.json` for all projects.
3. Write the `hooks` block. Use `${CLAUDE_PROJECT_DIR}` for script paths so they
   resolve regardless of cwd.
4. For anything non-trivial (parsing tool input, emitting a decision), write a
   **script** and point the hook at it rather than inlining shell. Read
   `references/hooks-reference.md` for the exact input/output JSON per event.
5. Tell the user to run `/hooks` to verify the hook is registered, and to
   restart / re-trust the workspace if it was just added to `.claude/settings.json`.

For the full event catalog, every input/output field, exit-code semantics, matcher
rules, handler types, and worked examples, read **`references/hooks-reference.md`**.

## Config skeleton

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard.sh",
            "args": [],
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

- `matcher` filters *within* an event (usually tool name). `"*"`, `""`, or
  omitted = match all. Plain names/lists (`Bash`, `Edit|Write`) are exact;
  anything with other characters is treated as an unanchored JS regex
  (`mcp__memory__.*`).
- The inner `hooks` array can hold several handlers; all matching hooks run.
- `if` (tool events only) narrows by permission rule: `"if": "Bash(rm *)"`.

## Config locations and scope

| Location | Scope | Committed to git |
|---|---|---|
| `~/.claude/settings.json` | All projects for this user | No |
| `.claude/settings.json` | This project, all users | Yes |
| `.claude/settings.local.json` | This project, this user | No |
| Managed policy settings | Organization-wide | Yes |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes |
| Skill / subagent frontmatter (`hooks:` key) | That session / subagent | Yes |

## Most-used events (quick pick)

| Want to… | Event | Key mechanism |
|---|---|---|
| Block / rewrite a tool call before it runs | `PreToolUse` | JSON `hookSpecificOutput.permissionDecision` = `allow`/`deny`/`ask`; `updatedInput` to modify; or exit 2 |
| Lint/format/test after an edit or command | `PostToolUse` | run tooling; exit 2 or `additionalContext` feeds results back to Claude |
| Add context to (or block) every user prompt | `UserPromptSubmit` | stdout becomes context; exit 2 blocks and erases the prompt |
| Load project state when a session starts | `SessionStart` | matchers `startup`/`resume`/`clear`/`compact`/`fork`; stdout adds context |
| Force Claude to keep going / validate before it stops | `Stop` | exit 2 or `{"continue": true, "stopReason": "..."}` |
| Run something before compaction | `PreCompact` | matchers `manual`/`auto`; exit 2 blocks |
| Desktop notification on permission/idle | `Notification` | matchers like `permission_prompt`, `idle_prompt` |
| Cleanup when session ends | `SessionEnd` | output shown to user only |

Many more events exist (`PostToolUseFailure`, `PostToolBatch`, `PermissionRequest`,
`PermissionDenied`, `SubagentStart`, `SubagentStop`, `PostCompact`,
`UserPromptExpansion`, `InstructionsLoaded`, `ConfigChange`, `FileChanged`,
`CwdChanged`, `WorktreeCreate`, etc.) — all documented in the reference file.

## Exit codes (command / http hooks)

| Code | Effect |
|---|---|
| `0` | Success. stdout used as context on some events (`UserPromptSubmit`, `SessionStart`). JSON on stdout is parsed. |
| `2` | **Blocking error** on events that support blocking; stderr is shown to Claude. |
| other | Non-blocking error; stderr shown to user. |

JSON stdout (any exit) can carry structured decisions:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Blocked: writes to /etc are not allowed"
  },
  "systemMessage": "optional note surfaced to Claude"
}
```

## Common recipes

**Format TS/JS after every write:**
```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [ { "type": "command",
          "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/format.sh" } ] }
    ]
  }
}
```
`format.sh` reads `jq -r '.tool_input.file_path'` from stdin and runs the formatter.

**Deny `rm -rf` / risky bash:** `PreToolUse` matcher `Bash`, script inspects
`.tool_input.command`, emits `permissionDecision: "deny"`.

**Inject current git branch + ticket into each prompt:** `UserPromptSubmit`
command hook that prints the info to stdout.

See `references/hooks-reference.md` for full example scripts and the config that
wires each one up.

## Debugging checklist

- `/hooks` — lists registered hooks (event, matcher, type, source). If it's not
  there, the config file isn't being loaded or JSON is invalid.
- `.claude/settings.json` hooks need **workspace trust**; a freshly added hook
  may require re-trusting or restarting the session.
- Run Claude with `--debug` to see hook invocation and output.
- Test the script directly: `echo '{"tool_input":{"command":"rm -rf /"}}' | .claude/hooks/guard.sh`
- Command hooks run from the project dir; use `${CLAUDE_PROJECT_DIR}` for paths.
- `"disableAllHooks": true` in settings (or `--settings`) turns everything off.
- On Windows, set `"shell": "powershell"` on the handler or point `command` at a
  `.ps1` / `.cmd`; the default shell is bash.
