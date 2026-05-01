---
description: "Generate a dated version tag, prepend update_log.txt, commit with that version, and push the current branch after confirmation."
---

# update-commit

When invoked via `/update-commit`, execute the release-log-and-commit workflow in the current project directory.

## Version iteration mechanism

### Tag format

`vYYMMDD` or `vYYMMDD-N` where `YYMMDD` is the local date and `N` is the daily sequence number.

- Base tag (suffix 0): `v260501`
- Increments: `v260501-1`, `v260501-2`, `v260501-3`, ...

### How the next version is determined

Sources scanned (at least 200 recent commits + `update_log.txt`):

1. Collect all tags matching `vYYMMDD(-N)?` for today's date.
2. Parse suffix: `vYYMMDD` → 0, `vYYMMDD-N` → N.
3. Find the maximum suffix across all sources.
4. Next version = `max + 1` (or `vYYMMDD` if no match exists).

### Concrete example (date: 2026-05-01)

| Commit messages found | update_log.txt found | Max suffix | Next version |
|-----------------------|----------------------|------------|--------------|
| —                     | —                    | —          | `v260501`   |
| `v260501`             | —                    | 0          | `v260501-1` |
| `v260501`             | `v260501-1`          | 1          | `v260501-2` |
| `v260501-2`, `v260501`| `v260501-1`          | 2          | `v260501-3` |

### Daily sequence lifecycle

Each calendar day resets to `vYYMMDD`. Multiple commits within the same day produce an incrementing sequence. The next day starts fresh.

```
2026-04-30: v260430, v260430-1
2026-05-01: v260501                          ← new day, suffix resets
2026-05-01: v260501, v260501-1, v260501-2    ← same day, suffix increments
2026-05-02: v260502                          ← new day, suffix resets again
```

## Required behavior

### Step 0 - Preflight checks

1. Confirm this is a git repository using `git rev-parse --is-inside-work-tree`.
2. If not inside a git work tree, stop and ask the user via a Claude Code choice list whether to initialize git now.
   - Options: `Initialize git now` / `Do not initialize`.
   - Only run `git init` when the user explicitly selects `Initialize git now`.
3. Inspect changes across all states:
   - tracked unstaged: `git diff --name-only`
   - tracked staged: `git diff --cached --name-only`
   - untracked: `git ls-files --others --exclude-standard`
4. Treat the tree as clean only when all three lists are empty.
5. If clean, report and stop (do not update `update_log.txt`, commit, or push).

### Step 1 - Determine next version

Apply the version iteration mechanism defined above:

1. Scan at least 200 recent commit messages and `update_log.txt` for today's tags.
2. Parse suffixes, find the maximum, and compute the next version.
3. If no tag exists for today, start at `vYYMMDD`.

### Step 2 - Draft update log and request confirmation

1. Generate a brief change summary from diff output (1-3 bullets or one short sentence).
2. Show the draft to the user with this exact prompt:

   `Confirm or modify the pre-generated update log.`

3. Present the draft through a Claude Code choice list and wait for explicit selection before continuing.
   - Options: `Use as-is`, `Modify update log`, `Cancel`.
   - If `Use as-is`, continue with the draft unchanged.
   - If `Modify update log`, collect the user's edited text and use that as the confirmed log.
   - If `Cancel`, stop without writing `update_log.txt`, committing, or pushing.
4. Wait for the user-confirmed text before continuing.

### Step 3 - Prepend `update_log.txt`

After user confirmation, prepend this block to `update_log.txt` in project root:

```text
YYYY.M.D hh:mm:ss - VERSION
update logs...

```

Rules:
- Use current local timestamp (example: `2026.4.16 09:42:31 - v260416`).
- Append the git username (`git config user.name`) to the header line when it is set, so the line becomes `YYYY.M.D hh:mm:ss - VERSION - USERNAME`.
- Keep exactly one blank line after the entry.
- If `update_log.txt` does not exist, create it with this entry as the first block.
- Always prepend the new block at the top, preserving existing content below.

### Step 4 - Commit

1. Stage intended changes explicitly:
   - always stage files included in the reviewed diff for this run
2. Before staging `update_log.txt`, inspect `.gitignore` handling and ask the user through a Claude Code choice list.
   - If `update_log.txt` is currently ignored, options must be:
     - `Temporarily force-add and commit update_log.txt`
     - `Keep update_log.txt ignored and do not commit it`
     - `Cancel`
   - If `update_log.txt` is not currently ignored, options must be:
     - `Commit update_log.txt in this commit`
     - `Add update_log.txt to .gitignore, stage and commit .gitignore, and do not stage update_log.txt`
     - `Cancel`
   - Apply the selected option before final staging.
   - If `Cancel`, stop without committing.
3. Before commit, check staged paths for obvious sensitive files (for example: `.env`, `*.pem`, `*.key`, `credentials*`). If found, stop and ask user confirmation before including them.
4. Commit with message body equal to version + confirmed log:

```bash
git commit -m "$(cat <<'EOF'
VERSION

update logs...
EOF
)"
```

5. If commit fails (hooks, conflicts, etc.), report error and stop.

### Step 5 - Post-commit push decision

After a successful commit, do not auto-push.

1. Ask the user via a Claude Code choice list whether to push the current local branch to remote.
   - Options: `Push now`, `Do not push`.
2. Only push when the user explicitly selects `Push now` in the current run.
3. Before pushing, verify:
   - current branch name
   - matching remote exists (for example `origin`)
4. Use branch-aware push (push the current local branch, not forced `main`).
5. If remote or upstream is missing, report state and stop for user decision.

## Error and edge-case handling

- Not a git repo -> ask whether to run `git init`; do nothing else until user decides.
- Clean working tree -> inform user and exit.
- Cannot infer previous version -> start today's sequence from `vYYMMDD`.
- `update_log.txt` malformed -> still prepend the new block at top, preserving existing content below.
- Push rejected -> report git output and stop (do not force push).
