---
name: deploy-claude-env
description: >
  Deploy Claude Code plugins and skills from personal repositories to a new environment.
  Installs plugins from zhiding512/claude-plugins (configs, marketplace plugins) and
  skills from zhiding512/claude-skills. Plugins are deployed to personal (~/.claude/).
  Skills default to personal unless a project path is specified.
  Use when setting up a new machine, "deploy my claude env", "setup claude plugins",
  "install my skills", or "/deploy-claude-env".
  Usage: /deploy-claude-env [--skills-scope personal|project] [--skip-plugins] [--skip-skills]
disable-model-invocation: true
user-invocable: true
argument-hint: "[--skills-scope personal|project] [--skip-plugins] [--skip-skills]"
allowed-tools: Bash Read Write Edit Glob Grep Agent
---

# Deploy Claude Code Environment

Deploy plugins and skills from personal GitHub repositories to the current environment.

**Repositories:**
- **Plugins + Configs**: `https://github.com/zhiding512/claude-plugins`
- **Skills**: `https://github.com/zhiding512/claude-skills`

## Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--skills-scope` | `personal` | Where to install skills: `personal` (~/.claude/skills/) or `project` (.claude/skills/) |
| `--skip-plugins` | — | Skip plugin/config deployment |
| `--skip-skills` | — | Skip skills deployment |
| `--project-skills` | — | Comma-separated list of skill names to deploy to current project instead of personal |

## Steps

### Step 1: Parse Arguments

Parse `$ARGUMENTS` for flags:
- `--skills-scope personal` (default) or `--skills-scope project`
- `--skip-plugins` to skip plugin deployment
- `--skip-skills` to skip skills deployment
- `--project-skills skill1,skill2` to deploy specific skills to the project scope

If `--project-skills` is specified, those skills go to `.claude/skills/<name>/` in the current working directory. All other skills go to `~/.claude/skills/<name>/`.

### Step 2: Deploy Plugins and Configs

Skip this step if `--skip-plugins` was specified.

#### 2a: Clone or update the plugins repo

```bash
PLUGINS_REPO="https://github.com/zhiding512/claude-plugins"
PLUGINS_TMP="/tmp/claude-plugins-deploy"
rm -rf "$PLUGINS_TMP"
git clone --depth 1 "$PLUGINS_REPO" "$PLUGINS_TMP"
```

#### 2b: Deploy settings.json

Read the current `~/.claude/settings.json` and the repo's `configs/settings.json`.
**Merge** them intelligently:

1. For `permissions.allow`: union both arrays (keep all unique entries)
2. For `hooks`: merge hook entries (keep existing hooks, add new ones from repo)
3. For `model`: use the repo version (it represents the user's preferred model)
4. For `statusLine`: use the repo version
5. For `extraKnownMarketplaces`: merge (add new marketplaces, keep existing)
6. For `enabledPlugins`: merge (add new enabled plugins, keep existing)

Write the merged result back to `~/.claude/settings.json`.

**Important**: Do NOT blindly overwrite. Always read existing config first and merge.

#### 2c: Deploy statusline-command.sh

```bash
cp "$PLUGINS_TMP/configs/statusline-command.sh" ~/.claude/statusline-command.sh
chmod +x ~/.claude/statusline-command.sh
```

#### 2d: Deploy installed_plugins.json

Read existing `~/.claude/plugins/installed_plugins.json` (if it exists) and the repo's version.
Merge the `plugins` object: add new plugin entries, keep existing ones. Update version/sha if the repo has a newer version.

Write to `~/.claude/plugins/installed_plugins.json`.

#### 2e: Install marketplace plugins

For each marketplace defined in the settings.json `extraKnownMarketplaces`, check if the plugin is already installed in `~/.claude/plugins/marketplaces/`. If not, the user needs to run the plugin's install process.

For `claude-notifications-go`:
1. Check if the binary exists and works
2. If not installed, inform the user they need to run `/notifications-init` after this deployment to complete the notification plugin setup

#### 2f: Cleanup

```bash
rm -rf "$PLUGINS_TMP"
```

### Step 3: Deploy Skills

Skip this step if `--skip-skills` was specified.

#### 3a: Clone or update the skills repo

```bash
SKILLS_REPO="https://github.com/zhiding512/claude-skills"
SKILLS_TMP="/tmp/claude-skills-deploy"
rm -rf "$SKILLS_TMP"
git clone --depth 1 "$SKILLS_REPO" "$SKILLS_TMP"
```

#### 3b: Enumerate skills

List all directories in the repo root that contain a `SKILL.md` file:

```bash
find "$SKILLS_TMP" -maxdepth 2 -name "SKILL.md" -printf '%h\n' | sort
```

#### 3c: Deploy each skill

For each skill directory found:

1. Extract the skill name from the directory name
2. Determine target path:
   - If skill name is in `--project-skills` list: `.claude/skills/<name>/`
   - Otherwise use the default scope:
     - `personal` (default): `~/.claude/skills/<name>/`
     - `project`: `.claude/skills/<name>/`
3. Check if skill already exists at target:
   - If yes, compare content. If different, overwrite and note as "updated"
   - If same, note as "unchanged"
   - If not exists, note as "new"
4. Copy the entire skill directory (SKILL.md + any supporting files like scripts/, references/):

```bash
mkdir -p "$TARGET_DIR"
cp -r "$SKILLS_TMP/$SKILL_NAME/"* "$TARGET_DIR/"
```

#### 3d: Cleanup

```bash
rm -rf "$SKILLS_TMP"
```

### Step 4: Post-deployment Verification

1. Verify `~/.claude/settings.json` is valid JSON:
   ```bash
   python3 -c "import json; json.load(open('$HOME/.claude/settings.json'))"
   ```

2. Verify each deployed skill's SKILL.md exists and has valid frontmatter

3. Check if `ccusage` is installed (needed by statusline):
   ```bash
   which ccusage 2>/dev/null
   ```
   If not found, note it in the report.

### Step 5: Report Summary

Print a summary:

```
Claude Environment Deployment Complete!

Plugins & Configs:
  settings.json          - merged (N new permissions, M new hooks)
  statusline-command.sh  - deployed
  installed_plugins.json - merged
  claude-notifications   - configured (run /notifications-init to activate)

Skills deployed (personal):
  create-skill             - updated
  commit-and-push          - new
  sync-context             - new
  kernel-trace-analysis    - new (with scripts/)
  lds-optimization         - unchanged
  vgpr-pressure-analysis   - new

Skills deployed (project):
  (none)

Notes:
  - Run /notifications-init to complete notification plugin setup
  - ccusage not found: statusline cost display will show "—"
```

## Error Handling

- **git clone fails**: Check network connectivity and repo accessibility. Repos must be public or gh auth must be configured.
- **JSON parse error**: If settings.json merge produces invalid JSON, restore from backup. Always create a backup before modifying:
  ```bash
  cp ~/.claude/settings.json ~/.claude/settings.json.bak
  ```
- **Permission denied**: Ensure ~/.claude/ directory is writable.
- **Skill conflict**: If a skill exists in both personal and project scope, warn but don't overwrite the other scope.

## Notes

- This skill always creates backups of existing configs before modifying them
- Plugin marketplace syncing (downloading plugin code) requires Claude Code's built-in plugin system. This skill only configures the metadata.
- Skills with supporting files (scripts/, references/) are copied as complete directories
- The skill is idempotent: running it multiple times is safe
