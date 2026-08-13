# Hooks

## Native format (Claude Code)

`.claude/settings.json` → `"hooks"` key. Events in PascalCase (`PreToolUse`, `PostToolUse`,
`UserPromptSubmit`, `Stop`, `SessionStart`, `SessionEnd`, `PostCompaction`, `PermissionRequest`), each
event mapped to a list of `{matcher, hooks}` groups, `matcher` a regex against the tool name (`Bash`,
`Read`, `Edit`, ...), `hooks` entries with a shell `command`.

## From Copilot CLI

Source: one JSON file per hook under `.github/hooks/<name>.json` (or `~/.copilot/hooks/` personal),
camelCase event keys (`sessionStart`, `sessionEnd`, `userPromptSubmitted`, `preToolUse`, `postToolUse`,
`agentStop`), `bash`/`powershell` command keys, no per-tool scoping — every hook in the file runs on every
occurrence of the event.

### Already compatible?

**No — needs a real rewrite, not a field rename.** The schemas diverge structurally: Copilot has no
`matcher` concept at all (one file = unconditional on that event), uses separate `bash`/`powershell`
command fields instead of a single `command` string, and is one-file-per-hook instead of one config with
arrays.

### Translation steps

1. Reverse the event-name mapping: `sessionStart`→`SessionStart`, `sessionEnd`→`SessionEnd`,
   `userPromptSubmitted`→`UserPromptSubmit`, `preToolUse`→`PreToolUse`, `postToolUse`→`PostToolUse`,
   `agentStop`→`Stop` (closest match, but verify semantics — "agent finished responding" in Copilot vs.
   "agent wants to end its turn" in Claude Code may not be identical; flag this to the user rather than
   assuming equivalence for anything timing-sensitive).
2. Since Copilot has no `matcher`, the translated Claude Code hook applies to *every* tool call for that
   event by default — set `matcher` to empty/omitted (matches everything) unless the user wants to narrow
   scope, which is new behavior on the Claude Code side, not a translation.
3. Pick `bash` (or `powershell` if that's the only one populated and cross-platform support matters) as
   the `command` string. If both are present and differ meaningfully, ask the user which behavior should
   be authoritative rather than merging them silently — Claude Code's `command` is a single shell string
   with no separate Windows branch.
4. Add the translated group to the appropriate event array under `"hooks"` in `.claude/settings.json`
   (merge with any existing entries for that event rather than overwriting the key).
5. Don't assume Copilot's stdout contract (block/inject decisions, if any) matches Claude Code's
   (`{"decision": "block"}`, `hookSpecificOutput`, `updatedInput`) — verify against current docs or test
   live before relying on a translated hook to gate or inject anything.
6. Keep the original `.github/hooks/*.json` files — Copilot keeps reading them independently.

## From DevIn CLI

Source: `.devin/hooks.v1.json` (or `.devin/config.json`/`.devin/config.local.json` under `"hooks"`),
**same PascalCase event names and `{matcher, hooks}` array shape as Claude Code** — this is a genuinely
shared schema, not a lookalike.

### Already compatible?

**Yes, structurally — with one important caveat.** The JSON shape is identical, so this is close to a
direct copy. **What is NOT inherited automatically:** `matcher` regexes written against DevIn's tool names
(`exec`, `read`, `edit`, `grep`, `glob`) will silently fail to match anything in Claude Code, because
Claude Code's tool names are capitalized (`Bash`, `Read`, `Edit`, `Grep`, `Glob`). A hook with `"^exec$"`
needs to become `"^Bash$"` to fire under Claude Code.

### Translation steps

1. Copy the contents of `.devin/hooks.v1.json` into `.claude/settings.json` under the `"hooks"` key
   (merge with any existing entries per event rather than overwriting).
2. Rewrite every `matcher` to Claude Code's tool names: `exec`→`Bash`, `read`→`Read`, `edit`→`Edit`,
   `grep`→`Grep`, `glob`→`Glob`. This is the single most common mistake when porting in this direction —
   review every `matcher` even if the JSON "looks fine" (it parses without error either way).
3. `command` hooks: scripts generally don't need to change, since the stdin JSON shape (`tool_name`,
   `tool_input`, ...) is shared — but if a script branches on the string value of `tool_name` (comparing
   against `"exec"`, `"read"`, ...), those string comparisons need the same remapping.
4. `prompt` hooks: no changes needed beyond confirming the event name, since these are evaluated against
   the same stdin fields regardless of harness.
5. Keep `.devin/hooks.v1.json` in place — DevIn keeps reading it (and, per DevIn's own passthrough, may
   *also* be reading `.claude/settings.json` directly already; check whether the user actually needs a
   separate translated copy or whether DevIn's existing passthrough already covers it before duplicating).
