---
description: "Generate a dated version tag, prepend update_log.txt, commit with that version, and push the current branch after confirmation."
---

# update-commit

When invoked via `/update-commit`, execute this workflow exactly as written. Every step is mandatory. Do not skip, reorder, combine, or improvise any step.

---

## Step 0 — Preflight

### 0.1 Verify git repository

Run:

```bash
git rev-parse --is-inside-work-tree
```

- If output is `true` → continue to 0.2.
- If this fails → **STOP**. Use `AskUserQuestion` with these options:
  - `Initialize git now`
  - `Do not initialize`
- Only run `git init` if the user selects `Initialize git now`. After init, continue to 0.2.
- If the user selects `Do not initialize` → **STOP**. Do nothing further.

### 0.2 Check working tree state

Run all three commands:

```bash
git diff --name-only
git diff --cached --name-only
git ls-files --others --exclude-standard
```

- If ALL three outputs are empty → the tree is clean. **Report "Working tree is clean — nothing to commit."** and **STOP**. Do not touch `update_log.txt`. Do not commit. Do not push.
- If any output is non-empty → the tree is dirty. Record the changed file lists for use in Step 4. Continue to Step 1.

---

## Step 1 — Determine next version

### 1.1 Get today's date string

Run:

```bash
date +%y%m%d
```

Store the result as `YYMMDD`. Example: `260502`.

### 1.2 Scan for existing today tags

Collect version tags matching today's date from BOTH sources:

**Source A — recent commit messages:**

```bash
git log --format=%s -200
```

**Source B — update_log.txt (if it exists):**

```bash
cat update_log.txt 2>/dev/null
```

From both sources combined, extract every string matching the pattern `vYYMMDD` or `vYYMMDD-N` where `YYMMDD` is today's date.

### 1.3 Parse suffixes and find maximum

Apply these rules mechanically:

| Match found | Suffix value |
|---|---|
| `vYYMMDD` (no `-N`) | 0 |
| `vYYMMDD-N` | N |

Find the maximum suffix value across all matches from both sources.

### 1.4 Compute next version

| Condition | Next version |
|---|---|
| No match found in either source | `vYYMMDD` |
| Maximum suffix is N | `vYYMMDD-(N+1)` |

**VIOLATION — never do this:** If `vYYMMDD` already appears in ANY commit message or in `update_log.txt`, the next version MUST be `vYYMMDD-1` or higher. Reusing `vYYMMDD` when it already exists is forbidden. The base tag counts as suffix 0.

**Verification:** After computing the next version, grep both sources again to confirm it does not already exist. If it does exist, increment until you find an unused version.

---

## Step 2 — Draft update log and request confirmation

### 2.1 Generate change summary

From the diff output collected in Step 0.2, write a brief summary: 1–3 bullet points or one short sentence describing what changed.

### 2.2 Present draft and wait for confirmation

Use `AskUserQuestion` with exactly these options:

- `Use as-is`
- `Modify update log`
- `Cancel`

Show the draft text in the question prompt. **Wait for the user's explicit selection before doing anything else.**

| User selection | Action |
|---|---|
| `Use as-is` | Use the draft unchanged. Continue to Step 3. |
| `Modify update log` | Collect the user's edited text. Use that as the confirmed log. Continue to Step 3. |
| `Cancel` | **STOP**. Do not write `update_log.txt`. Do not commit. Do not push. |

**Do not proceed past this point without an explicit user selection.**

---

## Step 3 — Prepend update_log.txt

After user confirmation, prepend this exact block to `update_log.txt`:

```text
YYYY.M.D hh:mm:ss - VERSION
update logs...

```

### Format rules (all mandatory)

1. `YYYY.M.D` — current local date. Use actual date values, not zero-padded. Example: `2026.5.2`, not `2026.05.02`.
2. `hh:mm:ss` — current local time, 24-hour format, zero-padded. Example: `00:28:10`.
3. `VERSION` — the version string from Step 1.
4. Username — append ` - USERNAME` to the header line if `git config user.name` returns a value. The header line becomes: `YYYY.M.D hh:mm:ss - VERSION - USERNAME`.
5. Update logs — the confirmed text from Step 2, immediately after the header line.
6. Exactly one blank line after the entry, before any existing content.
7. If `update_log.txt` does not exist, create it with this entry as the first block.
8. Always prepend — new block goes at the top. Existing content stays below, unchanged.

### Example entry

```text
2026.5.2 00:28:10 - v260502-1 - treep
Add critical rule to SKILL.md preventing reuse of base version tag

```

---

## Step 4 — Commit

### 4.1 Stage files

Stage each file explicitly by name. Use the file lists from Step 0.2. Do NOT use `git add .` or `git add -A`.

```bash
git add <file1> <file2> ...
```

### 4.2 Handle update_log.txt and .gitignore

First, check if `update_log.txt` is currently git-ignored:

