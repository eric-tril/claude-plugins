---
description: Generate commit message, PR description, and Linear ticket from branch changes
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git merge-base:*), Bash(git branch:*), Bash(git rev-parse:*), Agent, Read
---

# PR Writer — Generate PR Text from Branch Changes

Analyze staged changes and branch changes to produce ready-to-copy text for commits, PRs, and Linear tickets.

## Step 1 — Detect Current Branch and Base

Run this command to get the current branch name:

```bash
git rev-parse --abbrev-ref HEAD
```

Store as `CURRENT_BRANCH`.

If `CURRENT_BRANCH` is `main` or `master`, stop and tell the user:

> "You're on the main branch. Please switch to a feature branch first."

Find the merge-base with main:

```bash
git merge-base HEAD main
```

Store as `MERGE_BASE`.

## Step 2 — Gather Staged Changes

```bash
git diff --cached
```

Store as `STAGED_DIFF`.

Also get the file list:

```bash
git diff --cached --name-only
```

Store as `STAGED_FILES`.

## Step 3 — Gather Branch Changes

Get the full diff of everything on this branch:

```bash
git diff $MERGE_BASE..HEAD
```

Store as `BRANCH_DIFF`.

Get the list of commits on this branch:

```bash
git log --oneline $MERGE_BASE..HEAD
```

Store as `BRANCH_COMMITS`.

If both `STAGED_DIFF` and `BRANCH_DIFF` are empty, stop and tell the user:

> "No changes found. Stage some changes or make commits on your branch first."

## Step 4 — Launch the change-analyzer Agent

Use the Agent tool to launch the `change-analyzer` agent with all gathered context:

> Analyze the following changes and produce the PR text document.
>
> **Current branch**: `CURRENT_BRANCH`
>
> **Staged changes** (for commit message):
> ```
> STAGED_DIFF
> ```
>
> **Staged files**: STAGED_FILES
>
> **Branch changes** (for PR description and Linear ticket):
> ```
> BRANCH_DIFF
> ```
>
> **Branch commits**:
> ```
> BRANCH_COMMITS
> ```

## Step 5 — Present Output

Display the agent's output directly to the user. The output is already formatted as Markdown sections ready for copy/paste.

Add a brief note at the end:

> **Tip**: Copy any section above. The commit message is based on staged changes; the PR description and Linear ticket are based on all branch changes.
