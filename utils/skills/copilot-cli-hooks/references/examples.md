# Copilot CLI Hooks — Example configs & scripts

All example configs go in `.github/hooks/*.json` (repo-shared, cloud-agent
compatible) or `~/.copilot/hooks/*.json` (personal). Scripts read JSON on stdin,
print one JSON object on stdout, and use exit codes.

---

## 1. Block dangerous bash commands (`preToolUse`, fail-closed)

`.github/hooks/guard-bash.json`:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash|powershell",
        "type": "command",
        "bash": "./.github/hooks/guard-bash.sh",
        "timeoutSec": 5
      }
    ]
  }
}
```

`.github/hooks/guard-bash.sh` (chmod +x):

```bash
#!/usr/bin/env bash
set -euo pipefail
payload=$(cat)
cmd=$(printf '%s' "$payload" | jq -r '.toolArgs.command // .toolArgs.script // ""')

if printf '%s' "$cmd" | grep -Eq 'rm +-rf +/|:\(\)\{|mkfs|dd +if=|curl.*\| *sh'; then
  echo '{"permissionDecision":"deny","permissionDecisionReason":"Blocked by guard-bash: destructive command pattern."}'
  exit 0
fi
echo '{"permissionDecision":"allow"}'
```

Note: `preToolUse` is fail-closed — if this script errors (non-zero, non-timeout)
the tool call is denied. A timeout, however, is fail-open.

---

## 2. Deny edits to protected paths (`preToolUse`)

`~/.copilot/hooks/protect-paths.json`:

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "matcher": "create|edit",
        "type": "command",
        "bash": "p=$(cat | jq -r '.toolArgs.path // .toolArgs.filePath // \"\"'); case \"$p\" in *.github/workflows/*|*/secrets/*|*.env) echo '{\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"Protected path.\"}';; *) echo '{\"permissionDecision\":\"allow\"}';; esac"
      }
    ]
  }
}
```

---

## 3. Inject repo policy on session start (`sessionStart`)

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "bash": "echo '{\"additionalContext\":\"House rules: run `npm test` before finishing. Never touch files under infra/. Conventional Commits required.\"}'"
      }
    ]
  }
}
```

---

## 4. Log every shell command to an HTTP endpoint (`preToolUse` http hook)

```json
{
  "version": 1,
  "hooks": {
    "preToolUse": [
      {
        "matcher": "bash|powershell",
        "type": "http",
        "url": "https://audit.example.com/copilot/exec",
        "headers": { "Authorization": "Bearer ${AUDIT_TOKEN}" },
        "allowedEnvVars": ["AUDIT_TOKEN"],
        "timeoutSec": 5
      }
    ]
  }
}
```

The endpoint receives the full `preToolUse` payload. To just observe, return
`{}` or `{"permissionDecision":"allow"}`; to block, return
`{"permissionDecision":"deny","permissionDecisionReason":"..."}`.

---

## 5. Force the agent to keep working until tests pass (`agentStop`)

```json
{
  "version": 1,
  "hooks": {
    "agentStop": [
      {
        "type": "command",
        "bash": "./.github/hooks/require-green-tests.sh",
        "timeoutSec": 600
      }
    ]
  }
}
```

`require-green-tests.sh`:

```bash
#!/usr/bin/env bash
set -uo pipefail
cat > /dev/null   # consume payload
if npm test --silent >/tmp/copilot-test.log 2>&1; then
  echo '{"decision":"allow"}'
else
  tail=$(tail -c 1500 /tmp/copilot-test.log | jq -Rs .)
  echo "{\"decision\":\"block\",\"reason\":\"Tests are failing. Fix them before stopping. Last output: ${tail}\"}"
fi
```

⚠️ Converges only if the agent can actually make tests pass. The CLI hard-stops
after 8 consecutive blocks.

---

## 6. Rewrite tool args (`preToolUse` `modifiedArgs`)

Force `npm ci` instead of `npm install`:

```bash
#!/usr/bin/env bash
payload=$(cat)
cmd=$(printf '%s' "$payload" | jq -r '.toolArgs.command // ""')
if [ "$cmd" = "npm install" ]; then
  echo '{"permissionDecision":"allow","modifiedArgs":{"command":"npm ci"}}'
else
  echo '{"permissionDecision":"allow"}'
fi
```

---

## 7. Recovery guidance on tool failure (`postToolUseFailure`)

```json
{
  "version": 1,
  "hooks": {
    "postToolUseFailure": [
      {
        "matcher": "bash",
        "type": "command",
        "bash": "err=$(cat | jq -r '.error'); if printf '%s' \"$err\" | grep -q 'command not found'; then echo '{\"additionalContext\":\"A binary is missing. Check package.json scripts or install it with the project package manager before retrying.\"}'; else echo '{}'; fi"
      }
    ]
  }
}
```

---

## 8. PascalCase / Claude-format variant

Use PascalCase event names to get the VS Code extension payload shape
(snake_case fields, `hook_event_name`, ISO timestamps) and Claude matcher
semantics:

```json
{
  "version": 1,
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "type": "command",
        "bash": "jq -r '.tool_input.path' | { read p; echo \"{\\\"permissionDecision\\\":\\\"allow\\\"}\"; }"
      }
    ]
  }
}
```

---

## 9. Auto-submit a slash command on new interactive sessions (`prompt` hook)

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      { "type": "prompt", "prompt": "/context load the architecture doc and summarize open TODOs" }
    ]
  }
}
```

CLI interactive only; does not run on resume or with `-p`.

---

## Debugging checklist

- JSON valid? A structural error rejects the whole file (siblings in the same
  directory still load; an inline `settings.json` error rejects the entire
  `hooks` field).
- `version: 1` present.
- Script executable (`chmod +x`) and path correct relative to repo root / `cwd`.
- Emitting exactly **one** non-progress JSON object on stdout.
- For deny on `preToolUse`, `permissionDecisionReason` is required.
- `modifiedPrompt` on `userPromptSubmitted` is ignored by config-file hooks (SDK only).
- Cloud agent: only `.github/hooks/*.json`, only `bash`, no `permissionRequest` / `notification`, `preToolUse` `ask` becomes `deny`.
- Set `disableAllHooks: true` to temporarily turn a file (or, in repo `settings.json`, everything) off.
