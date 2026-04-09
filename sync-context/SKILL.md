---
name: sync-context
description: >
  Sync Claude Code context (CLAUDE.md and skills) to Cursor IDE rules format,
  so both tools share the same project knowledge and capabilities.
  Use when the user says "sync context", "sync to cursor", "sync rules",
  "update cursor rules", "/sync-context", or wants Claude Code and Cursor
  to share the same instructions and skills.
allowed-tools: Bash Read Write Glob Edit
---

# Sync Context: Claude Code ↔ Cursor

Synchronize project context and skills from Claude Code to Cursor so both
tools operate with the same project knowledge.

## What Gets Synced

| Source (Claude Code) | Target (Cursor) | Notes |
|----------------------|------------------|-------|
| `CLAUDE.md` | `.cursor/rules/claude-project-guide.mdc` | Project-level instructions, always applied |
| `.claude/skills/*/SKILL.md` | `.cursor/rules/skill-<name>.mdc` | Each skill becomes a Cursor rule |

## Steps

### 1. Ensure target directory exists

```bash
mkdir -p .cursor/rules
```

### 2. Sync CLAUDE.md → Cursor rule

Read `CLAUDE.md` from the project root. Convert it to a Cursor rule file
(`.mdc` format) with appropriate frontmatter:

**Output file**: `.cursor/rules/claude-project-guide.mdc`

```
---
description: "FlyDSL project guide — shared context from CLAUDE.md"
globs:
alwaysApply: true
---

<contents of CLAUDE.md>
```

- `alwaysApply: true` ensures Cursor always loads this context (same as
  CLAUDE.md being always loaded in Claude Code).
- If CLAUDE.md does not exist, skip this step and warn the user.

### 3. Sync skills → Cursor rules

For each skill directory in `.claude/skills/*/SKILL.md`:

1. Read the skill's `SKILL.md`
2. Extract frontmatter fields: `name`, `description`
3. Extract the body content (everything after the closing `---`)
4. Generate a Cursor rule file with converted frontmatter

**Frontmatter conversion**:

| Claude Code field | Cursor field | Mapping |
|-------------------|--------------|---------|
| `name` | (used in filename) | `skill-<name>.mdc` |
| `description` | `description` | Copy as-is (collapse to single line) |
| `user_invocable` | — | Not used in Cursor |
| `tools` | — | Not used in Cursor |

**Output file**: `.cursor/rules/skill-<name>.mdc`

```
---
description: "<description from skill, single line>"
globs:
alwaysApply: false
---

<body content from SKILL.md>
```

- Skills are set to `alwaysApply: false` so Cursor uses them on-demand
  based on the description match (similar to how Claude Code skills are
  invoked by description matching or `/name`).

### 4. Clean up stale rules

List all `skill-*.mdc` files in `.cursor/rules/`. If any correspond to
skills that no longer exist in `.claude/skills/`, delete them and report
which ones were removed.

Do NOT delete files in `.cursor/rules/` that don't start with `skill-` or
aren't `claude-project-guide.mdc` — those are user-created Cursor rules.

### 5. Report summary

Print a summary table:

```
Sync complete!

  CLAUDE.md → .cursor/rules/claude-project-guide.mdc  ✓
  Skills synced: N
    - skill-format-code.mdc                            ✓ (new)
    - skill-build-flydsl.mdc                           ✓ (updated)
    - skill-old-removed.mdc                            ✗ (deleted)

  To use in Cursor: open any file and the rules will auto-apply or
  match based on description. Reference skills with @rules in Cursor chat.
```

## Notes

- This is a **one-way sync** (Claude Code → Cursor). Edits to `.cursor/rules/skill-*.mdc`
  will be overwritten on next sync. Edit the source `.claude/skills/*/SKILL.md` instead.
- `claude-project-guide.mdc` is also overwritten from CLAUDE.md on each sync.
- User-created Cursor rules (files not matching `skill-*.mdc` or `claude-project-guide.mdc`)
  are never touched.
- Cursor `.mdc` files use the same markdown format as `.md` but with Cursor-specific
  frontmatter (`description`, `globs`, `alwaysApply`).
- After syncing, add `.cursor/rules/` to `.gitignore` if you don't want generated
  rules committed, or commit them if the team uses Cursor.

## .gitignore consideration

After syncing, check if `.cursor/rules/` is in `.gitignore`. If not, ask the user:

> `.cursor/rules/` is not in .gitignore. These are auto-generated from Claude Code
> context. Would you like to:
> 1. Add to .gitignore (keep generated, don't commit)
> 2. Commit them (team shares Cursor rules)
> 3. Do nothing
