---
name: create-skill
description: >
  Create a new Claude Code skill following the official specification (agentskills.io standard).
  Interactively collects skill name, description, invocation mode, tools, and content type,
  then generates a properly structured SKILL.md with correct frontmatter.
  Use when the user says "create a skill", "new skill", "generate skill", "make a slash command",
  or "/create-skill".
  Usage: /create-skill <skill-name> [project|personal]
disable-model-invocation: true
user-invocable: true
argument-hint: "<skill-name> [project|personal]"
allowed-tools: Read Write Bash(mkdir *) Bash(ls *)
---

# Create a New Claude Code Skill

Generate a new skill that follows the official Claude Code Skills specification.

## Step 1: Parse Arguments

Parse `$ARGUMENTS` to extract:
- **Skill name** (`$0`): lowercase letters, numbers, and hyphens only, max 64 chars
- **Scope** (`$1`, optional): `project` (default) or `personal`

If no skill name is provided, ask the user for one.

Determine the target directory:
- `project` → `.claude/skills/<name>/SKILL.md` (relative to current working directory)
- `personal` → `~/.claude/skills/<name>/SKILL.md`

Check if the skill already exists. If it does, ask the user whether to overwrite or pick a different name.

## Step 2: Collect Skill Configuration (Interactive)

Ask the user the following questions using AskUserQuestion. Ask them in batches of 2-3 to avoid overwhelming the user.

### Batch 1: Core Identity

1. **Description** (required): "What does this skill do? When should it be triggered?"
   - Remind the user: front-load the key use case; max ~250 chars for the listing display
   - This is the most important field — Claude uses it to decide when to auto-load the skill

2. **Content type**: "Is this skill reference knowledge or a task workflow?"
   - **Reference**: Conventions, patterns, domain knowledge Claude applies to current work
   - **Task**: Step-by-step instructions for a specific action (deploy, generate, migrate)
   - **Hybrid**: Both reference material and actionable steps

### Batch 2: Invocation Control

3. **Invocation mode**: "Who should be able to trigger this skill?"
   - **Both (default)**: User via `/name` and Claude automatically when relevant
   - **User only** (`disable-model-invocation: true`): For workflows with side effects (deploy, commit, send messages)
   - **Claude only** (`user-invocable: false`): Background knowledge, not a meaningful user action

4. **Argument hint** (optional): "Does this skill accept arguments? What format?"
   - Examples: `[filename]`, `<issue-number>`, `[source] [target]`, `<command> -- <args>`
   - If yes, note where `$ARGUMENTS` / `$0` / `$1` should appear in the skill body

### Batch 3: Advanced Options (ask only if relevant)

5. **Allowed tools** (optional): "Should this skill pre-approve any tools?"
   - Common patterns: `Read Grep Glob` (read-only), `Read Edit Bash Grep Glob Agent` (full edit), `Bash(git *)` (git only)
   - Remind: this grants permission, not restricts — all tools remain callable

6. **Execution context** (optional): "Should this skill run in isolation (subagent)?"
   - **Inline (default)**: Runs in the main conversation, can reference chat history
   - **Fork** (`context: fork`): Runs in a subagent, isolated from conversation
   - If fork: which agent type? (`Explore`, `Plan`, `general-purpose`, or custom)

7. **Path restriction** (optional): "Should this skill only activate for certain file types?"
   - Example: `"*.py"`, `"src/**/*.ts"`, `"kernels/*.py"`

Only ask Batch 3 if the user's answers in Batch 1-2 suggest these are relevant (e.g., task-type skills often need allowed-tools; reference skills rarely need fork context).

## Step 3: Generate SKILL.md

### Frontmatter

Build the YAML frontmatter using the **official field names** (not legacy names):

```yaml
---
name: <skill-name>
description: >
  <user-provided description>
# Only include fields that were explicitly configured:
# argument-hint: "<hint>"
# disable-model-invocation: true
# user-invocable: false
# allowed-tools: <space-separated list>
# context: fork
# agent: <agent-type>
# paths: <comma-separated globs>
# effort: <low|medium|high|max>
# model: <model-id>
---
```

