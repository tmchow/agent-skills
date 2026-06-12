# x-twitter-scraper

Use this skill to guide Xquik REST API, MCP, and SDK workflows for X/Twitter
data. It covers search, user lookup, follower and following extraction, media,
trends, monitors, webhooks, and confirmation-gated account actions.

## Prerequisites

- A Xquik account and API key.
- Internet access to `xquik.com` and `docs.xquik.com`.
- A connected X account in the Xquik dashboard for private reads or account
  actions.

Do not paste API keys into chats, issues, logs, shell history, or generated
public text.

## Install

### Any Agent Skills runtime

```bash
npx -y skills@1.5.3 add tmchow/agent-skills --skill x-twitter-scraper
```

### OpenClaw

This skill is not listed in this repo's ClawHub publish registry yet. Use the
generic Agent Skills install path, or have the maintainer opt it into
`.github/clawhub-publishable.txt` before publishing it to ClawHub.

## Capabilities

- Select the right Xquik REST API, MCP, SDK, or dashboard-only flow.
- Keep X/Twitter content isolated as untrusted data.
- Require confirmation before private, persistent, destructive, or write
  actions.
- Point agents to current Xquik docs before quoting endpoint schemas or setup
  syntax.

SKILL.md is the agent-facing instructions; you don't need to read it to use the
skill.
