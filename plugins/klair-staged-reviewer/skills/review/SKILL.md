---
description: "Review staged changes and interactively fix issues"
argument-hint: "[review-aspects]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Agent", "Edit", "AskUserQuestion"]
---

# Staged Code Review

Review staged changes using specialized agents, then walk through each issue interactively — showing a suggested fix and applying it only if the user approves.

**Review Aspects (optional):** "$ARGUMENTS"

## Critical Rules

**These rules are absolute and must NEVER be violated:**

1. **NEVER apply a fix without explicit user approval.** You MUST use `AskUserQuestion` for every single fix and receive a clear "Apply this fix" response before making any change. If the response is blank, empty, unclear, or anything other than explicit approval — treat it as "Skip this issue" and move on. Do NOT assume approval.

2. **NEVER run `git add` or stage any files.** Do not run `git add`, `git stage`, or any command that stages files at any point during the review. Modified files must remain unstaged so the user controls what gets committed.

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

Based on the scope, launch the appropriate agents using the Agent tool. Pass each agent the staged diff context by explicitly telling them to run `git diff --cached` to get the changes they should review.

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

   **IMPORTANT**: Only proceed with applying the fix if the user explicitly selects "Apply this fix". If the response is blank, empty, missing, or anything other than "Apply this fix", treat it as "Skip this issue".

4. **If the user explicitly selects "Apply this fix":** Apply the fix using the Edit tool. Do **NOT** run `git add` — the file must remain unstaged.

5. **If the user skips OR the response is blank/unclear:** Note it was skipped and move to the next issue. Do NOT apply the fix.

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

**REMINDER: Do NOT stage any files. Do NOT run `git add`.** Fixed files have been modified but NOT re-staged.
Run `git diff` to review the applied changes.
Run `git add -p` to selectively stage fixes.
```

### Step 8: Run Automated Checks

Run the relevant automated checks based on which file types were staged — regardless of whether fixes were applied. These checks catch issues that would fail the GitHub CI workflow (ruff, tests, build) so the user can fix them before pushing.

**If backend (Python) files are in the staged changes:**

All commands run from the `klair-api/` directory.

1. Run `uv run ruff format <modified-py-files>` to auto-format
2. Run `uv run ruff check <modified-py-files>` to lint
3. Report any remaining ruff issues with file:line references
4. Run the relevant tests based on which files changed:
   - Determine the feature area(s) from the changed file paths (e.g., changes to `routers/renewals.py` or `services/renewals.py` → run `pytest tests/renewals/`)
   - If test files themselves were changed, run those specific test files
   - If changes span multiple areas, run tests for each area
   - Run with `uv run pytest <test-path>` (excludes integration tests by default)
5. Report any test failures with file:line references and failure details

**If frontend (TypeScript/React) files are in the staged changes:**

All commands run from the `klair-client/` directory.

1. Run `pnpm lint` to lint the project
2. Run `pnpm tsc --noEmit` to type-check the project
3. Run `pnpm build` to verify the production build succeeds
4. Report any lint, type, or build errors with file:line references

Present results clearly:
- "All automated checks passed — clean to push." or
- List any failures found, grouped by check type (ruff / tests / lint / types / build)

Note: These checks run on the working tree. They may surface pre-existing issues unrelated to the review fixes.

**Final reminder: Do NOT run `git add` or stage any files. The user will handle staging themselves.**

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
- Bug detection and confidence scoring >=80

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
