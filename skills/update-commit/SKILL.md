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

### 1.1 Get today's date and current time

Run:

```bash
date +%y%m%d
date +%H%M%S
```

Store the results as `YYMMDD` and `HHMMSS`. Example: `260601` and `145633`.

### 1.2 Form the version string

The version format is `vYYMMDD-HHMMSS`. Form the version by combining the date and time:

```
v{YYMMDD}-{HHMMSS}
```

Example: `v260601-145633`.

### 1.3 Collision check

**CRITICAL RULE — VERSION MUST ALWAYS CHANGE.** Every invocation of this skill MUST produce a version string that is different from ALL existing versions for today. Reusing an existing version is the single most serious violation of this skill. There are NO exceptions.

Although timestamps make collisions extremely unlikely, verify the computed version is truly unique:

```bash
git log --format=%s -200 | grep -F "v{YYMMDD}-{HHMMSS}"
```

- If **no match** → version is unique. Continue to Step 2.
- If **match found** (two invocations within the same second, or clock skew) → wait 2 seconds, re-run `date +%H%M%S`, and form a new version. Repeat until a unique version is found. This check is not optional.

**Verification (MANDATORY):** After forming the version, grep commit messages to confirm it does not already exist. If it does exist, wait and regenerate until unique. This check is not optional.

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
vYYMMDD-HHMMSS <type>. <summary>
```

Rules:
- Max ~72 characters total.
- Summary is a short imperative phrase (e.g., "add macOS .DS_Store gitignore check").
- No trailing period on the summary.

Examples:

```
v260601-145633 feat. add macOS .DS_Store gitignore check
v260601-150821 fix. prevent reuse of base version tag
v260531-093045 docs. rewrite SKILL.md for strict AI enforceability
v260601-112314 refactor. simplify version sync logic in plugin metadata
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
vYYMMDD-HHMMSS type. summary

1. first change description
2. second change description
3. third change description
```

Use a heredoc to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
vYYMMDD-HHMMSS type. summary

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

Where `VERSION` is the version string (e.g., `v260601-145633`) and `SUBJECT_LINE` is the full first line of the commit message.

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

## Version generation algorithm (reference)

This is the authoritative algorithm for Step 1. Follow it mechanically.

### Input

- Today's date as `YYMMDD` (e.g., `260601`)
- Current time as `HHMMSS` (e.g., `145633`)
- List of existing version strings from commit messages

### Process

```
1. Form: version = "v" + YYMMDD + "-" + HHMMSS
2. Verify: grep commit history for this exact version string
3. If found (collision): wait 2 seconds, get new HHMMSS, goto step 1
4. Result: return the unique version string
```

### Format specification

```
v{YYMMDD}-{HHMMSS}
 ├─ prefix (always "v")
 ├─ date part: 2-digit year, 2-digit month, 2-digit day
 ├─ hyphen separator
 └─ time part: 2-digit hour (24h), 2-digit minute, 2-digit second
```

### Examples

| Date | Time | Version |
|---|---|---|
| 2026-06-01 | 14:56:33 | `v260601-145633` |
| 2026-06-01 | 15:08:21 | `v260601-150821` |
| 2026-05-31 | 09:30:45 | `v260531-093045` |
| 2026-06-01 | 23:59:01 | `v260601-235901` |

### Daily reset rule

The date part changes at midnight (local time). Each calendar day produces versions with a different `YYMMDD` prefix:

```
2026-05-31 23:59:01 → v260531-235901
2026-06-01 00:00:05 → v260601-000005  ← new date prefix
2026-06-01 14:56:33 → v260601-145633
2026-06-01 15:08:21 → v260601-150821
```

---

## Violations

The following are violations of this skill. Do not do any of them:

1. **Reusing an existing version tag.** This is the #1 violation. The version MUST change on every single invocation. If a collision occurs (same-second invocation or clock skew), wait and regenerate until unique. There are zero exceptions to this rule.
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
| Version collision (same-second invocation) | Wait 2 seconds, regenerate timestamp, re-check. |
| Commit fails | Report error and stop. |
| Tag already exists | Regenerate timestamp and retry. |
| Tag creation fails | Report error and stop. |
| Push rejected | Report git output and stop. Never force push. |
| Remote or upstream missing | Report state and stop for user decision. |
