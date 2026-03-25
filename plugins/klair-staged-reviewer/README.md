# Klair Staged Reviewer

A code review plugin for Claude Code customized for the **Klair project** (Python FastAPI backend + React TypeScript frontend). Reviews your **staged changes** (`git diff --cached`) using specialized agents, then walks you through each issue interactively — showing a suggested fix and applying it only if you approve.

## How It Works

1. You stage your changes with `git add`
2. Run the review command
3. Auto-detects backend (`.py`) vs frontend (`.ts/.tsx`) files
4. Specialized agents analyze your staged code against Klair conventions
5. Each issue is presented one at a time with a suggested fix
6. You approve or skip each fix
7. Approved fixes are applied but **not re-staged** — you control what gets committed
8. Automated checks run after fixes: `ruff` for Python, `pnpm lint:pr` + `tsc` for TypeScript

## Quick Start

```
/klair-staged-reviewer:review
```

That's it. The command will detect your staged changes, run the appropriate agents, and walk you through any issues found.

## Usage

**Full review (all agents):**
```
/klair-staged-reviewer:review
```

**Review specific aspects:**
```
/klair-staged-reviewer:review code
/klair-staged-reviewer:review tests errors
/klair-staged-reviewer:review simplify
/klair-staged-reviewer:review comments
```

**Available aspects:** `code`, `tests`, `errors`, `types`, `comments`, `simplify`, `all` (default)

## Agents

### code-reviewer
Core review agent. Backend: FastAPI patterns, error bubbling, Pydantic v2, Python 3.12, ruff. Frontend: TypeScript strict, Tailwind/design tokens, shadcn/ui, `@/` imports, component patterns. Confidence >= 80 to report.

### code-simplifier
Simplification pass. Backend: ruff formatting, Pydantic v2 patterns, async consistency. Frontend: Tailwind > MUI, design tokens > raw colors, `@/` paths. Preserves functionality.

### comment-analyzer
Backend: Pydantic field descriptions, FastAPI docstrings (OpenAPI), exception docs. Frontend: Component JSDoc, no restating types. Catches comment rot.

### staged-test-analyzer
Backend: pytest markers required, pytest-asyncio, mock external services. Frontend: Vitest + happy-dom, Testing Library, `*.{spec,vitest}.{ts,tsx}` naming.

### silent-failure-hunter
Backend: Exceptions must bubble from services to routers — NEVER catch-and-return-empty. Frontend: Proper HTTP error handling, no silent Axios swallowing.

### type-design-analyzer
Backend: Pydantic v2 models, Field constraints, validators, discriminated unions. Frontend: TypeScript strict, separate Props interfaces, no unused locals/params.

## Agent Selection

The command automatically selects which agents to run based on your staged changes:

| Condition | Agent |
|---|---|
| Always | code-reviewer |
| Test files staged | staged-test-analyzer |
| Comments/docs changed | comment-analyzer |
| Error handling changed | silent-failure-hunter |
| Types added/modified | type-design-analyzer |
| After other reviews | code-simplifier |

## Interactive Fix Workflow

After agents complete their review, issues are sorted by severity and presented one at a time:

1. **Issue description** — which agent found it, severity, file:line, and why it matters
2. **Suggested fix** — the current code and proposed replacement
3. **Your choice** — approve to apply the fix, or skip to move on

After all issues are processed, you get a summary of what was applied vs skipped.

**Important:** Fixed files are modified but NOT re-staged. Use `git diff` to review changes, then `git add -p` to selectively stage what you want.

## Recommended Workflow

```
1. Write code
2. git add -p              # Stage what you want reviewed
3. /klair-staged-reviewer:review
4. Approve/skip fixes
5. Review automated check results (ruff / eslint / tsc)
6. git diff                # Review what was changed
7. git add -p              # Stage the fixes you want
8. git commit
```

## Pre-Push Automated Checks

At the end of the review, the command automatically runs checks that mirror the GitHub CI workflow:
- **Python files** (from `klair-api/`): `uv run ruff format` + `uv run ruff check` + `uv run pytest` on relevant test directories
- **TypeScript files** (from `klair-client/`): `pnpm lint` + `pnpm tsc --noEmit` + `pnpm build`

This catches formatting, lint, test, and build failures before you push — preventing CI failures.

## Tips

- **Stage first** — the review only looks at `git diff --cached`
- **Use `git add -p`** — stage specific hunks to review only part of your work
- **Re-run after fixes** — stage your fixes and re-run to verify they resolved the issues
- **Target specific concerns** — use aspect arguments for focused reviews
- **Backend-only review** — stage only `.py` files and only backend rules apply
- **Frontend-only review** — stage only `.ts/.tsx` files and only frontend rules apply

## License

Apache License 2.0
