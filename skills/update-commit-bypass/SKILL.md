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

- Always push after a successful commit. Do not ask.
- Gather branch info: `git branch --show-current` (store as `CURRENT_BRANCH`).
- If the user has explicitly specified a target push branch → push to that branch. Do not merge.
- If `CURRENT_BRANCH` is `main` → run `git push origin main && git push --tags`.
- If `CURRENT_BRANCH` is NOT `main` AND no target branch was specified:
  1. Auto-merge current branch into main:
     ```bash
     git checkout main
     git merge <CURRENT_BRANCH>
     ```
  2. **If merge succeeds** (no conflicts) → run `git push origin main && git push --tags`.
  3. **If merge conflicts occur** → follow the conflict resolution protocol below, then push.
- If no upstream is configured, use `git push -u origin <branch-name> && git push --tags`.
- If the push is rejected, report the error and stop (same as base skill — no force push).

### Conflict resolution (when merging to main)

When merge conflicts occur, the overriding principle is: **preserve as much code as possible. Be cautious about removing source files or code.**

1. List all conflicted files: `git diff --name-only --diff-filter=U`
2. For each conflicted file, read both sides (use `grep` to find conflict markers).
3. Resolve by combining both sides:
   - Keep additions from both branches — do not delete code unless it's a clear, intentional removal.
   - When the same region was modified on both sides, keep both versions with a comment separator.
   - If a file was deleted on one side but modified on the other → **keep the file**.
   - Only remove code that is provably dead, duplicate, or the user explicitly requested removed.
4. Stage resolved files: `git add <resolved-file>`
5. Complete merge: `git commit -m "merge: merge <CURRENT_BRANCH> into main (conflict resolution)"`
6. If unable to resolve with confidence → **STOP** and report the conflicting sections to the user.
7. **Never** use `git merge -X ours` or `git merge -X theirs` — these blindly discard one side's changes.
8. After merge commit succeeds → run `git push origin main && git push --tags`.

## Summary of behavior

| Situation | Base skill | This skill |
|---|---|---|
| Not a git repo | Ask | Auto `git init` |
| Clean tree | Stop | Stop |
| Commit message draft | Ask (Use/Modify/Cancel) | Use as-is |
| Sensitive staged files | Ask | Auto-unstage + warn |
| After commit | Create tag | Create tag (same) |
| Non-main branch, no target specified | Ask (merge to main / push as-is / don't push) | Auto-merge to main, resolve conflicts, push |
| On `main` or target specified | Ask (Push/Don't) | Auto-push with tags |
| Merge conflicts | Ask for guidance | Auto-resolve preserving all code, stop only if unsure |
| Push rejected | Stop | Stop |
