---
description: "Assess and respond to PR review comments"
argument-hint: "[auto]"
allowed-tools: ["Bash", "Glob", "Grep", "Read", "Agent", "Edit", "AskUserQuestion"]
---

# PR Review Responder

Fetch unresolved PR review comments from GitHub, deeply assess each one using a specialized agent, and interactively fix or respond to them. All fixes are staged (not committed) so you can review before pushing.

**Mode:** "$ARGUMENTS"
- No argument or "interactive" = interactive mode (default)
- "auto" = auto-fix mode (fix all valid issues, summarize at end)

## Critical Rules

**These rules are absolute and must NEVER be violated:**

1. **NEVER push commits or run `git push`.** Fixes are staged with `git add` but never committed or pushed. The user controls when to commit and push.

2. **NEVER post a GitHub reply without user approval.** All replies must be shown to the user and explicitly approved before posting via `gh api`.

3. **In interactive mode, NEVER apply a fix without explicit user approval.** You MUST use `AskUserQuestion` for every fix decision and receive a clear approval response before making any change. If the response is blank, empty, unclear, or anything other than explicit approval — treat it as "Skip silently" and move on.

## Workflow

### Step 1: Detect PR

Run the following command to get the current branch's PR:

```bash
gh pr view --json number,url,title,headRefName
```

If no PR exists for the current branch, inform the user:
"No pull request found for the current branch. Push your branch and create a PR first."
Then stop.

Extract and store:
- `PR_NUMBER`: the PR number
- `PR_URL`: the PR URL
- `PR_TITLE`: the PR title
- `BRANCH`: the head branch name
- `OWNER` and `REPO`: extract from the URL (e.g., `https://github.com/OWNER/REPO/pull/N`)

### Step 2: Fetch Unresolved Review Comments

Use the GitHub GraphQL API to fetch all review threads:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            isOutdated
            path
            line
            startLine
            comments(first: 10) {
              nodes {
                id
                databaseId
                body
                author { login }
                path
                line
                startLine
                diffHunk
                url
                createdAt
              }
            }
          }
        }
      }
    }
  }
' -f owner='OWNER' -f repo='REPO' -F pr=PR_NUMBER
```

Replace `OWNER`, `REPO`, and `PR_NUMBER` with the values from Step 1.

**Filter** the results:
- Only keep threads where `isResolved == false`
- Exclude threads where `isOutdated == true` (code has already changed under them)

If no unresolved comments remain:
"No unresolved review comments found on PR #X. Nothing to do!"
Then stop.

### Step 3: Parse and Present Summary

For each unresolved thread, extract from the **first comment** in the thread:
- **Reviewer**: `author.login`
- **File**: `path`
- **Line**: `line` (and `startLine` if different, indicating a range)
- **Comment**: `body` (the reviewer's actual feedback)
- **Diff context**: `diffHunk`
- **Comment Database ID**: `databaseId` (needed for REST reply API)
- **Thread ID**: thread `id` (node ID)
- **Comment URL**: `url`

Present a summary to the user:

```
Found X unresolved review comments on PR #N "[title]" from [reviewer(s)].

1. [file:line] — [first ~80 chars of comment]
2. [file:line] — [first ~80 chars of comment]
...

Let's assess each one.
```

### Step 4: Determine Mode

Check the `$ARGUMENTS`:
- If "auto" → go to **Step 5a: Auto Mode**
- Otherwise → go to **Step 5: Interactive Mode**

### Step 5: Interactive Mode

For each unresolved comment, in order:

**5.1 — Launch the review-assessor agent**

Use the Agent tool to launch the `review-assessor` agent. Pass it all the context:

```
Assess this PR review comment:

**Reviewer**: @[reviewer_login]
**File**: [path]
**Line**: [line]
**Comment**: "[body]"
**Diff context**:
```
[diffHunk]
```

Read the actual file at [path] and surrounding context. Determine if this concern is valid, suggest a fix if needed, and draft friendly replies for both fix and skip scenarios.
```

**5.2 — Present the assessment**

Display the agent's assessment to the user:

```
## Comment [N] of [total] — [file:line]

**Reviewer** (@username): "[comment text]"

