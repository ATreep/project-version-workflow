---
description: "Non-interactive version of update-commit. Generates a dated version tag, prepends update_log.txt, commits, and auto-pushes — all without asking the user."
---

# update-commit-noask

When invoked via `/update-commit-noask`, execute the same release-log-and-commit workflow as `/update-commit` (defined in `project-version-workflow:update-commit`), but with all interactive prompts resolved to their default choices. **Never ask the user any questions.**

## Relationship to update-commit

This skill is a thin wrapper around `project-version-workflow:update-commit`. All version tag rules, diff collection, update log formatting, commit message format, and error handling are defined there. This skill only overrides the decision points — everything else is delegated to the base skill.

### Update log confirmation

- Always use the pre-generated draft as-is. Do not present the `Use as-is / Modify / Cancel` choice list.
- Skip directly to prepending `update_log.txt`.

### `.gitignore` handling

- **If `update_log.txt` is NOT currently ignored**: Commit it normally in this commit. Do not ask.

### Push

- Always push the current local branch to its remote after a successful commit. Do not ask.
- If the push is rejected, report the error and stop (same as base skill — no force push).

## Summary of behavior

| Situation | Base skill | This skill |
|---|---|---|
| Not a git repo | Ask | Auto `git init` |
| Clean tree | Stop | Stop |
| Update log draft | Ask (Use/Modify/Cancel) | Use as-is |
| `update_log.txt` tracked | Ask | Commit normally |
| Sensitive staged files | Ask | Auto-unstage + warn |
| After commit | Ask (Push/Don't) | Auto-push |
| Push rejected | Stop | Stop |
