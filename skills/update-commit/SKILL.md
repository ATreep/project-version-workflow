---
description: "Generate a dated version tag, create a professional conventional-commit, tag it, and optionally push."
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

- If ALL three outputs are empty → the tree is clean. **Report "Working tree is clean — nothing to commit."** and **STOP**.
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

Collect version tags matching today's date from recent commit messages:

```bash
git log --format=%s -200
```

Extract every string matching the pattern `vYYMMDD` or `vYYMMDD-N` where `YYMMDD` is today's date.

### 1.3 Parse suffixes and find maximum

| Match found | Suffix value |
|---|---|
| `vYYMMDD` (no `-N`) | 0 |
| `vYYMMDD-N` | N |

Find the maximum suffix value across all matches.

### 1.4 Compute next version

| Condition | Next version |
|---|---|
| No match found | `vYYMMDD` |
| Maximum suffix is N | `vYYMMDD-(N+1)` |

**VIOLATION — never do this:** If `vYYMMDD` already appears in ANY commit message, the next version MUST be `vYYMMDD-1` or higher. Reusing `vYYMMDD` when it already exists is forbidden. The base tag counts as suffix 0.

**Verification:** After computing the next version, grep commit messages again to confirm it does not already exist. If it does exist, increment until you find an unused version.

---

## Step 2 — Draft commit message

### 2.1 Classify the change

Analyze the diff from Step 0.2 and pick exactly one conventional commit type:

| Type | Use when |
|---|---|
| `feat.` | New feature or capability |
| `fix.` | Bug fix or error correction |
| `docs.` | Documentation only |
| `refactor.` | Code restructuring without behavior change |
| `perf.` | Performance improvement |
| `chore.` | Maintenance, tooling, dependencies |
| `style.` | Formatting, whitespace, naming conventions |

### 2.2 Draft the subject line (first line)

Format — the version comes first, then the type with a trailing period, then a concise imperative summary:

```
vYYMMDD(-N) <type>. <summary>
```

Rules:
- Max ~72 characters total.
- Summary is a short imperative phrase (e.g., "add macOS .DS_Store gitignore check").
- No trailing period on the summary.

Examples:

```
v260503 feat. add macOS .DS_Store gitignore check
v260502-1 fix. prevent reuse of base version tag
v260416 docs. rewrite SKILL.md for strict AI enforceability
v260427-2 refactor. simplify version sync logic in plugin metadata
```

### 2.3 Draft the detailed body (following lines)

Below the subject line, leave one blank line, then list each meaningful change as a numbered item:

```
1. <what was added or changed>
2. <what was added or changed>
3. ...
```

Rules:
- Each item is a single concise sentence describing one logical change.
- Include 1–5 items. Omit trivial or formatting-only changes.
- Do not repeat the subject line verbatim.

### 2.4 Present draft and request confirmation

Use `AskUserQuestion` with exactly these options:

- `Use as-is`
- `Modify commit message`
- `Cancel`

Show the full draft (subject + blank line + numbered body) in the question prompt. **Wait for the user's explicit selection before doing anything else.**

| User selection | Action |
|---|---|
| `Use as-is` | Use the draft unchanged. Continue to Step 3. |
| `Modify commit message` | Collect the user's edited text. Use that as the confirmed message. Continue to Step 3. |
| `Cancel` | **STOP**. Do not commit. |

**Do not proceed past this point without an explicit user selection.**

---

## Step 3 — Ensure .DS_Store is ignored on macOS

Detect the operating system:

```bash
uname -s
```

If the output is `Darwin` (macOS), check whether `.DS_Store` is already git-ignored:

```bash
git check-ignore .DS_Store
```

**If NOT ignored** (command exits 1) → check whether `.gitignore` exists:

```bash
test -f .gitignore && echo exists || echo missing
```

- If `.gitignore` exists → append `.DS_Store` on a new line at the end, then run `git add .gitignore`.
- If `.gitignore` does not exist → create `.gitignore` containing `.DS_Store`, then run `git add .gitignore`.

**If already ignored** (command exits 0) → do nothing. Continue to Step 4.

If the OS is not macOS → skip this step entirely. Continue to Step 4.

---

## Step 4 — Commit

### 4.1 Stage files

Stage each file explicitly by name. Use the file lists from Step 0.2. Do NOT use `git add .` or `git add -A`.

```bash
git add <file1> <file2> ...
```

### 4.2 Sensitive file check

Before committing, inspect all staged paths:

```bash
git diff --cached --name-only
```

If any staged file matches these patterns → **STOP** and ask the user to confirm:

- `.env` or `.env.*`
- `*.pem`, `*.key`, `*.p12`
- `credentials*`, `secret*`
- Any file that appears to contain secrets

Use `AskUserQuestion` with options: `Commit anyway` / `Cancel`.

### 4.3 Create commit

Commit with the confirmed message from Step 2. The exact format passed to `git commit -m` is:

```
vYYMMDD(-N) type. summary

1. first change description
2. second change description
3. third change description
```

Use a heredoc to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
vYYMMDD(-N) type. summary

1. first change description
2. second change description
3. third change description
EOF
)"
```

Replace the placeholder text with the actual confirmed values.

### 4.4 Handle commit failure

If the commit fails (hook error, merge conflict, etc.) → **STOP**. Report the error output to the user. Do not retry. Do not amend. Do not force.

---

## Step 5 — Create git tag

After a successful commit, create an annotated tag matching the version:

```bash
git tag -a VERSION -m "SUBJECT_LINE"
```

Where `VERSION` is the version string (e.g., `v260503` or `v260502-1`) and `SUBJECT_LINE` is the full first line of the commit message.

Verify the tag exists:

```bash
git tag -l VERSION
```

If the output is empty → **STOP**. Report that tag creation failed. Do not force.

---

## Step 6 — Post-commit push decision

After a successful commit and tag, do NOT auto-push.

### 6.1 Gather branch and remote info

Run:

```bash
git branch --show-current
git remote -v
```

### 6.2 Ask user whether to push

Use `AskUserQuestion` with exactly these options:

- `Push now`
- `Do not push`

| User selection | Action |
|---|---|
| `Push now` | Continue to 6.3 |
| `Do not push` | **STOP**. Workflow complete. |

### 6.3 Push

Push the current local branch and the new tag to the remote:

```bash
git push && git push --tags
```

If no upstream is configured, use:

```bash
git push -u origin <branch-name> && git push --tags
```

If push is rejected → **STOP**. Report the git output. Do NOT force push.

---

## Version iteration algorithm (reference)

This is the authoritative algorithm for Step 1. Follow it mechanically.

### Input

- Today's date as `YYMMDD` (e.g., `260502`)
- List of version strings from commit messages

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
11. Omitting the git tag after a successful commit.
12. Using a commit type other than the seven allowed (`feat.` / `fix.` / `docs.` / `refactor.` / `perf.` / `chore.` / `style.`).

---

## Error and edge-case handling

| Situation | Action |
|---|---|
| Not a git repo | Ask whether to init. Do nothing until user decides. |
| Clean working tree | Report and exit. |
| Cannot determine previous version | Start from `vYYMMDD`. |
| Commit fails | Report error and stop. |
| Tag already exists | Increment suffix and retry. |
| Tag creation fails | Report error and stop. |
| Push rejected | Report git output and stop. Never force push. |
| Remote or upstream missing | Report state and stop for user decision. |
