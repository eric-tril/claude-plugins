---
description: Post a PR review request to Google Chat
allowed-tools: Bash(gh pr view:*), Bash(git branch:*), Bash(curl:*), Bash(cat:*), Bash(python3:*)
---

# Ping — Post PR Review Request to Google Chat

Post a rich card message to the team's Google Chat channel requesting a PR review.

## Step 0 — Load Configuration

Read the config from:

```bash
cat ~/.claude/config/ping-chat.json
```

Extract these values:

- `gchat_webhook_url` (required) — the Google Chat webhook URL
- `bot_name` (required) — character name (e.g., "Sackboy", "Kratos")
- `bot_catchphrase` (optional) — defaults to "requests a review!"
- `bot_avatar_url` (optional) — URL to the bot's avatar image (displayed in the card header)
- `theme` (optional) — card color theme. Options: `ocean` (default), `forest`, `sunset`, `slate`, `violet`
- `gchat_user_id` (optional) — your Google Chat user ID (e.g., `users/123456789`). When set, you'll be @mentioned in the posted message so you get notified when someone replies to the thread. To find your user ID: open [Google Chat](https://chat.google.com) in a browser, open Developer Tools (F12 / Cmd+Option+I), go to the Console tab, and run `document.querySelector('[data-member-id]')?.getAttribute('data-member-id')`. The result will contain your numeric ID — use just the numeric portion with a `users/` prefix (e.g., `users/109869351304660926740`).

If the file doesn't exist or `gchat_webhook_url` or `bot_name` is missing, stop and tell the user:

> "Missing config. Create `~/.claude/config/ping-chat.json` with at minimum `gchat_webhook_url` and `bot_name` keys."

Store: `WEBHOOK_URL`, `BOT_NAME`, `BOT_CATCHPHRASE`, `BOT_AVATAR_URL`, `THEME`, `GCHAT_USER_ID`.

## Step 1 — Gather PR Details

Check the recent conversation context for any GitHub PR links (e.g., `https://github.com/.../pull/123`).

If no PR link is found in the conversation, run:

```bash
gh pr view --json url,title,headRefName,number,additions,deletions,author
```

If no open PR exists for the current branch, stop and tell the user:

> "No PR found. Please open a PR first or provide a PR link, then re-run /ping-chat:send."

Store: `PR_URL`, `PR_TITLE`, `PR_NUMBER`, `AUTHOR` (the `author.login` value), `HEAD_BRANCH`, `ADDITIONS`, `DELETIONS`.

## Step 2 — Extract the KLAIR ID

Try to extract a `KLAIR-<number>` identifier:

1. **PR title first**: Look for `KLAIR-\d+` in `PR_TITLE`.
2. **Branch name fallback**: Look for `KLAIR-\d+` (case-insensitive) in `HEAD_BRANCH`.
3. **If neither works**: Use `AskUserQuestion` to ask the user to provide the KLAIR linear ID explicitly.

Store: `KLAIR_ID` (e.g., `KLAIR-1993`).

## Step 3 — Determine the display label

Use `KLAIR_ID: PR_TITLE` as the label. If the PR title already starts with the KLAIR ID, just use the PR title as-is to avoid duplication.

## Step 4 — Write the AI Summary

Write a concise 1-3 sentence summary of the PR's changes. Use the conversation context (what you've been working on with the user) to write a clear, technical summary of what the PR does and why. This should read like a brief explanation for a reviewer.

Store: `AI_SUMMARY`.

## Step 5 — Post Rich Card to Google Chat

Use Python to safely build the JSON payload and send it. This avoids shell escaping issues with special characters in PR titles or summaries.

```bash
python3 << 'PYEOF'
import json, subprocess, sys

# Variables — Claude fills these in from previous steps
webhook_url = "WEBHOOK_URL_VALUE"
bot_name = "BOT_NAME_VALUE"
catchphrase = "BOT_CATCHPHRASE_VALUE"
avatar_url = "BOT_AVATAR_URL_VALUE"
theme_name = "THEME_VALUE"
gchat_user_id = "GCHAT_USER_ID_VALUE"  # empty string if not configured

# Theme definitions — button color
themes = {
    "ocean":  {"red": 0.10, "green": 0.46, "blue": 0.82},
    "forest": {"red": 0.18, "green": 0.60, "blue": 0.33},
    "sunset": {"red": 0.90, "green": 0.49, "blue": 0.13},
    "slate":  {"red": 0.35, "green": 0.40, "blue": 0.45},
    "violet": {"red": 0.54, "green": 0.25, "blue": 0.87},
}
button_color = themes.get(theme_name, themes["ocean"])

pr_url = "PR_URL_VALUE"
pr_number = PR_NUMBER_VALUE
pr_title = "LABEL_VALUE"
author = "AUTHOR_VALUE"
additions = ADDITIONS_VALUE
deletions = DELETIONS_VALUE
ai_summary = """AI_SUMMARY_VALUE"""

# Build card header
header = {
    "title": f"{bot_name} {catchphrase}",
    "subtitle": f"PR #{pr_number}",
    "imageType": "CIRCLE"
}
if avatar_url:
    header["imageUrl"] = avatar_url

# Build card sections
diff_stats = f'<font color="#1a7f37">+{additions}</font> / <font color="#cf222e">\u2212{deletions}</font>'
link_html = f'<a href="{pr_url}">{pr_title}</a>  {diff_stats}'

sections = [
    {"widgets": [{"textParagraph": {"text": link_html}}]}
]

if ai_summary.strip():
    sections.append({
        "widgets": [{"textParagraph": {"text": f"<b>Summary:</b> {ai_summary.strip()}"}}]
    })

sections.append({
    "widgets": [{
        "buttonList": {
            "buttons": [{
                "text": "Review PR",
                "icon": {"materialIcon": {"name": "rate_review"}},
                "onClick": {"openLink": {"url": pr_url}},
                "color": {**button_color, "alpha": 1}
            }]
        }
    }]
})

# Build full payload
text_preview = f"PR #{pr_number} \u2014 {pr_title} (by {author})"
if gchat_user_id:
    text_preview += f" <{gchat_user_id}>"

payload = {
    "text": text_preview,
    "cardsV2": [{
        "cardId": "pr-review-card",
        "card": {"header": header, "sections": sections}
    }]
}

# Send
with open("/tmp/gchat_payload.json", "w") as f:
    json.dump(payload, f)

result = subprocess.run(
    ["curl", "-s", "-X", "POST", webhook_url,
     "-H", "Content-Type: application/json",
     "-d", "@/tmp/gchat_payload.json"],
    capture_output=True, text=True
)
print(result.stdout)
if result.returncode != 0:
    print(result.stderr, file=sys.stderr)
    sys.exit(1)
PYEOF
```

**Important**: Claude replaces all `_VALUE` placeholders with actual values from previous steps. Python handles all JSON escaping safely — no manual escaping needed.

## Step 6 — Confirm

Tell the user the message was posted successfully and show them a summary of what was sent (bot name, PR title, AI summary preview).
