# PR Review Responder

A Claude Code plugin that helps you assess and respond to PR review comments. Uses a specialized agent to deeply analyze each review comment against the actual codebase, determines if the concern is valid, applies fixes when appropriate, and posts friendly replies back to the reviewer on GitHub.

## How It Works

1. Detects the PR for your current branch
2. Fetches all unresolved review comments via GitHub's GraphQL API
3. For each comment, an agent reads the code in context (deeper than the diff hunk) and assesses whether the concern is valid
4. You choose to fix or skip each comment (or use auto mode)
5. Fixes are applied and **staged** — you review before committing and pushing
6. Friendly replies are drafted and posted to GitHub after your approval

## Quick Start

```
/pr-review-responder:respond
```

Make sure you're on the branch that has the PR with review comments.

## Usage

**Interactive mode (default):**
```
/pr-review-responder:respond
```

Walk through each review comment one at a time. For each comment you see:
- The reviewer's comment
- The agent's deep assessment (VALID / INTENTIONAL / PARTIAL)
- A suggested fix (if applicable)
- Drafted replies for both fix and skip scenarios

Then choose:
- **Fix and reply** — apply the fix, stage it, queue the reply
- **Skip and reply** — skip the fix, approve/edit a reply explaining why
- **Skip silently** — move on without posting a reply
- **Auto-fix remaining** — switch to auto mode for the rest

**Auto mode:**
```
/pr-review-responder:respond auto
```

The agent assesses all comments and automatically fixes issues it determines are valid (confidence >= 70). Intentional/low-confidence items are skipped. You review everything at the end before replies are posted.

## Agent: review-assessor

The `review-assessor` agent handles the deep analysis for each review comment. It:

- **Reads the actual file** — not just the diff hunk, but surrounding context (50+ lines)
- **Searches for callers/usage** — determines if a concern is valid by checking how code is actually used
- **Checks for existing tests** — verifies if the concern is already covered
- **Looks at codebase conventions** — determines if the code follows established patterns
- **Drafts replies** — writes friendly, professional responses for both fix and skip scenarios

## Interactive Fix Workflow

After the agent assesses each comment:

1. **Assessment** — verdict (VALID/INTENTIONAL/PARTIAL), confidence score, analysis
2. **Suggested fix** — current code, proposed replacement, and explanation
3. **Your decision** — fix, skip with reply, skip silently, or auto-fix remaining
4. **Reply approval** — all replies shown for your approval before posting to GitHub

## Reply Posting

Replies are batched and posted at the end of the workflow:
- **Post all** — send every queued reply
- **Review one by one** — approve/edit each reply individually
- **Don't post** — skip posting, handle replies manually

Replies are posted as threaded comments on the original review comment using the GitHub API.

## Recommended Workflow

```
1. Receive PR review feedback
2. /pr-review-responder:respond
3. Walk through each comment (or use auto mode)
4. Review staged changes: git diff --cached
5. Commit: git commit -m "address PR review feedback"
6. Push: git push
```

## Key Behaviors

- **Fixes are staged** (`git add`) but never committed or pushed — you control that
- **Replies require approval** — nothing is posted to GitHub without your explicit consent
- **Deep analysis** — the agent reads surrounding code, not just the diff context
- **Two replies drafted** — you always see what would be posted for both fix and skip scenarios
- **Auto-fix remaining** — start interactive, switch to auto once you trust the assessments

## Tips

- **Start interactive** — review the first few assessments to calibrate trust, then switch to auto
- **Use auto mode** for large reviews with many comments
- **Edit replies** — customize any reply before posting to match your voice
- **Check staged changes** — always review with `git diff --cached` before committing

## License

Apache License 2.0
