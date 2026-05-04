---
description: "Non-interactive version of update-commit. Generates a dated version tag, commits with a conventional message, creates a git tag, and auto-pushes — all without asking the user."
---

# update-commit-bypass

When invoked via `/update-commit-bypass`, execute the same workflow as `/update-commit` (defined in `project-version-workflow:update-commit`), but with all interactive prompts resolved to their default choices. **Never ask the user any questions.**

## Relationship to update-commit

This skill is a thin wrapper around `project-version-workflow:update-commit`. All version tag rules, diff collection, commit message format, and error handling are defined there. This skill only overrides the decision points — everything else is delegated to the base skill.

### Git init

- If not inside a git repo, run `git init` automatically. Do not ask.

### Commit message confirmation

- Always use the pre-generated draft as-is. Do not present the `Use as-is / Modify / Cancel` choice list.
- Skip directly to staging and committing.

### Sensitive file check

- If sensitive files are staged, auto-unstage them and warn the user. Do not ask.

### Git tag

- Always create the annotated git tag after a successful commit. Do not ask.

### Push

- Always push the current local branch and tags to its remote after a successful commit. Do not ask.
- Run `git push && git push --tags`.
- If no upstream is configured, use `git push -u origin <branch-name> && git push --tags`.
- If the push is rejected, report the error and stop (same as base skill — no force push).

## Summary of behavior

| Situation | Base skill | This skill |
|---|---|---|
| Not a git repo | Ask | Auto `git init` |
| Clean tree | Stop | Stop |
| Commit message draft | Ask (Use/Modify/Cancel) | Use as-is |
| Sensitive staged files | Ask | Auto-unstage + warn |
| After commit | Create tag | Create tag (same) |
| After tag | Ask (Push/Don't) | Auto-push with tags |
| Push rejected | Stop | Stop |
