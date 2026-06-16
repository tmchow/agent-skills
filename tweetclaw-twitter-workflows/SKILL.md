---
name: tweetclaw-twitter-workflows
description: >-
  This skill should be used when the user names TweetClaw, @xquik/tweetclaw,
  or asks to use an already configured TweetClaw/OpenClaw setup for X/Twitter
  source evidence, tweet search, reply search, user lookup, follower export,
  media workflows, monitors, webhooks, giveaway draws, or reviewed posting.
  Do NOT use for generic social-media strategy, drafting, scheduling, or
  analytics when TweetClaw is not part of the workflow.
version: 0.1.0
author: Trevin Chow
license: MIT
metadata:
  hermes:
    tags: [tweetclaw, x, twitter, social-media, openclaw, source-evidence]
    category: social-media
    requires_toolsets: [terminal]
  openclaw:
    homepage: https://github.com/Xquik-dev/tweetclaw
---

# TweetClaw Twitter Workflows

Use this skill only when TweetClaw is already installed or the user explicitly
asks to use TweetClaw. TweetClaw is the source tool; this skill is the workflow
guardrail for collecting X/Twitter evidence and keeping side effects reviewed.

## Setup Check

Confirm these before using TweetClaw:

- TweetClaw is installed from https://github.com/Xquik-dev/tweetclaw or
  `@xquik/tweetclaw`.
- The user has completed TweetClaw credential setup outside this skill.
- The task names the desired X/Twitter job: search tweets, search tweet replies,
  scrape tweets, user lookup, follower export, media upload or download, direct
  messages, monitors, webhooks, giveaway draws, post tweets, or post replies.
- The user understands whether the workflow is read-only evidence collection or
  an approval-gated write action.

Do not ask the user to paste credentials into chat, terminal output, issue text,
or documents. Do not read, print, transform, or store secret values. If a
credential appears in output, treat it as compromised and ask the owner to
rotate it.

## Read-Only Evidence Workflow

For search, lookup, export, media download, and monitoring research:

1. Restate the exact query, account, tweet URL, date range, and output shape.
2. Prefer TweetClaw's current tool help, plugin inspection, or README for exact
   command and tool names.
3. Collect source packets with enough provenance to review later.
4. Keep drafts, scoring, analytics, scheduling, and publishing in the target
   workflow. TweetClaw supplies source evidence, not final judgment.
5. Cite only public source identifiers and retrieved facts. Do not include raw
   cookies, account secrets, or private session material.

Read `references/source-packets.md` before preparing a report, scorecard,
handoff, or content draft from TweetClaw output.

## Write Or Recurring Actions

For post tweets, post replies, media upload, direct messages, monitor setup,
webhooks, and giveaway draws:

1. Stop before execution and ask for explicit approval of the exact action.
2. Show the account, action type, target, payload summary, and expected side
   effect.
3. Use one-time approval only. Do not turn one approval into persistent
   permission for future writes or recurring jobs.
4. After execution, report the stable public identifier or delivery status.

If the user asks for automation that repeats later, make the recurrence and
stop conditions explicit before scheduling or configuring it.

## When Not To Use

Do not use TweetClaw for:

- Generic social-media advice without a TweetClaw task.
- Impersonation, harassment, spam, or evasive account activity.
- Hidden scraping or publishing that bypasses account owner approval.
- Credential recovery, cookie extraction, or raw session handling.
- Claims about X/Twitter data that were not retrieved or verified.

## Handoff Shape

For any downstream agent or human reviewer, include:

- the TweetClaw task and tool used
- query or target identifiers
- capture time
- source packet count
- important limits, missing fields, or failed fetches
- whether any write-like action was requested, approved, denied, or not run
