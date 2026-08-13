# Hooks

## Native format

One JSON file per hook definition (not one big config with matcher-scoped arrays like Claude Code).

### File locations

| Scope | Location | Filename |
|---|---|---|
| Project | `.github/hooks/` | `<name>.json` — name describes the hook's purpose |
| Personal (macOS/Linux) | `~/.copilot/hooks/` (or `$COPILOT_HOME/hooks/`) | `<name>.json` |
| Personal (Windows) | `%USERPROFILE%\.copilot\hooks\` (or `%COPILOT_HOME%\hooks\`) | `<name>.json` |

Hook configuration loads only when the CLI starts — changes require a restart, same as instructions.

### Config schema

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "bash": "echo \"Session started: $(date)\" >> logs/session.log",
        "powershell": "Add-Content -Path logs/session.log -Value \"Session started: $(Get-Date)\"",
        "cwd": ".",
        "timeoutSec": 30,
        "env": { "KEY": "VALUE" }
      }
    ]
  }
}
```

- Top level is `{"version": 1, "hooks": {...}}`, where `hooks` keys are event names and values are arrays
  of command objects.
- Each command object: `type` (`"command"`), `bash` (Linux/macOS script), `powershell` (Windows script —
  requires PowerShell 7+ in `PATH`), `cwd` (optional working dir), `timeoutSec` (optional, default 30),
  `env` (optional).
- **No `matcher` concept.** Every hook in an event's array runs on every occurrence of that event; there's
  no per-tool regex scoping the way Claude Code's `PreToolUse`/`PostToolUse` matchers work.
- Provide both `bash` and `powershell` for cross-platform hooks; a hook missing the key for the current OS
  simply doesn't run there.

### Events

| Event | Fires |
|---|---|
| `sessionStart` | Session begins |
| `sessionEnd` | Session ends |
| `userPromptSubmitted` | User submits a prompt |
| `preToolUse` | Before tool execution |
| `postToolUse` | After tool execution |
| `errorOccurred` | An error occurs |
| `agentStop` | Agent finishes responding |

Event names are **camelCase**, unlike Claude Code's PascalCase (`SessionStart`, `PreToolUse`, ...).

### Exit codes & output

Not explicitly documented for stdout-based decisions (block/allow, context injection) the way Claude
Code's hooks are. Don't assume Claude's `{"decision": "block"}` / `hookSpecificOutput` JSON contract
carries over — if the user needs a hook to block an action or inject context, verify the exact contract
against current Copilot CLI docs or test it live rather than porting Claude's output schema unchanged.

## Already compatible?

**No.** Unlike rules and skills, Copilot CLI does not read `.claude/settings.json`'s `hooks` key directly
— the schemas are structurally different enough (single global config with `matcher`-scoped arrays vs. one
file per hook keyed by camelCase event, `bash`/`powershell` command strings instead of a generic `command`
+ regex) that there's no meaningful passthrough. Every hook needs an explicit rewrite, not a file move.

## Translation steps (Claude Code hook → Copilot CLI hook)

1. For each `{matcher, hooks}` entry under a Claude Code event, decide whether the matcher actually
   narrows behavior. If it matches every tool (`matcher` empty/omitted), the Copilot equivalent is a
   direct per-event command. If it targets specific tools (e.g., `"^Bash$"`), Copilot has no matcher
   field — the hook script itself must inspect the tool/event context (if available) to replicate the
   narrowing, or the translation genuinely can't be scoped the same way and the user should be told so.
2. Map the event name: `SessionStart`→`sessionStart`, `SessionEnd`→`sessionEnd`,
   `UserPromptSubmit`→`userPromptSubmitted`, `PreToolUse`→`preToolUse`, `PostToolUse`→`postToolUse`,
   `Stop`→`agentStop` (closest match, verify semantics — "agent finishes responding" vs Claude's "agent
   wants to end its turn" may not be identical). Claude Code's `PermissionRequest` and `PostCompaction`
   have no documented Copilot equivalent — flag rather than drop silently if the hook depends on them.
3. Convert the `command` (shell string or script path) into `bash` (and a `powershell` translation if
   Windows support matters — don't leave Windows users with a silently no-op hook).
4. Create one `.github/hooks/<descriptive-name>.json` file per hook (or per related group), each with the
   `{"version": 1, "hooks": {...}}` wrapper.
5. Do not port Claude's stdout-based block/context-injection contract assuming it's identical — verify
   current Copilot behavior before relying on a hook to alter agent behavior rather than just log/notify.
