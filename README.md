# Project Version Workflow (PVW)

Manage project's versions and iterations by a standard workflow on Claude Code.

## Installation

### As a Plugin (Recommended)

Install via Claude Code's plugin system:

```bash
/plugin marketplace add ATreep/project-version-workflow
/plugin install project-version-workflow
```

After installation, invoke the skills with the plugin namespace:

```
/project-version-workflow:update-commit
/project-version-workflow:update-commit-bypass
```



### As a Standalone Skill 

Copy the `skills` folder to your `.claude` directory:

```bash
cp -r skills/* ~/.claude/skills/
```

Then use the short-form commands:

```
/update-commit
/update-commit-bypass
```



## What PVW Does

Two skills are provided:

- `/update-commit` — Interactive versioned commit workflow
- `/update-commit-bypass` — Non-interactive variant; same workflow with all defaults auto-applied



### `/update-commit`

Generates a version name, commits a new version and manages update log automatically.

`/update-commit` generates a daily version tag (e.g. `v260416`), records the changes in `update_log.txt` along with the git username when available, commits them, and optionally pushes to remote. It checks for a clean repo, asks you to confirm the change summary, warns about sensitive files, and never force-pushes.

```mermaid
flowchart TD
    A[Check git repo] -->|No| B[Offer git init]
    A -->|Yes| C[Check working tree]
    C -->|Clean| D[Stop]
    C -->|Dirty| E[Pick next version]
    E --> F[Draft update log]
    F --> G{Confirm?}
    G -->|No| F
    G -->|Yes| H[Write update_log.txt]
    H --> I[Stage files]
    I --> J[Commit]
    J -->|Fail| K[Stop]
    J -->|OK| L[Ask to push]
    L -->|Yes| M[Push branch]
    M -->|Reject| K
    M -->|OK| N[Done]
    L -->|No| N
```

### `/update-commit-bypass`

Same workflow as `/update-commit`, but fully non-interactive — never asks the user. Auto-initializes git if needed, uses the pre-generated update log as-is, always commits `update_log.txt`, and auto-pushes the current branch after a successful commit. Suitable for CI, loops, or hands-off workflows.
