# clawpatch

An agent skill that lets your coding agent operate the
[Clawpatch](https://clawpatch.ai) CLI (`openclaw/clawpatch`) — running its
automated code review, reading the findings, choosing how to fix them, and
steering around its sharp edges.

**Clawpatch is the engine; this skill is the operating manual.** Clawpatch
itself is a terminal CLI that reviews a repo into structured findings and can
apply validated per-finding fixes. Handed to an agent cold, it gets fumbled:
misread JSON, the wrong fix workflow, misleading hints followed, parallel-state
hazards hit. This skill is the procedural knowledge that makes an agent drive
it well. Installing it doesn't install Clawpatch (see Prerequisites) — it
teaches your agent to use it.

## What the skill adds

With it installed, your agent knows how to:

- **Run and read the review** — drive `init → map → review → report` and parse
  Clawpatch's *actual* wire-format JSON (which differs from its docs) instead
  of guessing at fields.
- **Pick the right fix strategy** — choose deliberately between *scanner-only*
  (fix findings in parallel via subagents, each in its own worktree, using your
  agent's own tooling) and *full-cycle* (Clawpatch's own `fix` loop, gated by
  format/typecheck/lint/test), based on the repo and the task.
- **Avoid the footguns** — not act on false positives, not over-correct past a
  finding's scope, ignore Clawpatch's misleading `next:` hint, and never race
  `clawpatch fix` on shared state.
- **Hand off cleanly** — turn findings into commits/PRs via your agent's own
  workflow or Clawpatch's `open-pr`, with the right context in each.

## Prerequisites

- **Node.js + npm** — Clawpatch is an npm package (`npm install -g clawpatch`).
- **A coding-agent provider CLI** — one of `codex` (default), `claude` (routes
  through your local Claude Code CLI), `cursor`, `grok`, `opencode`, `pi`, or
  `acpx`.

The skill checks both with `clawpatch doctor` and walks you through install if
either is missing.

## Install

```bash
npx skills add tmchow/agent-skills --skill clawpatch --global
```

Update later with `npx skills update clawpatch`.

## Using it

Install, then ask your agent things like *"review with clawpatch"*,
*"clawpatch fix the auth findings"*, or *"dispatch subagents to fix the
clawpatch findings in parallel."* The skill triggers only when Clawpatch is
named — it won't hijack generic "review my code" requests.

`SKILL.md` is the agent-facing instruction set (it's what your agent reads).
You don't need to read it to use the skill.

## License & upstream

This skill: [MIT](../LICENSE) © Trevin Chow.
Clawpatch itself: [openclaw/clawpatch](https://github.com/openclaw/clawpatch).
