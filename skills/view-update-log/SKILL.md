---
name: view-update-log
description: Use when the user wants to see git commit history, version changelog, update log, or project history. Handles viewing latest commit, browsing history, or looking up specific versions.
---

# View Update Log

Display formatted git commit history. Supports latest, history, specific version lookup, and date-range queries.

## Commands

### Default — Latest commit

`/view-update-log` (no arguments)

```bash
git log -1 --format="%H%n%ad%n%s%n%n%b" --date=format:"%Y-%m-%d %H:%M:%S"
```

Format output as:

```
Version:  <subject>
Date:     <date>
Details:  <body>
```

Append hint: `Use /view-update-log history to see more.`

### History — Multiple commits

`/view-update-log history` or `/view-update-log history N`

Default count: **10** if no number given.

```bash
git log -<count> --format="%ad | %s" --date=format:"%Y-%m-%d %H:%M:%S"
```

Format as numbered list. If commit body has meaningful text, show it indented below the entry.

End with: `Showing <count> of <total> total commits.`

### Specific version

`/view-update-log <version>` (e.g., `/view-update-log v260502`)

```bash
git log --all --grep="<version>" --format="%H%n%ad%n%s%n%n%b" --date=format:"%Y-%m-%d %H:%M:%S"
```

Show full commit details. If not found: `No commit found matching "<version>".`

### Date range

`/view-update-log today` — commits from today
`/view-update-log week` — commits from the past 7 days

```bash
git log --since="midnight" --format="%ad | %s" --date=format:"%Y-%m-%d %H:%M:%S"
git log --since="7 days ago" --format="%ad | %s" --date=format:"%Y-%m-%d %H:%M:%S"
```

## Error handling

| Situation | Action |
|---|---|
| Not a git repo | Report and stop |
| No commits | Report and stop |
| Version not found | Suggest browsing history |
