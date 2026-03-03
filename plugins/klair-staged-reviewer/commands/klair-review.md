---
description: "Review staged changes and interactively fix issues"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task", "Edit", "AskUserQuestion"]
---

# Staged Code Review

Review staged changes using specialized agents, then walk through each issue interactively — showing a suggested fix and applying it only if the user approves.

**Review Aspects (optional):** "$ARGUMENTS"

## Workflow:

### Step 1: Get Staged Changes

- Run `git diff --cached` to get all staged changes
- Run `git diff --cached --name-only` to get the list of staged files
- If no staged changes exist, inform the user: "No staged changes found. Stage files with `git add` first." Then stop.

### Step 2: Detect File Types

Categorize the staged files from the `--name-only` output:
- **Backend (Python)**: Files ending in `.py` — these belong to klair-api (FastAPI)
- **Frontend (TypeScript/React)**: Files ending in `.ts`, `.tsx`, `.js`, `.jsx`, `.css` — these belong to klair-client (React)
- **Other**: Config files, markdown, JSON, etc.

Set context flags: `has_backend_changes`, `has_frontend_changes`

This determines which rules agents apply AND which automated checks run after fixes.

### Step 3: Determine Review Scope

- Parse arguments to see if user requested specific review aspects
- Available aspects: **comments**, **tests**, **errors**, **types**, **code**, **simplify**, **all** (default)

### Step 4: Launch Review Agents

Based on the scope, launch the appropriate agents using the Task tool. Pass each agent the staged diff context by explicitly telling them to run `git diff --cached` to get the changes they should review.

**When invoking each agent, include the file-type context:**
- If only backend changes: "Review these staged Python/FastAPI changes. Apply backend (klair-api) rules only."
- If only frontend changes: "Review these staged TypeScript/React changes. Apply frontend (klair-client) rules only."
- If both: "Review these staged changes. Python files are backend (klair-api), TypeScript/React files are frontend (klair-client). Apply the appropriate rules for each."

**Agent selection logic:**
- **Always applicable**: code-reviewer (general quality)
- **If test files are staged**: staged-test-analyzer
- **If comments/docs changed**: comment-analyzer
- **If error handling changed** (try/except in .py, try/catch in .ts/.tsx): silent-failure-hunter
- **If types added/modified** (Pydantic models in .py, interfaces/types in .ts/.tsx): type-design-analyzer
- **After other reviews complete**: code-simplifier (polish and refine)

Launch agents **sequentially** — each report completes before the next starts. This makes results easier to process for the interactive fix loop.

### Step 5: Aggregate and Sort Findings

Collect all issues from all agents into a single list. Deduplicate any overlapping findings (multiple agents flagging the same line). Sort by severity:

1. **Critical** (must fix) — bugs, security issues, explicit guideline violations
2. **Important** (should fix) — quality issues, error handling gaps, test coverage gaps
3. **Suggestions** (nice to have) — simplifications, style improvements, comment improvements

### Step 6: Interactive Fix Loop

If no issues were found across all agents, congratulate the user:
"All agents reviewed your staged changes — no issues found. Your code looks good!"
Then skip to Step 7.

Otherwise, show a brief summary first:
"Found X issues across your staged changes (Y critical, Z important, W suggestions). Let's walk through them one at a time."

Then, for **each issue** starting with the most critical:

1. **Present the issue clearly:**
   - Which agent found it and its severity level
   - File path and line number
   - Description of the problem
   - Why it matters

2. **Show a suggested fix:**
   - Display the current code snippet (the problematic lines)
   - Display the proposed replacement code
   - Briefly explain what the fix does and why

3. **Ask the user** using AskUserQuestion with these options:
   - "Apply this fix" — Apply the suggested change
   - "Skip this issue" — Leave the code as-is and move on

4. **If the user approves:** Apply the fix using the Edit tool. Do **NOT** run `git add` — leave the file unstaged so the user controls staging.

5. **If the user skips:** Note it was skipped and move to the next issue.

6. **Move to the next issue** and repeat until all issues are processed.

### Step 7: Summary

After all issues are processed, show a final summary:

```
# Review Complete

## Applied Fixes (X)
- [severity] [agent]: description [file:line]

## Skipped Issues (X)
- [severity] [agent]: description [file:line]

## Clean Reviews
- [agents that found no issues]

Note: Fixed files have been modified but NOT re-staged.
Run `git diff` to review the applied changes.
Run `git add -p` to selectively stage fixes.
```

### Step 8: Run Automated Checks

If any fixes were applied in Step 6, run the relevant automated checks on the modified files:

**If backend (Python) files were modified by fixes:**
1. Run `uv run ruff format <modified-py-files>` to auto-format
2. Run `uv run ruff check <modified-py-files>` to lint
3. Report any remaining ruff issues with file:line references

**If frontend (TypeScript/React) files were modified by fixes:**
1. Run `pnpm lint:pr` to lint changed files against origin/main
2. Run `pnpm tsc --noEmit` to type-check the project
3. Report any remaining lint/type errors with file:line references

**If no fixes were applied**, skip this step entirely.

Present results clearly:
- "All automated checks passed — your fixes are clean." or
- List any lint/type errors found, noting which were likely introduced by fixes vs pre-existing

Note: These checks run on the working tree. They may surface pre-existing issues unrelated to the review fixes.

## Usage Examples:

**Full review (default):**
```
/klair-staged-reviewer:review
```

**Specific aspects:**
```
/klair-staged-reviewer:review tests errors
# Reviews only test coverage and error handling

/klair-staged-reviewer:review code
# General code review only

/klair-staged-reviewer:review simplify
# Simplification pass only
```

## Agent Descriptions:

**code-reviewer**:
- Backend: FastAPI patterns, error bubbling, Pydantic v2, Python 3.12 compliance, ruff style
- Frontend: TypeScript strict mode, Tailwind/design tokens, shadcn/ui, `@/` imports, component patterns
- Bug detection and confidence scoring ≥80

**code-simplifier**:
- Backend: ruff formatting, Pydantic v2 patterns, async consistency
- Frontend: Tailwind > MUI, design tokens > raw colors, `@/` path aliases
- Preserves functionality while improving clarity

**comment-analyzer**:
- Backend: Pydantic field descriptions, FastAPI docstrings (OpenAPI), exception documentation
- Frontend: Component JSDoc, avoids restating TypeScript types
- Verifies comment accuracy vs code

**staged-test-analyzer**:
- Backend: pytest markers required, pytest-asyncio, test structure
- Frontend: Vitest + happy-dom, Testing Library, `*.{spec,vitest}.{ts,tsx}` naming
- Behavioral coverage focus

**silent-failure-hunter**:
- Backend: Exceptions must bubble from services to routers — NEVER catch-and-return-empty
- Frontend: Proper HTTP error code handling, no silent Axios error swallowing
- Zero tolerance for silent failures

**type-design-analyzer**:
- Backend: Pydantic v2 models, Field constraints, validators, discriminated unions
- Frontend: TypeScript strict, separate Props interfaces, no unused locals/params
- Rates encapsulation, invariants, usefulness, enforcement

## Tips:

- **Stage first**: Run `git add` on files you want reviewed before invoking this command
- **Use `git add -p`**: Stage specific hunks to review only part of your work
- **Focus reviews**: Use aspect arguments to target specific concerns
- **Re-run after fixes**: Verify issues are resolved by staging fixes and re-running
- **Check diffs**: After the review, `git diff` shows what was fixed, `git diff --cached` shows what's still staged