```bash
git check-ignore update_log.txt
```

**If ignored** (command exits 0) → use `AskUserQuestion` with these options:

- `Temporarily force-add and commit update_log.txt`
- `Keep update_log.txt ignored and do not commit it`
- `Cancel`

**If NOT ignored** (command exits 1) → use `AskUserQuestion` with these options:

- `Commit update_log.txt in this commit`
- `Add update_log.txt to .gitignore, stage and commit .gitignore, and do not stage update_log.txt`
- `Cancel`

| User selection | Action |
|---|---|
| Force-add option | Run `git add -f update_log.txt` |
| Commit option | Run `git add update_log.txt` |
| Gitignore option | Append `update_log.txt` to `.gitignore`, then `git add .gitignore` |
| Cancel | **STOP**. Do not commit. Do not push. |

### 4.3 Sensitive file check

Before committing, inspect all staged paths:

```bash
git diff --cached --name-only
```

If any staged file matches these patterns → **STOP** and ask the user to confirm before proceeding:

- `.env` or `.env.*`
- `*.pem`, `*.key`, `*.p12`
- `credentials*`, `secret*`
- Any file that appears to contain secrets

Use `AskUserQuestion` with options: `Commit anyway` / `Cancel`.

### 4.4 Create commit

Commit with this exact message format. The first line is the VERSION. A blank line. Then the confirmed update logs:

```bash
git commit -m "$(cat <<'EOF'
VERSION

update logs...
EOF
)"
```

Replace `VERSION` and `update logs...` with the actual values.

### 4.5 Handle commit failure

If the commit fails (hook error, merge conflict, etc.) → **STOP**. Report the error output to the user. Do not retry. Do not amend. Do not force.

---

## Step 5 — Post-commit push decision

After a successful commit, do NOT auto-push.

### 5.1 Gather branch and remote info

Run:

```bash
git branch --show-current
git remote -v
```

### 5.2 Ask user whether to push

Use `AskUserQuestion` with exactly these options:

- `Push now`
- `Do not push`

| User selection | Action |
|---|---|
| `Push now` | Continue to 5.3 |
| `Do not push` | **STOP**. Workflow complete. |

### 5.3 Push

Push the current local branch to its remote tracking branch:

```bash
git push
```

If no upstream is configured, use:

```bash
git push -u origin <branch-name>
```

If push is rejected → **STOP**. Report the git output. Do NOT force push.

---

## Version iteration algorithm (reference)

This is the authoritative algorithm for Step 1. Follow it mechanically.

### Input

- Today's date as `YYMMDD` (e.g., `260502`)
- List of version strings from commit messages and `update_log.txt`

### Process

```
1. Filter: keep only strings matching `vYYMMDD` or `vYYMMDD-N`
2. Parse: `vYYMMDD` → suffix 0, `vYYMMDD-N` → suffix N
3. Find: max_suffix = maximum of all parsed suffixes (or -1 if no matches)
4. Compute: next_suffix = max_suffix + 1
5. Result: if next_suffix == 0 → "vYYMMDD", else → "vYYMMDD-{next_suffix}"
```

### Examples

| Existing versions found | Max suffix | Next version |
|---|---|---|
| (none) | -1 | `v260502` |
| `v260502` | 0 | `v260502-1` |
| `v260502`, `v260502-1` | 1 | `v260502-2` |
| `v260502-2`, `v260502` | 2 | `v260502-3` |
| `v260502-3`, `v260502-1`, `v260502` | 3 | `v260502-4` |

### Daily reset rule

Each calendar day starts fresh at `vYYMMDD`. The sequence from previous days does not carry over.

```
2026-05-01: v260501, v260501-1, v260501-2
2026-05-02: v260502                          ← resets
2026-05-02: v260502, v260502-1               ← increments within day
2026-05-03: v260503                          ← resets again
```

---

## Violations

The following are violations of this skill. Do not do any of them:

1. Reusing an existing version tag.
2. Skipping user confirmation at any `AskUserQuestion` gate.
3. Auto-pushing without explicit user selection.
4. Using `git add .` or `git add -A` instead of explicit file names.
5. Using `--force`, `--no-verify`, or `--no-gpg-sign` on any git command.
6. Committing sensitive files without user confirmation.
7. Continuing after the user selects `Cancel`.
8. Continuing when the working tree is clean.
9. Amending a commit instead of creating a new one.
10. Proceeding to the next step before the current step completes.

---

## Error and edge-case handling

| Situation | Action |
|---|---|
| Not a git repo | Ask whether to init. Do nothing until user decides. |
| Clean working tree | Report and exit. |
| Cannot determine previous version | Start from `vYYMMDD`. |
| `update_log.txt` malformed | Still prepend new block at top. |
| Commit fails | Report error and stop. |
| Push rejected | Report git output and stop. Never force push. |
| Remote or upstream missing | Report state and stop for user decision. |
