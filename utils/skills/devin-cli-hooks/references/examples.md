# Devin CLI Hook Examples

## 1. Block dangerous shell commands (PreToolUse, command)

`.devin/hooks.v1.json`:

```json
{
  "PreToolUse": [
    { "matcher": "^exec$", "hooks": [ { "type": "command", "command": "$DEVIN_PROJECT_DIR/.devin/scripts/block-rm.sh" } ] }
  ]
}
```

`.devin/scripts/block-rm.sh`:

```bash
#!/usr/bin/env bash
input=$(cat)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')
if printf '%s' "$cmd" | grep -Eq 'rm\s+-rf?\s+(/|~|\*)'; then
  printf '{"decision":"block","reason":"Refusing destructive rm. Narrow the path."}'
  exit 2
fi
exit 0
```

## 2. Rewrite tool input (PreToolUse, updatedInput)

Route every shell command through a logging wrapper:

```bash
#!/usr/bin/env bash
input=$(cat)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command')
jq -n --arg c "safe-wrapper $cmd" \
  '{hookSpecificOutput:{hookEventName:"PreToolUse",updatedInput:{command:$c}}}'
exit 0
```

## 3. Log all shell commands (PostToolUse)

```json
{
  "PostToolUse": [
    { "matcher": "exec", "hooks": [ { "type": "command", "command": ".devin/scripts/audit.sh" } ] }
  ]
}
```

```bash
#!/usr/bin/env bash
input=$(cat)
printf '%s\t%s\n' "$(date -Is)" \
  "$(printf '%s' "$input" | jq -c '{cmd:.tool_input.command, ok:.tool_response.success}')" \
  >> "$DEVIN_PROJECT_DIR/.devin/command-log.tsv"
exit 0
```

## 4. Auto-approve safe commands (PermissionRequest)

```bash
#!/usr/bin/env bash
input=$(cat)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // ""')
case "$cmd" in
  git\ status*|git\ diff*|git\ log*) printf '{"decision":"approve"}' ;;
  *) printf '{"decision":"deny","reason":"Manual review required."}' ;;
esac
exit 0
```

## 5. Inject policy on every prompt (UserPromptSubmit)

```json
{
  "UserPromptSubmit": [
    { "hooks": [ { "type": "command", "command": ".devin/scripts/policy.sh" } ] }
  ]
}
```

```bash
#!/usr/bin/env bash
cat > /dev/null
jq -n '{hookSpecificOutput:{hookEventName:"UserPromptSubmit",
  additionalContext:"Reminder: never deploy without a green CI run and an approved PR."}}'
exit 0
```

## 6. Prompt-type hook (LLM evaluation)

```json
{
  "PreToolUse": [
    {
      "matcher": "^edit$",
      "hooks": [
        { "type": "prompt",
          "prompt": "If this edit touches files outside src/ or tests/, respond with decision block and a short reason; otherwise approve." }
      ]
    }
  ]
}
```

## 7. Setup on session start (SessionStart, with timeout)

```json
{
  "SessionStart": [
    { "hooks": [ { "type": "command", "command": ".devin/scripts/setup.sh", "timeout": 10 } ] }
  ]
}
```

```bash
#!/usr/bin/env bash
cat > /dev/null
make deps >/dev/null 2>&1
jq -n '{hookSpecificOutput:{hookEventName:"SessionStart",additionalContext:"Env ready. Use `make test` to run the suite."}}'
exit 0
```

## 8. Keep working until tests pass (Stop — use with care)

```bash
#!/usr/bin/env bash
input=$(cat)
active=$(printf '%s' "$input" | jq -r '.stop_hook_active')
[ "$active" = "true" ] && exit 0          # already looping once, let it stop
if ! make test >/dev/null 2>&1; then
  printf '{"decision":"block","reason":"Tests are failing. Fix them before stopping."}'
  exit 0
fi
exit 0
```

Guard every blocking `Stop` hook with `stop_hook_active` (and/or an attempt counter) so it converges.

## Notes

- Make scripts executable (`chmod +x`) and keep them fast; use `timeout`.
- `jq` is the simplest way to parse the stdin JSON; ensure it is available in the environment.
- Exit `0` with stdout JSON is the normal control path. Exit `2` hard-blocks. Any other exit = error, logged, non-blocking.
- Run `/hooks` in Devin CLI to list loaded hooks and their source files.
