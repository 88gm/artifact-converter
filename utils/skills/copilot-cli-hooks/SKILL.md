---
name: copilot-cli-hooks
description: >-
  Authoritative reference for creating, configuring, and debugging GitHub Copilot
  CLI hooks. Use whenever the user wants to add, edit, review, or troubleshoot a
  Copilot CLI hook, write a .github/hooks/*.json or ~/.copilot/hooks config, an
  inline hooks block in settings.json, block or rewrite tool calls, inject context
  on session/prompt start, gate permissions, force the agent to continue, or send
  lifecycle events to an HTTP endpoint. Triggers: "Copilot CLI hook", "copilot
  hooks", ".github/hooks", "~/.copilot/hooks", "preToolUse", "postToolUse",
  "permissionRequest", "userPromptSubmitted", "agentStop hook", "sessionStart hook",
  "preCompact", "copilot policy hook", "copilot cloud agent hook".
---

# GitHub Copilot CLI Hooks

Hooks are external commands that execute at specific lifecycle points during a
Copilot CLI session, for custom automation, security controls, and integrations.
They run on the developer's local machine in the same shell as the CLI, and also
run on the Copilot cloud agent surface (with restrictions — see "Cloud agent").

Source: https://docs.github.com/en/copilot/reference/hooks-reference

## Workflow for a hook request

1. Identify the **event** (see Events table) and whether a **matcher** regex is needed.
2. Choose hook **type**: `command` (shell script), `http` (POST payload to a URL), or `prompt` (auto-submit text — CLI interactive only).
3. Pick a **config location** (see Configuration). Prefer `.github/hooks/*.json` for repo-shared hooks (also the only location the cloud agent reads), or `~/.copilot/hooks/*.json` for personal hooks.
4. For `command` hooks: read the JSON payload from **stdin**, emit exactly **one** final decision JSON object to **stdout**, use **exit codes** to signal outcome.
5. Remember the two payload casings: `sessionStart` (camelCase fields) vs `SessionStart` (PascalCase event, snake_case fields, includes `hook_event_name`). Choose the event-name casing that matches the payload shape you want.

## Events

| Event | Fires when | Output processed | Can block / force continue | Cloud agent |
|---|---|---|---|---|
| `sessionStart` | new/resumed session begins | optional; can inject context (`additionalContext`) | no | once per job (new only, not resume) |
| `sessionEnd` | session terminates | no | no | yes; `reason` usually complete/error/timeout |
| `userPromptSubmitted` | user submits a prompt | optional; `modifiedPrompt` **SDK-only**, ignored by config-file hooks | no | once for the job prompt |
| `userPromptTransformed` | runtime transforms prompt before model sees it | yes; `modifiedTransformedPrompt` rewrites model-facing text | no (mutation only) | yes |
| `preToolUse` | before a tool executes | yes; allow / deny / ask / modify args | **deny** blocks the tool | yes; `ask` treated as `deny` |
| `postToolUse` | after a tool succeeds | yes; `modifiedResult`, `additionalContext` | no | yes |
| `postToolUseFailure` | after a tool fails | yes; recovery guidance via `additionalContext` | no | yes |
| `permissionRequest` | before the permission service runs | yes; `behavior` allow/deny, `interrupt` | deny blocks; `interrupt`+deny stops agent | **does not fire** (tools pre-approved) — use `preToolUse` |
| `preCompact` | context compaction begins | no; notification only | no | fires with `trigger: "auto"` only |
| `agentStop` | main agent finishes its turn | yes; `decision` block/allow | **block** forces another turn | yes |
| `subagentStart` | a subagent is spawned | optional; can inject context (cannot block) | no | yes; matcher on `agentName` |
| `subagentStop` | a subagent completes | yes; `decision` block, `modifiedResponse` | block forces another turn | yes |
| `errorOccurred` | an error occurs during execution | no | no | yes |
| `notification` | CLI emits a system notification | optional; `additionalContext` (fire-and-forget) | no (never blocks) | **does not fire** |

⚠️ After **8** consecutive `block` continuations on `agentStop`/`subagentStop`, the
CLI overrides the hook to prevent an infinite loop. Still, always make the block
condition converge.

## Configuration

### Locations (load/priority order: policy → user → project → plugins; all matching hooks from all sources run)

1. **Policy** (machine-wide, cannot be disabled):
   - Linux/macOS: `/etc/github-copilot/policy.d/*.json` (must be root-owned, not group/world-writable)
   - Windows: `C:\ProgramData\GitHub\Copilot\policy.d\*.json`, or registry `HKLM\Software\Policies\GitHub\Copilot`
2. **Repository**: `.github/hooks/*.json` (repo root) — *the only location the cloud agent reads*
3. **User**: `~/.copilot/hooks/*.json` (Windows: `%USERPROFILE%\.copilot\hooks\`)
4. **Inline repo settings**: `hooks` field in `.github/copilot/settings.json` or `settings.local.json`
5. **Inline user settings**: `hooks` field in `~/.copilot/settings.json`
6. **Plugin**: `hooks.json` declared by a plugin

### File format

Every hook file uses JSON with `version: 1`. In a dedicated hook file the object is the whole file:

```json
{
  "version": 1,
  "disableAllHooks": false,
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash|powershell",
        "type": "command",
        "bash": "./.github/hooks/check-command.sh",
        "timeoutSec": 10
      }
    ]
  }
}
```

Inline in `settings.json`, nest the same `hooks` object (and optionally `version`) under the settings root.

### `disableAllHooks`

- In a `.github/hooks/*.json` file: skips only that file's hooks (honored by CLI **and** cloud agent).
- At the top level of repo `settings.json`: **CLI only** — skips every hook from every source except policy hooks. (Cloud agent does not load `settings.json`.)

### Malformed config handling

- Directory-loaded files (`.github/hooks/`, `~/.copilot/hooks/`): a single malformed hook item is dropped and logged; siblings still load. Structural errors (bad JSON, bad `version`, non-array event list) reject the whole file.
- Inline `settings.json` hooks are strict: any item-level error rejects the entire `hooks` field.

## Hook types

### `command`

```json
{
  "type": "command",
  "bash": "BASH_COMMAND",
  "powershell": "POWERSHELL_COMMAND",
  "command": "CROSS_PLATFORM_FALLBACK",
  "cwd": "optional/dir",
  "env": { "VAR": "VALUE" },
  "timeoutSec": 30
}
```

- One of `bash` / `powershell` / `command` is required. `command` is copied to both `bash` and `powershell` when those are absent.
- `cwd`: relative to repo root or absolute. `env`: supports variable expansion. `timeoutSec` default 30 (`timeout` is an alias, ignored if `timeoutSec` present). `type` defaults to `"command"`.
- Cloud agent (Linux) honors only `bash` (and `command` as fallback); `powershell` is ignored.
- **Progress messages**: print JSON lines to stdout during execution; they are stripped from the output stream:
  ```bash
  echo '{"type": "progress", "message": "Checking policy..."}'
  echo '{"type": "progress", "message": "Routing...", "temporary": true}'
  ```
  Emit exactly one *final* decision object (it may span multiple lines). Two or more non-progress JSON objects concatenate into invalid JSON and are ignored.

### `http`

```json
{
  "type": "http",
  "url": "https://hooks.example.com/copilot",
  "headers": { "X-Source": "copilot-cli" },
  "allowedEnvVars": ["GITHUB_TOKEN"],
  "timeoutSec": 30
}
```

- POSTs the input payload as JSON. `url` and `type` required.
- `https://` only by default. `http://` rejected except localhost (`http://localhost`, `http://127.*`, `http://[::1]`) when env `COPILOT_HOOK_ALLOW_LOCALHOST=1`.
- `allowedEnvVars`: names expandable inside `headers` values; requires `https://`.
- The HTTP response body is parsed as the hook output JSON (same schema as command-hook stdout).

### `prompt`

```json
{ "type": "prompt", "prompt": "TEXT_OR_/slash-command" }
```

- Auto-submits text as if typed. **CLI only**, fires only on **new interactive sessions** — not on resume, not in non-interactive `-p` mode.

## Command hook I/O

**Stdin**: the event payload as JSON. camelCase events (`sessionStart`, `preToolUse`, …) get camelCase fields and numeric ms `timestamp`. PascalCase events (`SessionStart`, `PreToolUse`, …) get snake_case fields, ISO-8601 `timestamp`, and a `hook_event_name` field (matches the VS Code Copilot extension format). Common fields: `sessionId`/`session_id`, `timestamp`, `cwd`.

**Stdout**: exactly one final JSON object. Fields by event:

| Event | Output fields | Effect |
|---|---|---|
| `sessionStart`, `subagentStart` | `additionalContext` | inject context |
| `userPromptSubmitted` | `modifiedPrompt` | **SDK-only**; config hooks cannot rewrite the prompt |
| `userPromptTransformed` | `modifiedTransformedPrompt` | rewrite model-facing prompt (non-string ignored; empty string rejected) |
| `preToolUse` | `permissionDecision` (`allow`/`deny`/`ask`), `permissionDecisionReason` (required for deny), `modifiedArgs` | control + rewrite tool args |
| `postToolUse` | `modifiedResult: {resultType:"success", textResultForLlm:string}`, `additionalContext` | rewrite result / add context. Multiple hooks' `additionalContext` joined with `\n\n`, capped 10 KB |
| `postToolUseFailure` | `additionalContext` | recovery guidance |
| `permissionRequest` | `behavior` (`allow`/`deny`), `message` (reason on deny), `interrupt` (bool; deny+interrupt stops agent) | programmatic permission. Sandbox-bypass requests: `allow` still needs interactive user confirm; only `deny` propagates |
| `agentStop` | `decision` (`block`/`allow`), `reason` (prompt for next turn) | block forces another turn |
| `subagentStop` | `decision` (`block`), `reason`, `modifiedResponse` (replaces returned response) | block / rewrite |
| `notification` | `additionalContext` | prepended as a user message (async, never blocks) |

**Exit codes**:

| Code | Meaning |
|---|---|
| `0` | success; stdout parsed as hook output JSON |
| `2` | warning by default. For `preToolUse` / `permissionRequest`: treated as **deny** (stdout JSON merged with `{"behavior":"deny"}` for permissionRequest). For `postToolUseFailure`: treated as `additionalContext` appended to the failure |
| other non-zero | logged as failure, execution continues (**fail-open**) — *except* `preToolUse`, which is **fail-closed**: denies the tool ("Denied by preToolUse hook (hook errored)") |
| timeout | **always fail-open for every event, including `preToolUse`**: warning surfaced, call proceeds as if the hook had not run |

Output is bounded at 10 MiB per invocation (larger is truncated). Null `additionalContext` = absent.

## Matchers

Optional `matcher` regex, compiled anchored as `^(?:PATTERN)$`.

| Event | Matched against |
|---|---|
| `preToolUse` | `toolName` |
| `postToolUse` | `toolName` |
| `permissionRequest` | `toolName` |
| `subagentStart` | `agentName` |
| `preCompact` | `trigger` (`manual` / `auto`) |
| `notification` | `notification_type` (`shell_completed`, `shell_detached_completed`, `agent_completed`, `agent_idle`, `permission_prompt`, `elicitation_dialog`) |

Omit `matcher` to receive all. Non-tool events (`sessionStart`, `agentStop`, etc.) ignore it.

### Tool names (runtime)

`ask_user`, `bash`, `create`, `edit`, `glob`, `grep`, `powershell`, `task`, `view`, `web_fetch`, `web_search`.

### Claude-format matchers (PascalCase `PreToolUse` only)

Uses Claude matcher semantics: `*` / `**` / empty = all; literal or `|`-alternation (e.g. `Edit|Write`) matches a token; else case-sensitive anchored regex against the Claude tool name. Mappings: `bash`/`powershell`→`Bash`, `view`→`Read`, `create`→`Write`, `edit`/`str_replace_editor`/`apply_patch`→`Edit`, `grep`/`rg`→`Grep`, `glob`→`Glob`, `web_fetch`→`WebFetch`, `web_search`→`WebSearch`, `ask_user`→`AskUserQuestion`, `update_todo`→`TodoWrite`, `task`→`Agent` (`Task` also accepted). Tools without a Claude equivalent keep runtime names.

## Cloud agent (GitHub Copilot coding agent)

| Property | Value |
|---|---|
| OS | Linux; only `bash` (or `command`) honored, `powershell` ignored |
| Config discovery | **only** `.github/hooks/*.json` in the cloned repo — no user files, no `settings.json`, no plugins |
| Working dir | `/workspace` (repo cloned) or `/root` |
| Filesystem | ephemeral; exfiltrate data via an `http` hook to keep it |
| Network | firewalled; only GitHub/Copilot reachable unless an admin adds an allow rule |
| Env vars | `GITHUB_COPILOT_API_TOKEN`, `GITHUB_COPILOT_GIT_TOKEN`, `COPILOT_AGENT_PROMPT`, `HOME=/root` set; `GITHUB_TOKEN` **not** set |
| Interactivity | none; all permissions pre-granted; `permissionRequest` and `notification` never fire; `preToolUse` `ask` = `deny` |

## Examples

See `references/examples.md` for ready-to-use configs: block dangerous `bash`
commands, log every shell command to an HTTP endpoint, inject repo policy on
`sessionStart`, auto-deny edits to protected paths via `preToolUse`, force the
agent to keep working until tests pass via `agentStop`, and rewrite tool args.
