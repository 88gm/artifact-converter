---
name: optimize-skill-frontmatter
description: >-
  Optimizes the YAML frontmatter of Devin skills across a workspace without
  touching the skill body. Standardizes frontmatter to English, writes clear
  non-ambiguous descriptions, adds a "use when" guidance section, creates missing
  frontmatter, and renames user-only skill folders with a "cmd-" prefix.
  Use when the user asks to review, standardize, clean up, or optimize the
  frontmatter / metadata / descriptions of one or more skills, or to audit a
  folder of Devin skills for frontmatter quality. Also use when the user mentions
  a skills workspace and wants consistency, better triggering, or a progress
  checklist for that work.
---

# Optimize Skill Frontmatter

Improve the YAML frontmatter of every Devin skill in the workspace so each skill
triggers reliably and is described consistently. The body of each skill is off
limits — only the frontmatter block between the `---` fences may change.

## Scope and boundaries

- Edit **only** the YAML frontmatter. Never modify, reformat, or "improve" the
  Markdown body of a skill, even if it looks wrong. If the body has a real
  problem, note it in the checklist instead of fixing it.
- Frontmatter is always written in **English**, regardless of the language used
  in the body or by the user.
- Work skill by skill. Finish one completely (edit + checklist update) before
  starting the next, so progress is always recoverable.

## Devin frontmatter reference

Standard fields (Agent Skills spec):

| Field | Purpose |
|-------|---------|
| `name` | Skill identifier. Defaults to the folder name if omitted. |
| `description` | What the skill does and when to use it. Primary triggering signal. |
| `allowed-tools` | Restricts tools available while the skill is active. |

Devin-specific fields:

| Field | Purpose |
|-------|---------|
| `argument-hint` | Expected argument format shown next to the skill name. |
| `triggers` | Who may invoke the skill. `["user", "model"]` is the default (agent may auto-invoke). `["user"]` means **user-invocable only** — the agent must not auto-activate it. |

### Two kinds of skills

1. **Agent-invocable skills** — any skill that does not restrict `triggers` to
   user-only. If `triggers` is absent or is `["user", "model"]`, treat it as
   agent-invocable.
2. **User-invocable-only skills** — the frontmatter explicitly sets
   `triggers: ["user"]`. These are always marked explicitly; if that marker is
   absent, the skill is not user-only.

When a skill is user-invocable-only, **rename its folder** to add a `cmd-`
prefix (e.g. `deploy/` → `cmd-deploy/`). Skip renaming if the prefix is already
present. Keep the `name` field consistent with the new folder name unless the
user says otherwise.

## What good frontmatter looks like

Every skill's frontmatter must end up with:

- `name` — present, kebab-case, matching the folder.
- `description` — one cohesive statement covering **what the skill does** and
  **when to use it**, including an explicit "use when" clause. It must not be
  ambiguous, not be a single terse fragment, and not ramble. Aim for roughly
  2–5 sentences. Describe only what the skill actually does (read the body to
  confirm) — never invent capabilities.
- A **"use when" guidance** portion inside the description that names concrete
  situations that should trigger the skill, e.g. "Use when you need to create,
  edit, or manipulate SQL scripts of any kind." Prefer several phrasings and
  concrete triggers over one narrow example, since skills tend to under-trigger.
- `triggers` — keep as-is if already correct. Only set it when you have evidence
  of intent (e.g. the old description said "user runs this", or the user tells
  you). Do not silently convert an agent-invocable skill to user-only or back.
- Other fields (`allowed-tools`, `argument-hint`) — preserve unless clearly
  wrong; fix formatting only.

If a skill has **no frontmatter at all**, create it: read the body, infer name
from the folder, and write a `description` with the "use when" clause. Add
`triggers` only if the body makes the intent obvious.

### Description example

Weak:

```yaml
description: SQL helper.
```

Strong:

```yaml
description: >-
  Creates, edits, reviews, and refactors SQL scripts (DDL, DML, migrations,
  and analytical queries) following the team's style conventions. Use when the
  user asks to write a query, change a schema, add a migration, optimize slow
  SQL, or review an existing `.sql` file.
```

## Workflow

### 1. Locate the checklist

Look for `checklist.md` at the root of the workspace. If it does not exist,
create it.

### 2. Populate the checklist inventory

List **every** skill in the workspace. For each, record its folder name and a
short summary of what the skill does (from reading its body), plus an unchecked
status box. Use this structure:

```markdown
# Skill Frontmatter Optimization — Checklist

## Inventory

- [ ] `skill-folder-name` — <one-line summary of what the skill does>
- [ ] `another-skill` — <one-line summary>

## Change log
```

### 3. Optimize each skill

For one skill at a time:

1. Read the full skill file (body included, for understanding — not editing).
2. Determine whether it is agent-invocable or user-invocable-only.
3. Rewrite the frontmatter per "What good frontmatter looks like". Create it if
   missing.
4. If user-invocable-only, rename the folder to add the `cmd-` prefix.
5. Update the checklist (next step).

### 4. Update the checklist after each skill

The moment a skill is done, tick its box and append an entry to the **Change
log** summarizing exactly what changed in the frontmatter. Be specific:

```markdown
### `cmd-deploy` (was `deploy`)
- Renamed folder: `deploy/` → `cmd-deploy/` (user-invocable only: `triggers: ["user"]`).
- `description`: rewrote from "Deploy stuff" to a 3-sentence version stating what
  it deploys and adding a "use when" clause covering release, rollback, and
  smoke-test requests.
- Translated `description` from Portuguese to English.
- Added `argument-hint: <environment>`.
- Body unchanged.
```

Keep going until every box in the inventory is checked.

## Reporting

When all skills are done, give the user a brief summary: how many skills were
processed, how many were renamed with `cmd-`, how many had frontmatter created
from scratch, and anything that needs their decision (ambiguous `triggers`,
body problems you spotted but did not touch).
