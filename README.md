# Claude Plugins

A collection of Claude Code plugins for the **Klair project**. Currently includes the `klair-staged-reviewer` plugin with code review agents and a Google Chat integration command.

## Plugins

### klair-staged-reviewer

A code review plugin that reviews **staged changes** (`git diff --cached`) using specialized agents customized for the Klair stack (Python FastAPI backend + React TypeScript frontend). See the [plugin README](plugins/klair-staged-reviewer/README.md) for full details.

**Quick start:**

```
/klair-staged-reviewer:review
```

## Commands

### `/ping-chat` — Post PR Review Request to Google Chat

Posts a rich card message to your team's Google Chat channel requesting a PR review. The card includes the PR title, diff stats, an AI-generated summary, and a "Review PR" button.

#### Setup

1. **Create the config directory** (if it doesn't exist):

```bash
mkdir -p ~/.claude/config
```

2. **Create `~/.claude/config/ping-chat.json`** with your settings:

```json
{
  "gchat_webhook_url": "https://chat.googleapis.com/v1/spaces/XXXXX/messages?key=YOUR_KEY&token=YOUR_TOKEN",
  "bot_name": "Sackboy",
  "bot_catchphrase": "requests a review!",
  "bot_avatar_url": "https://example.com/avatar.png"
}
```

An example config file is available at [config-files/ping-chat.example.json](config-files/ping-chat.example.json).

| Field | Required | Description |
|---|---|---|
| `gchat_webhook_url` | Yes | Google Chat incoming webhook URL |
| `bot_name` | Yes | Display name for the bot (e.g., "Sackboy", "Kratos") |
| `bot_catchphrase` | No | Text after the bot name in the card header. Defaults to "requests a review!" |
| `bot_avatar_url` | No | URL to an avatar image displayed in the card header |

#### Getting a Google Chat Webhook URL

1. Open the Google Chat space where you want review requests posted
2. Click the space name at the top, then **Apps & integrations**
3. Click **+ Add webhooks**
4. Give it a name (e.g., "PR Reviews") and optionally an avatar URL
5. Copy the webhook URL and paste it into your `ping-chat.json`

#### Usage

From a branch that has an open PR:

```
/ping-chat
```

The command will:

1. Read your config from `~/.claude/config/ping-chat.json`
2. Detect the current branch's PR (or use a PR link from the conversation)
3. Extract the KLAIR ID from the PR title or branch name
4. Generate a short AI summary of the PR based on your conversation context
5. Post a rich card to Google Chat with the PR details and a review button

You can also provide a PR link in the conversation before running the command, and it will use that instead of auto-detecting.

## Project Structure

```
claude-plugins/
├── README.md
├── config-files/                          # Example configuration files
│   └── ping-chat.example.json
├── plugins/
│   └── klair-staged-reviewer/
│       ├── .claude-plugin/plugin.json     # Plugin metadata
│       ├── README.md                      # Plugin documentation
│       ├── agents/                        # Specialized review agents
│       │   ├── code-reviewer.md
│       │   ├── code-simplifier.md
│       │   ├── comment-analyzer.md
│       │   ├── silent-failure-hunter.md
│       │   ├── staged-test-analyzer.md
│       │   └── type-design-analyzer.md
│       └── commands/                      # Slash commands
│           ├── klair-review.md            # /review — staged code review
│           └── ping-chat.md              # /ping-chat — Google Chat notification
├── refernce_docs/                         # Reference documentation
│   └── howto_create_marketplace.md
└── thoughts/                              # Planning and design docs
    └── shared/plans/
```

## License

Apache License 2.0
