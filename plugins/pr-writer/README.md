# pr-writer

Generate commit messages, GitHub PR descriptions, and Linear ticket text from your branch changes.

## Usage

```
/pr-writer:generate
```

## What It Does

1. Detects your current branch and finds the merge-base with main
2. Gathers staged changes (`git diff --cached`) and branch changes
3. Analyzes all changes with an AI agent
4. Outputs four copy-ready Markdown sections:
   - **Commit Message** — from staged changes
   - **GitHub PR Title** — from branch changes
   - **GitHub PR Description** — with Business Value section
   - **Linear Ticket** — title and description

## Requirements

- Must be on a feature branch (not main/master)
- Either staged changes or branch commits must exist

## No Configuration Needed

This plugin requires no config files or API tokens. It reads git state only.
