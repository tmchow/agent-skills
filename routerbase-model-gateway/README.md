# routerbase-model-gateway

Agent skill for using [routerbase](https://routerbase.com/) as an OpenAI-compatible model gateway: migrate existing SDK calls, choose model-routing fallbacks, and keep RouterBase credentials out of chats, logs, screenshots, and public repos.

## Prerequisites

- A RouterBase account and API key.
- An OpenAI-compatible client such as the OpenAI Python or JavaScript SDK.
- A local credentials file created by the user, when live API calls are needed:
  `~/.config/routerbase/credentials.json`.

The agent-facing skill tells agents not to ask for keys in chat and not to read shared environment variables for RouterBase credentials.

## Install This Skill

### Any Agent Skills runtime (skills CLI)

Install only this skill from the repo:

```bash
npx skills add tmchow/agent-skills --skill routerbase-model-gateway
```

Add `--global` to install it at the user level instead of the current project:

```bash
npx skills add tmchow/agent-skills --skill routerbase-model-gateway --global
```

### Hermes

Install from the GitHub directory identifier:

```bash
hermes skills install tmchow/agent-skills/routerbase-model-gateway
```

From an interactive Hermes CLI session, use the slash command path:

```text
/skills install tmchow/agent-skills/routerbase-model-gateway
/reload-skills
/skill routerbase-model-gateway
```

Use `/reload-skills` if installing into an already-running session; then load it with `/skill routerbase-model-gateway` when needed.

### OpenClaw

After this repo publishes the skill to ClawHub, install it with:

```bash
openclaw skills install routerbase-model-gateway
```

Provisional ClawHub page: https://clawhub.ai/tmchow/routerbase-model-gateway

## What the skill teaches agents

- How to scope RouterBase-specific tasks instead of hijacking generic model-selection requests.
- How to change OpenAI-compatible clients to `https://routerbase.com/v1`.
- How to handle credentials through a user-owned local config file.
- How to choose primary and fallback models by modality, latency, quality, and cost constraints.
- How to validate streaming, tool calling, JSON mode, vision, media, audio, and embeddings before production claims.

`SKILL.md` is the agent-facing instruction set; you don't need to read it to use the skill.

## License & upstream

This skill: [MIT](../LICENSE) © Trevin Chow.
RouterBase skill source: [zenlee123/routerbase-agent-skills](https://github.com/zenlee123/routerbase-agent-skills).
