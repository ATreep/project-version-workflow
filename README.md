# Project Version Workflow (PVW)

Manage project's versions and iterations by a standard workflow on Claude Code.



## Installation

Copy `skills` folder to your `.claude` directory.



## What PVW Does

The following skills/commands are provided:

1. `/update-commit`: generates a version name, commits a new version and manages update log automatically.

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
