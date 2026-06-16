# tweetclaw-twitter-workflows

Agent skill for using [TweetClaw](https://github.com/Xquik-dev/tweetclaw) as a
reviewed X/Twitter source-evidence layer. It helps agents collect tweet search,
reply search, scrape tweets, user lookup, follower export, media workflow,
monitor, webhook, giveaway draw, post tweet, and post reply context while
keeping credentials private and side effects approval-gated.

## Prerequisites

- TweetClaw installed and configured by the account owner.
- Access to the runtime where TweetClaw is available, such as OpenClaw with the
  TweetClaw plugin installed.
- Account-owner approval before any posting, reply, direct-message, media
  upload, monitor, webhook, or giveaway action.

Install the TweetClaw OpenClaw plugin separately when needed:

```bash
openclaw plugins install npm:@xquik/tweetclaw
```

## Install

### Agent Skills CLI

```bash
npx skills add tmchow/agent-skills --skill tweetclaw-twitter-workflows --global
```

### Hermes

```bash
hermes skills install tmchow/agent-skills/tweetclaw-twitter-workflows
```

From an interactive Hermes session:

```text
/skills install tmchow/agent-skills/tweetclaw-twitter-workflows
/reload-skills
/skill tweetclaw-twitter-workflows
```

### OpenClaw

After this skill is published to ClawHub, install it with:

```bash
openclaw skills install tweetclaw-twitter-workflows
```

ClawHub page, after publish:
https://clawhub.ai/tmchow/tweetclaw-twitter-workflows

## Capabilities

- Plan reviewed TweetClaw source-evidence runs for X/Twitter workflows
- Separate read-only evidence collection from write-like side effects
- Package tweet, reply, user, follower, media, monitor, webhook, and giveaway
  outputs into reviewable source packets
- Keep drafting, scoring, scheduling, analytics, and publishing decisions in the
  user's chosen workflow
- Require one-time approval before posting, replying, direct messages, media
  upload, monitor setup, webhooks, or giveaway draws

SKILL.md is the agent-facing instructions; you don't need to read it to use the
skill.

## License & upstream

This skill: [MIT](../LICENSE) © Trevin Chow.
TweetClaw: [Xquik-dev/tweetclaw](https://github.com/Xquik-dev/tweetclaw).
