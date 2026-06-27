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

Use `git add .` to stage all files in the working tree. Do NOT stage files explicitly by name — `git add .` ensures no files are missed.

```bash
git add .
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

Store the current branch name as `CURRENT_BRANCH`. If the user has explicitly specified a target push branch (e.g., "push to staging" or "push to dev"), store it as `TARGET_BRANCH` and skip to 6.3.

### 6.2 Decide push target

If `CURRENT_BRANCH` is `main` → skip to 6.3 (standard push confirmation).

If `CURRENT_BRANCH` is NOT `main` AND user has NOT specified a `TARGET_BRANCH` → ask the merge-to-main question using `AskUserQuestion`:

- `Merge to main and push`
- `Push current branch as-is`
- `Do not push`

| User selection | Action |
|---|---|
| `Merge to main and push` | Continue to 6.2a (merge flow). |
| `Push current branch as-is` | Continue to 6.3 (standard push confirmation). |
| `Do not push` | **STOP**. Workflow complete. |

#### 6.2a Merge current branch into main

Execute the merge flow:

```bash
git checkout main
git merge <CURRENT_BRANCH>
```

**If merge succeeds** (no conflicts) → continue to 6.3. The push target is `main`.

**If merge conflicts occur** (git exits non-zero) → follow the conflict resolution protocol below. Do NOT proceed until conflicts are resolved.

##### Conflict resolution protocol

When merge conflicts occur, the overriding principle is: **preserve as much code as possible, be cautious about removing source files or code.**

1. List all conflicted files:
   ```bash
   git diff --name-only --diff-filter=U
   ```

2. For each conflicted file, read the conflict markers to understand both sides:
   ```bash
   grep -n "^<<<<<<< \|^=======$\|^>>>>>>> " <file>
   ```

3. Resolve conflicts by combining both sides where possible:
   - Keep additions from both branches (do not delete code unless it's a clear, intentional removal).
   - When the same region was modified on both sides, prefer keeping both versions with a comment separator rather than picking one and discarding the other.
   - If a file was deleted on one side but modified on the other → **keep the file** (restore it from the branch that has it).
   - Only remove code if it is provably dead, duplicate, or the user explicitly requested its removal.

4. After resolving each file, stage it:
   ```bash
   git add <resolved-file>
   ```

5. Complete the merge:
   ```bash
   git commit -m "merge: merge <CURRENT_BRANCH> into main (conflict resolution)"
   ```

6. If you cannot resolve a conflict with confidence → **STOP** and ask the user for guidance. Show the conflicting sections.

7. After the merge commit succeeds → continue to 6.3. The push target is `main`.

### 6.3 Ask user whether to push

Use `AskUserQuestion` with exactly these options:

- `Push now`
- `Do not push`

| User selection | Action |
|---|---|
| `Push now` | Continue to 6.4 |
| `Do not push` | **STOP**. Workflow complete. |

### 6.4 Push

If the push target is `main` (either naturally or after merge):

```bash
git push origin main && git push --tags
```

Otherwise (pushing a non-main branch directly):

```bash
git push && git push --tags
```

If no upstream is configured for a non-main branch, use:

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
4. Staging files explicitly by name instead of using `git add .` to capture all changes.
5. Using `--force`, `--no-verify`, or `--no-gpg-sign` on any git command.
6. Committing sensitive files without user confirmation.
7. Continuing after the user selects `Cancel`.
8. Continuing when the working tree is clean.
9. Amending a commit instead of creating a new one.
10. Proceeding to the next step before the current step completes.
11. Omitting the git tag after a successful commit.
12. Using a commit type other than the seven allowed (`feat.` / `fix.` / `docs.` / `refactor.` / `perf.` / `chore.` / `style.`).
13. Discarding code from either branch during merge conflict resolution without user approval.
14. Deleting source files during merge conflict resolution unless the user explicitly requested it.
15. Using `git merge -X ours` or `git merge -X theirs` (these blindly discard one side's changes).

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
| Merge conflicts during merge-to-main | Follow conflict resolution protocol — preserve code, combine both sides, never blindly discard. |
| Cannot resolve merge conflict with confidence | Stop and ask user for guidance with conflicting sections shown. |
| `main` branch does not exist locally | Ask user whether to create it from remote (`git checkout -b main origin/main`) or push current branch as-is. |
