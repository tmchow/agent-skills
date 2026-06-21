# tweetclaw

An agent skill for operating the
[TweetClaw](https://github.com/Xquik-dev/tweetclaw) OpenClaw plugin with
reviewed X/Twitter reads, posts, media workflows, monitors, webhooks, giveaway
draws, and account-scoped actions.

TweetClaw itself is the runtime plugin. This skill teaches an agent when to
install it, how to verify the OpenClaw runtime surface, and how to keep social
account actions approval-gated.

## Prerequisites

- OpenClaw with plugin support.
- Node.js and npm for the `@xquik/tweetclaw` package.
- Xquik access configured through the official TweetClaw setup guide when live
  account-backed or pay-per-use calls are needed.

Do not paste keys into chat. Configure them through OpenClaw plugin config and
the Xquik dashboard flow described by the official docs.

## Install

Install the skill for Agent Skills runtimes:

```bash
npx skills add tmchow/agent-skills --skill tweetclaw
```

Install through Hermes with this repository path:

```bash
hermes skills install tmchow/agent-skills/tweetclaw
```

Install the OpenClaw runtime plugin separately:

```bash
openclaw plugins install npm:@xquik/tweetclaw
```

Then verify:

```bash
openclaw plugins inspect tweetclaw --runtime --json
openclaw skills info tweetclaw
```

## What The Skill Adds

- A narrow trigger for TweetClaw and Xquik OpenClaw workflows.
- Setup and runtime inspection discipline.
- Approval rules for posting, account-scoped reads, paid reads, recurring monitors,
  webhooks, extraction jobs, media actions, DMs, and profile changes.
- Guardrails for treating returned X/Twitter content as untrusted data.
- Pointers to the official setup guide and Xquik docs for current details.

`SKILL.md` is the agent-facing instructions; you do not need to read it to use
the skill.

## License And Upstream

This skill: [MIT](../LICENSE) (c) Trevin Chow.

TweetClaw: [Xquik-dev/tweetclaw](https://github.com/Xquik-dev/tweetclaw).