**Agent Assessment**: [VALID/INTENTIONAL/PARTIAL] (confidence: X%)
**Summary**: [agent's one-line summary]

**Analysis**: [agent's reasoning]

[If VALID or PARTIAL:]
**Suggested Fix**:
  Current: [code snippet]
  Proposed: [fix snippet]
  Why: [explanation]

**Reply if fixed**: "[drafted reply]"
**Reply if skipped**: "[drafted reply]"
```

**5.3 — Ask the user for a decision**

Use `AskUserQuestion` with these options:
- "Fix and reply" — Apply the fix, stage it, and queue the "fixed" reply
- "Skip and reply" — Skip the fix, show the drafted skip reply for approval
- "Skip silently" — Skip without posting any reply
- "Auto-fix remaining" — Switch to auto mode for all remaining comments

**IMPORTANT**: If the response is blank, empty, or anything other than one of the explicit options above, treat it as "Skip silently" and move on.

**5.4 — Handle the decision**

**If "Fix and reply":**
1. Apply the fix using the Edit tool (use the agent's suggested fix)
2. Run `git add <file>` to stage the fixed file
3. Add the "fixed" reply to the reply queue for posting later

**If "Skip and reply":**
1. Show the drafted skip reply to the user
2. Use `AskUserQuestion` with options:
   - "Post this reply" — Queue the reply as-is
   - "Let me edit" — Wait for the user to provide custom reply text, then queue it
3. Add the approved reply to the reply queue

**If "Skip silently":**
1. Note it was skipped with no reply
2. Move to the next comment

**If "Auto-fix remaining":**
1. Switch to **Step 5a: Auto Mode** for all remaining unprocessed comments

**5.5 — Move to the next comment** and repeat from 5.1.

### Step 5a: Auto Mode

For each remaining unresolved comment:

1. **Launch the review-assessor agent** (same as Step 5.1)

2. **If verdict is VALID or PARTIAL AND confidence >= 70:**
   - Apply the fix using the Edit tool
   - Run `git add <file>` to stage the fixed file
   - Add the "fixed" reply to the reply queue

3. **If verdict is INTENTIONAL or confidence < 70:**
   - Add the "skipped/intentional" reply to the reply queue (will need user approval before posting)

4. **After ALL comments are assessed**, present a full summary:

```
## Auto-Assessment Complete

### Fixed (X issues):
1. [file:line] — [summary]
   Reply: "[queued reply]"

### Skipped — Intentional (X issues):
1. [file:line] — [summary]
   Draft reply: "[queued reply]"

Review staged changes with `git diff --cached`.
```

5. Proceed to **Step 6: Post Replies**.

### Step 6: Post Replies to GitHub

After all comments are processed (either interactive or auto mode):

1. **If the reply queue is empty**, skip this step.

2. **Show all queued replies** in a summary:

```
## Replies Ready to Post (X total)

1. [file:line] ([fixed/skipped]) → "[reply text]"
2. [file:line] ([fixed/skipped]) → "[reply text]"
...
```

3. **Ask the user** using `AskUserQuestion`:
   - "Post all replies" — Post every queued reply to GitHub
   - "Review one by one" — Walk through each reply for individual approval
   - "Don't post any" — Skip posting entirely

4. **If "Post all replies":**

   For each queued reply, post using the GitHub REST API:
   ```bash
   gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments/COMMENT_DATABASE_ID/replies -f body='REPLY_TEXT'
   ```

   Replace `OWNER`, `REPO`, `PR_NUMBER`, `COMMENT_DATABASE_ID`, and `REPLY_TEXT` with the stored values.

   **IMPORTANT**: Use a Python one-liner or heredoc to safely escape the reply text in the API call. Do NOT pass raw text that might contain quotes or special characters directly in the shell command.

   Safe approach:
   ```bash
   python3 -c "
   import subprocess, json
   body = json.dumps({'body': '''REPLY_TEXT'''})
   subprocess.run(['gh', 'api', 'repos/OWNER/REPO/pulls/PR_NUMBER/comments/COMMENT_DATABASE_ID/replies', '--input', '-'], input=body.encode(), check=True)
   "
   ```

5. **If "Review one by one":**

   For each queued reply, show it and ask using `AskUserQuestion`:
   - "Post this reply" — Post it
   - "Edit reply" — Wait for user to provide new text, then post
   - "Skip this reply" — Don't post

6. **If "Don't post any":**
   - Note that no replies were posted
   - The user will reply manually on GitHub

7. **Report posting results:**

```
## Replies Posted

Posted X of Y replies to PR #N.
[List each posted reply with a link to the comment thread]
```

### Step 7: Final Summary

```
# PR Review Response Complete

## PR: [title] (#number)
## URL: [PR_URL]

## Fixed Issues (X) — staged, not committed
- [file:line]: [summary] — Reply: [posted ✓ / not posted]
...

## Skipped Issues (X)
- [file:line]: [summary] — Reply: [posted ✓ / not posted]
...

## Next Steps
- Review staged changes: `git diff --cached`
- Commit when ready: `git commit -m "address PR review feedback"`
- Push to update the PR: `git push`
```

## Usage Examples

**Interactive mode (default):**
```
/pr-review-responder:respond
```

**Auto-fix mode:**
```
/pr-review-responder:respond auto
```

## Tips

- **Start interactive, switch to auto**: If the agent's assessments look good after a few comments, choose "Auto-fix remaining" to speed things up
- **Review before pushing**: All fixes are staged but not committed — use `git diff --cached` to review everything
- **Edit replies**: You can customize any reply before it's posted to GitHub
- **Skip silently**: If you want to handle a comment yourself later, skip it without posting a reply
