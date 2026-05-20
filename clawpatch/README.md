# clawpatch

An agent skill that drives the [Clawpatch](https://clawpatch.ai) CLI
(`openclaw/clawpatch`) — automated AI code review that produces concrete,
locatable findings, then fixes them with explicit, validated, per-finding
patches. The skill teaches your coding agent how to run Clawpatch, read its
findings, choose a fix strategy, and avoid its sharp edges.

## What you get

- Whole-repo or scoped review that yields structured findings (severity,
  category, evidence, suggested fix), not vibes-level prose.
- Two fix workflows with a clear decision guide:
  - **Scanner-only** (default) — dispatch subagents to fix findings in
    parallel, each in its own worktree, using your agent's own tooling.
  - **Full-cycle** — Clawpatch's own `fix` loop, gated by
    format/typecheck/lint/test validation.
- Accurate wire-format JSON parsing, exit-code handling, and built-in guards
  against the common pitfalls (acting on false positives, over-correction,
  parallel-state hazards).

## Prerequisites

- **Node.js + npm** — Clawpatch is an npm package (`npm install -g clawpatch`).
- **A coding-agent provider CLI** — one of `codex` (default), `grok`,
  `opencode`, `pi`, or `acpx`.

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
