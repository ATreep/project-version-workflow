# project-version-workflow

## update-commit: Sync .claude-plugin versions

After the `update-commit` skill determines the next version (Step 1), update the `version` field in every JSON file under `.claude-plugin/` to match before committing:

- `.claude-plugin/plugin.json` → `"version": "vYYMMDD"`
- `.claude-plugin/marketplace.json` → `"version": "vYYMMDD"` (in `plugins[0].version`)

This ensures the plugin manifest and marketplace listing stay in sync with the repo tag.
