# Claude Plugins

A Claude Code plugin marketplace for the **Klair project**. Includes code review agents and a Google Chat integration.

## Installation

### Step 1: Add the marketplace

From within Claude Code, run:

```
/plugin marketplace add https://github.com/eric-tril/claude-plugins.git
```

This registers the marketplace so you can browse and install its plugins. No plugins are installed yet.

### Step 2: Install plugins

Install the plugins you want:

```
/plugin install klair-staged-reviewer@eric-trilogy-plugins
/plugin install ping-chat@eric-trilogy-plugins
```

Or browse available plugins interactively with `/plugin` and go to the **Discover** tab.

### Team setup (optional)

To auto-prompt teammates to install this marketplace when they clone a project, add this to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "eric-trilogy-plugins": {
      "source": {
        "source": "url",
        "url": "https://github.com/eric-tril/claude-plugins.git"
      }
    }
  }
}
```

## Plugins

### klair-staged-reviewer

Reviews **staged changes** (`git diff --cached`) using specialized agents customized for the Klair stack (Python FastAPI backend + React TypeScript frontend). See the [plugin README](plugins/klair-staged-reviewer/README.md) for full details.

```
/klair-staged-reviewer:review
```

**Agents included:** code-reviewer, code-simplifier, comment-analyzer, staged-test-analyzer, silent-failure-hunter, type-design-analyzer

### ping-chat

Posts a rich card message to your team's Google Chat channel requesting a PR review. Includes PR title, diff stats, an AI-generated summary, and a "Review PR" button.

```
/ping-chat:send
```

#### Setup

1. Create `~/.claude/config/ping-chat.json`:

```json
{
  "gchat_webhook_url": "https://chat.googleapis.com/v1/spaces/XXXXX/messages?key=YOUR_KEY&token=YOUR_TOKEN",
  "bot_name": "Sackboy",
  "bot_catchphrase": "requests a review!",
  "bot_avatar_url": "https://example.com/avatar.png"
}
```

An example config is at [config-files/ping-chat.example.json](config-files/ping-chat.example.json).

| Field | Required | Description |
|---|---|---|
| `gchat_webhook_url` | Yes | Google Chat incoming webhook URL |
| `bot_name` | Yes | Display name for the bot (e.g., "Sackboy", "Kratos") |
| `bot_catchphrase` | No | Text after the bot name in the card header. Defaults to "requests a review!" |
| `bot_avatar_url` | No | URL to an avatar image displayed in the card header |
| `theme` | No | Card color theme: `ocean` (default), `forest`, `sunset`, `slate`, `violet` |

## Project Structure

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json                  # Marketplace catalog
├── README.md
├── config-files/
│   └── ping-chat.example.json            # Example configuration
├── plugins/
│   ├── klair-staged-reviewer/
│   │   ├── .claude-plugin/plugin.json    # Plugin metadata
│   │   ├── README.md                     # Plugin documentation
│   │   ├── LICENSE
│   │   ├── agents/                       # Specialized review agents
│   │   │   ├── code-reviewer.md
│   │   │   ├── code-simplifier.md
│   │   │   ├── comment-analyzer.md
│   │   │   ├── silent-failure-hunter.md
│   │   │   ├── staged-test-analyzer.md
│   │   │   └── type-design-analyzer.md
│   │   └── skills/
│   │       └── review/
│   │           └── SKILL.md              # /klair-staged-reviewer:review
│   └── ping-chat/
│       ├── .claude-plugin/plugin.json    # Plugin metadata
│       └── skills/
│           └── send/
│               └── SKILL.md              # /ping-chat:send
└── thoughts/
    └── shared/plans/
```

## License

Apache License 2.0
