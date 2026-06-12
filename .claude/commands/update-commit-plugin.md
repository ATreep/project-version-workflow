Execute the skill `project-version-workflow:update-commit-bypass`.
While the `project-version-workflow:update-commit` skill determines the next version (Step 1), update the `version` field in every JSON file under `.claude-plugin/` to match before committing:

- `.claude-plugin/plugin.json` → `"version": "vYYMMDD-HHmmss"`
- `.claude-plugin/marketplace.json` → `"version": "vYYMMDD-HHmmss"` (in `plugins[0].version`)

This ensures the plugin manifest and marketplace listing stay in sync with the repo tag.

You should commit and push this repo with auth information in /Users/zhangsonghan/Documents/repositories/pat_and_username.txt