**Rules for frontmatter generation:**
- Do NOT include fields with default values (omit rather than write `user-invocable: true`)
- Do NOT use legacy field names: use `allowed-tools` not `tools`, use `user-invocable` not `user_invocable`
- `description` should use `>` folded scalar style for multi-line
- `allowed-tools` uses space-separated format: `Read Edit Bash(git *)` not `Read,Edit,Bash`

### Body Content

Generate body content based on the content type:

#### For Reference Skills

```markdown
# <Skill Title>

<Brief overview of what knowledge this provides>

## Conventions

- Convention 1
- Convention 2

## Patterns

### Pattern Name

Description and code examples.

## Anti-patterns

- What to avoid and why
```

#### For Task Skills

```markdown
# <Skill Title>

<Brief overview of what this task accomplishes>

## Arguments

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `<arg>`  | Yes      | —       | What it does |

## Prerequisites

- Prerequisite 1
- Prerequisite 2

## Steps

### Step 1: <Action>

Detailed instructions with code blocks:

```bash
command-here
```

### Step 2: <Action>

...

## Error Handling

- **Problem**: Solution

## Validation

- [ ] Check 1
- [ ] Check 2
```

#### For Hybrid Skills

Combine both: reference section first, then task steps.

### Variable Substitutions

If the skill accepts arguments, use the correct substitution syntax in the body:
- `$ARGUMENTS` for all arguments as a string
- `$0`, `$1`, `$2` for positional arguments (0-based)
- `${CLAUDE_SKILL_DIR}` when referencing bundled scripts
- `${CLAUDE_SESSION_ID}` when logging per-session

### Dynamic Context Injection

If the skill needs runtime data, use `` !`command` `` syntax:
```markdown
Current branch: !`git branch --show-current`
```

For multi-line:
````markdown
```!
git status --short
npm test 2>&1 | tail -5
```
````

## Step 4: Write the Skill

1. Create the skill directory:
   ```bash
   mkdir -p <target-path>/<skill-name>
   ```

2. Write the `SKILL.md` file using the Write tool.

3. If the skill needs supporting files (scripts, templates, examples), ask the user if they want to create the directory structure now:
   ```
   <skill-name>/
   ├── SKILL.md
   ├── reference.md      (optional)
   ├── examples/          (optional)
   └── scripts/           (optional)
   ```

## Step 5: Verify and Report

After writing, verify:

1. Read back the generated file to confirm correctness
2. Report to the user:
   - Full path of the created skill
   - How to invoke: `/skill-name` or `/skill-name <args>`
   - Scope: project or personal
   - Whether Claude will auto-trigger or manual only

## Official Frontmatter Field Reference

This is the complete list of official Claude Code skill frontmatter fields. Use ONLY these field names:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | directory name | Skill name, becomes `/slash-command`. a-z, 0-9, hyphens only. Max 64 chars |
| `description` | string | first paragraph | What the skill does. Claude uses this for auto-triggering. Truncated at 250 chars in listings |
| `argument-hint` | string | — | Shown during autocomplete. E.g. `[issue-number]` |
| `disable-model-invocation` | bool | false | `true` = only user can invoke via `/name` |
| `user-invocable` | bool | true | `false` = hidden from `/` menu, only Claude can invoke |
| `allowed-tools` | string/list | — | Tools pre-approved when skill is active. Space-separated or YAML list |
| `model` | string | — | Override model for this skill |
| `effort` | string | inherit | `low`, `medium`, `high`, or `max` (Opus only) |
| `context` | string | — | `fork` = run in isolated subagent |
| `agent` | string | general-purpose | Subagent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom |
| `hooks` | object | — | Lifecycle hooks scoped to this skill |
| `paths` | string/list | — | Glob patterns limiting auto-activation. E.g. `"*.py, src/**/*.ts"` |
| `shell` | string | bash | `bash` or `powershell` for inline shell commands |

### Legacy Field Names (DO NOT USE in new skills)

| Legacy | Official |
|--------|----------|
| `tools:` | `allowed-tools:` |
| `user_invocable:` | `user-invocable:` |

## Notes

- Keep `SKILL.md` under 500 lines. Move detailed reference to separate files.
- Front-load the description with the key use case — it may be truncated at 250 chars.
- For task skills with side effects, always use `disable-model-invocation: true`.
- Skill content loads once when invoked and stays in context for the session.
- After generating, suggest the user test with `/skill-name` to verify it works.
