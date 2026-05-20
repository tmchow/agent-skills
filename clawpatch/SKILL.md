---
name: clawpatch
description: >-
  This skill is specifically for the Clawpatch CLI (openclaw/clawpatch,
  https://clawpatch.ai) — an npm-installed automated code-review and
  per-finding fix tool. Use it ONLY when Clawpatch itself is in play: the user
  says "review with clawpatch", "run clawpatch", "clawpatch fix", "find bugs
  with clawpatch", "clawpatch report/findings", "clawpatch open-pr", wants to
  dispatch subagents to fix Clawpatch findings in parallel, or otherwise names
  Clawpatch or one of its commands. Do NOT use it for generic "review this
  code", "find bugs", or "code review" requests that don't involve Clawpatch —
  those belong to a different tool. When Clawpatch is the tool, it covers
  install/doctor, the lifecycle (init → map → review → report → fix →
  revalidate → open-pr/land), the wire-format JSON schema, the
  scanner-only-vs-full-cycle mode decision, parallel subagent dispatch,
  exit-code handling, and PR handoff.
version: 0.1.0
metadata:
  openclaw:
    homepage: https://clawpatch.ai
    emoji: "🤓"
    install:
      - id: npm
        kind: node
        package: clawpatch
        bins: [clawpatch]
        label: Install Clawpatch (npm)
    envVars:
      - name: CLAWPATCH_PROVIDER
        required: false
        description: Override the default review/fix provider (codex). One of codex, grok, opencode, pi, acpx, mock.
      - name: CLAWPATCH_MODEL
        required: false
        description: Provider-specific model string (e.g. opencode/big-pickle, anthropic/claude-sonnet-4).
      - name: CLAWPATCH_REASONING_EFFORT
        required: false
        description: Codex reasoning level (low/medium/high/xhigh).
      - name: ANTHROPIC_API_KEY
        required: false
        description: Only required when CLAWPATCH_PROVIDER=pi.
  hermes:
    tags: [code-review, clawpatch, cli, bug-fixing, automation]
    category: code-review
    requires_toolsets: [terminal]
---

<!-- TOC: When to use | Agent flags | Install & verify | State directory | Lifecycle | Reviewing | Findings JSON | Presenting | Choose mode | Scanner-only mode | Full-cycle mode (fix / PR handoff / batch) | Exit codes | Safety | Anti-patterns | Harness notes -->

# Clawpatch — agent workflow

> **What it is.** Clawpatch (`openclaw/clawpatch`, https://clawpatch.ai) is an
> npm-installed CLI that maps a repo into semantic feature slices, asks a
> coding-agent provider (Codex CLI by default; also Grok, OpenCode, Pi, ACPX)
> to review each slice, persists findings under `.clawpatch/`, and runs an
> explicit per-finding fix loop with format → typecheck → lint → test
> validation. It does not auto-commit and refuses dirty worktrees by default.
>
> **What this skill adds.** Reliable agent operation: detection-first install,
> flag conventions for non-interactive use, accurate wire-format JSON schema
> for parsing findings, the scanner-only-vs-full-cycle mode decision that
> determines *how* findings get fixed, exit-code handling, and clean handoffs
> to a commit/PR step. The throughline: Clawpatch *identifies* issues well;
> this skill arms the agent to *fix* them without tripping its limitations.
>
> **Source of truth.** The Clawpatch spec at
> https://github.com/openclaw/clawpatch/blob/main/docs/spec.md is authoritative
> for command shapes, schemas, and exit codes. If anything in this skill
> conflicts with `clawpatch <cmd> --help` on the user's machine, trust the
> help output — Clawpatch evolves and this skill may lag.

## When to use this skill

Use Clawpatch when **any** of these hold:

- The user asks for an automated code review across a whole repo or a large
  diff and wants concrete, locatable findings, not vibes-level prose.
- The user wants AI-driven fixes gated by format/lint/test validation, with
  one finding fixed at a time so they can audit each patch.
- The user mentions Clawpatch by name.

Do **not** use Clawpatch when:

- The change in question is a small diff already under review. A whole-repo
  feature-mapping + review cycle is overkill — use a focused diff-review tool
  or the harness's native code-review helper.
- The user already knows the bug and wants you to write the fix yourself.
  Clawpatch's value is the *finding catalog*, not the patch.
- No supported coding-agent provider is installed (`codex`, `grok`,
  `opencode`, `pi`, or `acpx`). `clawpatch doctor` will fail fast — tell the
  user, stop, and do not improvise.

## Agent-friendly flag conventions

Two of these globals are safe to pass on *every* invocation; the rest are
context-dependent. The examples in this skill apply each flag only where
it's relevant, not uniformly.

**Pass on every command:**

| Flag         | Why                                                        |
| ------------ | ---------------------------------------------------------- |
| `--no-input` | Refuses any interactive prompt; fails fast instead of blocking. |
| `--no-color` | Strips ANSI; keeps output clean for parsing.               |

**Pass only when relevant:**

| Flag           | When                                                                 |
| -------------- | -------------------------------------------------------------------- |
| `--json`       | Only on commands whose stdout is parsed (`report`, `status`, `doctor`, `review`). Meaningless on `init`, `map`, `fix`, `open-pr`. |
| `--quiet`      | When the human progress stream on stderr isn't wanted. Skip it while debugging a failing command — the progress lines are diagnostic. |
| `--root <dir>` | Only when running against a repo other than the current working directory. Omit when already in the repo. |
| `--yes`        | Only on interactive-by-default commands (`fix`, `clean-locks`), and only after the user has approved the action. |

Not every command supports every global, and the flag surface shifts
between Clawpatch versions. If a flag errors with exit 2 (invalid usage),
drop it or check `clawpatch <cmd> --help` — don't assume a global is
universal.

## Install & verify

The first time you touch Clawpatch in a session, verify the install. Don't
assume — `clawpatch` may not be on PATH, or `doctor` may flag a missing
provider:

```bash
clawpatch --version
clawpatch doctor --json --no-input --no-color
```

If `clawpatch` is missing, install globally with whichever npm-family
package manager the host uses. Prefer `npm` unless the repo clearly uses
`pnpm` or `yarn`:

```bash
npm install -g clawpatch
# or:
pnpm add -g clawpatch
```

The default provider is the Codex CLI. If `doctor` reports the provider as
missing or unauthenticated, the user installs/authenticates it themselves —
do **not** run an auth command on their behalf without explicit permission,
because login flows touch credentials. Typical fix the user runs:

```bash
brew install codex   # macOS; other platforms have their own install
codex login          # interactive
```

To use a non-default provider, the user (or you, with their permission)
sets the env var or passes `--provider <name>` per command:

```
CLAWPATCH_PROVIDER=opencode
CLAWPATCH_MODEL=opencode/big-pickle
CLAWPATCH_REASONING_EFFORT=high       # Codex only
CLAWPATCH_PROVIDER_TIMEOUT_MS=120000  # generic timeout
```

## The `.clawpatch/` state directory

Treat `.clawpatch/` as **per-developer, disposable state** — not a
shared artifact. Each developer (or agent run) generates their own; the
directory is gitignored as a whole and rebuilt fresh by `init && map`
on demand. This is a deliberate simplification of the upstream model
(which tiers durability by subdir); it trades the "share config across
team" benefit for one-rule lifecycle simplicity.

After `init` runs, the layout is:

```
.clawpatch/
├── config.json     # provider, validators, include/exclude
├── project.json    # detected git remote, branch, languages
├── features/       # one JSON file per feature; populated by `map`
├── runs/           # per-invocation run records
├── findings/       # one JSON file per finding; regenerated by `review`
├── patches/        # patch attempt records (full-cycle mode only)
├── reports/        # generated markdown reports
└── locks/          # feature-level locks for in-flight commands
```

(`init` itself only prints a short success message — `created: true\nnext:
clawpatch map` — the layout is what's *on disk*, not what's printed.
Inspect with `ls .clawpatch/`.)

### Gitignore policy

One line in the repo's `.gitignore`:

```
.clawpatch/
```

That's it. Don't add the upstream's tiered subdir entries; they're for a
team-shared-config workflow this skill doesn't use.

### Gitignore preflight

Run this check **only when the user is about to commit or invoke a PR-
creating command** (full-cycle `open-pr`, a full-cycle Pattern B commit,
or any commit a scanner-only subagent makes). It is wasted work for
read-only flows. This is a tracked file — **never modify it without
explicit user approval**; the check just gathers state to report.

```bash
GI=.gitignore
HAS_BLANKET=$(grep -qxF '.clawpatch/' "$GI" 2>/dev/null && echo yes || echo no)
HAS_LEGACY=$(grep -qE '^[[:space:]]*\.clawpatch/(runs|findings|patches|reports|locks|features)/' "$GI" 2>/dev/null && echo yes || echo no)

case "$HAS_BLANKET/$HAS_LEGACY" in
  yes/*)    : ;;  # good
  no/yes)   echo "WARN: $GI has tiered .clawpatch/ subdir entries but no blanket .clawpatch/ — legacy upstream pattern." ;;
  no/no)    echo "WARN: $GI does not gitignore .clawpatch/ at all — review state will leak into commits." ;;
esac
```

If the blanket entry is missing, surface a short note ("`.gitignore`
doesn't ignore `.clawpatch/` — want me to add the line?"). Wait for
explicit approval before editing. If the file has the legacy tiered
entries, offer to replace them with the single line. If the repo has
no `.gitignore` at all, ask whether to create one rather than silently
generating a tracked file.

## Lifecycle

The supported lifecycle is fixed. Each step is resumable: state lives on
disk, so you can pick up at any point without re-running prior steps.

```
init → map → review → report → (triage) → fix → (revalidate) → open-pr | <handoff>
                                                              → land (user-driven only)
```

This is Clawpatch's full *native* pipeline. **How much of it you actually
run depends on the mode you pick** (*Choose your mode*, below): the
default scanner-only mode uses only `init → map → review → report`, then
hands findings to subagents that fix without Clawpatch; full-cycle mode
runs the whole chain through `fix`/`open-pr`. Identify is shared; the
fix half forks by mode.

Because state is disposable, the bootstrap is a single rule: **always
run `init && map` at the start of any review session.** No "is
`.clawpatch/` missing?" check needed. Both commands are idempotent on
existing state and cheap (no provider calls — `init` detects
git/languages/validators; `map` classifies files into features). On a
fresh checkout they set everything up; on an existing `.clawpatch/`
they're no-ops or quick refreshes.

```bash
clawpatch init --no-input --no-color
clawpatch map  --no-input --no-color
clawpatch status --json --no-input --no-color   # optional — see where we are
```

Pass `--force` to `init` only when the user explicitly wants to
overwrite an existing `project.json`/`config.json`.

The **gitignore preflight** (state-directory section above) runs *only*
before a commit-or-PR action — not on `init` or `review`. Read-only
flows don't need it. When the user is about to commit (full-cycle
`open-pr`, full-cycle Pattern B commit, or a scanner-only subagent
push), check whether `.clawpatch/` is gitignored before the commit
lands.

### Cleanup

Because state is disposable, cleanup is the same one-liner regardless
of mode. Run it at end-of-session or before a fresh review cycle:

```bash
[ -d .clawpatch ] && rm -r .clawpatch
```

The existence guard makes the call idempotent (no error if already
gone). **Do not use `rm -rf`** — `-r` does the recursive delete; `-f`
adds "ignore all errors, never prompt" semantics that mask mistakes
(wrong path, wrong cwd, accidental variable expansion). The guard
covers the same idempotence need without that risk.

If team policy forbids any `rm -r`, use `find .clawpatch -mindepth 1
-delete` to clear contents (leaves the dir itself) at equivalent cost
— `.clawpatch/` is typically kilobytes to a few MB, so neither
approach has a meaningful performance difference.

## Reviewing the codebase

`map` is local and cheap — it classifies files into features. Re-run when
the repo has changed materially since the last run:

```bash
clawpatch map --no-input --no-color
```

`review` is the expensive step (one provider call per feature). Rough
ballpark with `codex` + `gpt-5`: **~30–60 seconds per feature**. A 40-
feature repo is ~20–30 minutes of wall clock and proportional provider
spend. Treat the smoke test as the default outcome, not a step on the
way to a full sweep:

```bash
# Smoke test on 3 features
clawpatch review --limit 3 --json --no-input --no-color

# Targeted: one feature, one feature-kind, or resume an interrupted run
clawpatch review --feature <featureId> --json --no-input --no-color
clawpatch review --kind service     --json --no-input --no-color
clawpatch review --resume <runId>   --json --no-input --no-color

# Full sweep — only after user explicitly opts in to the time/cost
clawpatch review --json --no-input --no-color
```

**Smoke-test findings are often sufficient.** Three features routinely
produces 5–10 actionable findings — hours of fix work. The skill biases
toward "smoke test → review findings → ship" rather than reflexively
escalating to a full sweep. Surface the time/cost estimate to the user
before any full-repo `review` call.

Notable `review` flags (verify with `clawpatch review --help` — surface area
changes between releases):

- `--feature <id>` — single feature.
- `--kind <featureKind>` — restrict to one kind (see *feature kinds* below).
- `--limit <n>` — cap features per run.
- `--resume <runId>` — continue a previously interrupted run.
- `--provider <name> --model <name> --reasoning-effort <level>` — per-call
  override without editing `config.json`.
- `--dry-run` — show what would be reviewed without calling the provider.

Provider-level concurrency (worker pool size) is configured in `config.json`
or via flag depending on the installed version. Run `clawpatch review --help`
to check; assume sequential if unsure.

After `review`, dedupe and prioritize before showing the user:

```bash
clawpatch triage --json --no-input --no-color
```

## Findings JSON — the schema

`clawpatch report --json` is the canonical machine-readable output. Always
prefer it to scraping the human report.

**`--json` writes the JSON envelope to *stdout*. `--output <path>` writes the
*Markdown* report — not JSON — to a file.** There is no `--format` flag. To
capture JSON to a file, redirect stdout; do **not** use `--output`:

```bash
clawpatch report --json --no-input --no-color > /tmp/clawpatch-findings.json
```

`report` flags (verified against Clawpatch 0.3.0 `--help` — re-check on
your version):

| Flag                  | Purpose                                                  |
| --------------------- | -------------------------------------------------------- |
| `--status <status>`   | Filter by finding status (`open`, `fixed`, …)            |
| `--severity <level>`  | Filter by severity                                       |
| `--category <cat>`    | Filter by category                                       |
| `--triage <triage>`   | Filter by triage assessment                              |
| `--feature <id>`      | Findings for one feature                                 |
| `--project <name>`    | Select project by name or root path                      |
| `--json`              | Emit the JSON envelope on **stdout** (redirect to save)  |
| `--output <path>`     | Write the **Markdown** report to a file (not JSON)       |

### `report --json` envelope and finding shape

The wire format is an envelope, not a bare array. Probe once with `jq`
before constructing recipes — fields differ from the internal TypeScript
types described in `spec.md`.

```jsonc
// Top-level shape of `clawpatch report --json`:
{
  "findings": 14,          // INTEGER count, not the items
  "items": [ ... ]         // the actual finding records live here
}
```

Each item in `items[]` has roughly this shape (the field names below are
from live `report --json` output, not `spec.md`'s internal TypeScript
types):

```jsonc
{
  "id": "<hash>",          // finding id; spec.md calls this `findingId`
  "featureId": "<id>",     // link to the feature
  "title": "...",
  "category": "...",       // bug | security | ... (see enum below)
  "severity": "...",       // critical | high | medium | low
  "confidence": "...",     // high | medium | low
  "status": "...",         // open | uncertain | fixed | wont-fix | false-positive
  "triage": "...",         // confirmed-bug | risk | contract-mismatch | false-positive | wont-fix
                           // (NOTE: different concept than the `clawpatch triage` command — see below.)
  "evidence": [
    { "path": "src/foo.ts", "startLine": 47, "endLine": 51,
      "symbol": "fooFn", "snippet": "..." }
  ],
  "reasoning": "...",
  "reproduction": "...",                     // how to observe the bug; may be null
  "recommendation": "...",                   // what to change
  "whyTestsDoNotAlreadyCoverThis": "...",    // gap analysis
  "suggestedRegressionTest": "...",          // concrete test to add
  "minimumFixScope": ["src/foo.ts"],         // file paths the fix should stay within
  "next": "clawpatch fix --finding <id>",    // hint — IGNORE for selection (see below)
  "createdAt": "...",
  "updatedAt": "..."
}
```

Fields **on the internal `FindingRecord` type but absent from the wire
output**: `schemaVersion`, `signature`, `linkedPatchAttemptIds`,
`createdByRunId`. Do not parse for them.

The four enrichment fields `reproduction`, `whyTestsDoNotAlreadyCoverThis`,
`suggestedRegressionTest`, and `minimumFixScope` are the goldmine for
fixing the issue. In scanner-only mode they go straight into the subagent
prompt (*Scanner-only mode: subagent dispatch*); in full-cycle mode they
populate the PR body (`references/full-cycle-mode.md`). Either way, the
finding is already prose-ready — no per-finding template needed.

`triage` is a per-finding **assessment** the reviewer assigned to the
finding ("is this a real bug?"). The `clawpatch triage` **command** is a
separate dedup-and-prioritize pass that operates on the *set* of
findings. Same word, different layers.

**Always probe the shape first** before writing parser code — the docs and
older skill versions have drifted from the real CLI more than once:
top-level envelope shape; `findingId` → `id`; evidence `file`/`line` →
`path`/`startLine`/`endLine`; and `--output` writes Markdown while JSON
comes only from `--json` on stdout (there's no `--format` flag). One safe
preflight (note the stdout redirect, not `--output`):

```bash
clawpatch report --json --no-input --no-color > /tmp/cp.json
jq 'type, keys, (.items // .findings)[0] // empty' /tmp/cp.json | head -40
```

The internal record types in `references/finding-schema.md` come from
`spec.md` and may use different field names than the wire output; treat
that reference as a sketch, not a contract.

### Enums

**`FindingCategory`** (review mode dependent — `deslopify` uses this set):

`bug | security | performance | concurrency | api-contract | data-loss |
test-gap | docs-gap | build-release | maintainability`

**Finding `status`** — see schema. New findings start `open`. After a fix
attempt that validates, status moves to `uncertain` (not `fixed`); only
`revalidate` confirms `fixed`.

**`FeatureKind`** (for `review --kind` and `fix --feature` targeting):

`cli-command | route | ui-flow | service | job | agent-tool | library |
config | release | test-suite | infra | unknown`

**`FeatureStatus`** (what `clawpatch status` reports per feature):

`pending | claimed | reviewed | needs-fix | fixing | fixed | revalidated |
skipped | error`

**`PatchAttempt.status`** (visible in patch attempt records):

`planned | applying | applied | validated | failed | abandoned`

### jq recipes

The everyday recipe — open critical/high findings as a scannable table:

```bash
jq -r '.items
   | map(select(.status=="open" and (.severity=="critical" or .severity=="high")))
   | sort_by(.severity, .confidence) | reverse
   | .[] | "\(.id)\t\(.severity)\t\(.category)\t\(.title)"' \
  /tmp/clawpatch-findings.json
```

Path-filtering, category counts, patch-attempt joins, and feature-kind
joins live in [`references/finding-schema.md`](references/finding-schema.md)
("Advanced jq recipes"). All recipes assume the `{ items: [...], findings:
<count> }` envelope — re-probe with `jq 'type, keys'` if a version differs.

## Presenting findings to the user

Do not dump the JSON. Build a compact table the user can scan in one screen,
then ask which to act on. Recommended sort: **severity desc → confidence desc
→ triage (`confirmed-bug` first) → file asc**.

**Ignore the `next:` hint in `review` / `report` output.** Clawpatch
prints something like `next: clawpatch fix --finding <first-id-it-found>`
after most commands. That hint is "first finding encountered," not "best
finding to fix" — there is no severity/confidence/triage ranking behind
it. Selection is the agent's (and user's) job: rank deliberately using
the sort above, not by whatever finding happened to land first.

A reasonable shape:

```
Found 14 open findings (3 critical, 5 high, 6 medium).

  ID           SEV       CAT          FILE:LINE                         TITLE
  <id>         critical  security     src/auth/session.ts:47            Session cookie missing HttpOnly
  <id>         critical  data-loss    src/db/migrate.ts:88              Migration drops column without backup
  <id>         high      bug          src/api/users.ts:122              Null deref when email is missing
  ...
```

The `id` format is a hash (per spec, derived from category + evidence
+ title). Don't fabricate IDs in your examples — copy them from the actual
`report --json` output.

When the user asks for evidence on a finding, surface the `evidence`,
`reasoning`, `recommendation`, and (if the finding has it) `reproduction`
fields. These are the substantive content; everything else is metadata.

Do **not** auto-pick fixes for the user. Providers do produce false
positives; aggressive fixes for them can damage the codebase. Explicit
per-user-approval is the point.

## Choose your mode: scanner-only or full-cycle?

Before acting on any finding, decide which of two modes you're in. They
are different *workflows*, not just different PR styles, and they
determine which sections of this skill apply.

**Scanner-only mode (DEFAULT for non-trivial repos).** Clawpatch's only
job is to produce the findings. The host agent dispatches its own
subagents — one per finding — in isolated worktrees branched from
`main`. **Subagents never invoke any `clawpatch` command**; they
receive the finding JSON inline in their prompt and use the host's
normal edit/test/commit/PR tooling. `fix`, `revalidate`, `open-pr`,
and `land` are not used in this mode at all.
→ See *Scanner-only mode: subagent dispatch* below.

**Full-cycle mode.** The main thread runs `clawpatch fix --finding <id>`
against its `.clawpatch/` state. Clawpatch's provider produces the
patch; Clawpatch's format → typecheck → lint → test validators gate
acceptance; commits and PRs happen via either `clawpatch open-pr` or
the agent's own PR workflow.
→ See *Full-cycle mode* below, then read
`references/full-cycle-mode.md` in full before acting.

**Default to scanner-only mode when *any* of these hold** (most non-
trivial repos hit at least one):

- **2+ fixes need to run in parallel.** `clawpatch fix` does not
  parallelize safely on shared `.clawpatch/` state — feature locks
  serialize it.
- The repo has **strict commit/PR conventions** Clawpatch can't honor
  natively: conventional-commit scopes with a closed allowlist,
  maintainer-vs-community PR-body splits, Mergify-style queue labels,
  required AGENTS.md / CONTRIBUTING.md fields. `clawpatch open-pr`
  writes a generic body that needs rewriting anyway.
- **Explicit control over the patch shape is wanted** — briefing the
  fix agent with specific test setup pointers, helper signatures,
  "don't expand scope" constraints. `clawpatch fix` ships the finding
  to its provider with an opaque built-in prompt that can't be shaped.
- The **host agent has a stronger code-editing model** than the
  configured Clawpatch provider can call.

Full-cycle mode is right when **all** of these hold:

- Single-developer, sequential flow.
- The repo's commit/PR conventions are minimal — Clawpatch's auto-
  generated body is acceptable.
- The configured Clawpatch provider is trusted to produce the patch.
- Clawpatch's validator gate should pre-approve the patch before a
  human reviews it.

This decision is upstream of everything else in this skill. In
scanner-only mode, skip past the *Full-cycle mode* sections — they
describe machinery that mode never uses.

## Scanner-only mode: subagent dispatch

In this mode, Clawpatch is a high-quality reviewer and nothing else.
The findings JSON is the only output that matters. The main thread
never invokes `clawpatch fix`, `revalidate`, `open-pr`, or `land`.

**This is the mode when the user asks for parallel fixes.** "Fix these
findings in parallel," "dispatch subagents to fix each one," "fan these
out" — all route here. `clawpatch fix` doesn't parallelize safely on
shared `.clawpatch/` state, and the workarounds (copying state,
sharing state) are choreography against undocumented assumptions (see
*Safety rules*). Don't refuse the parallel request; do it the right
way — extract findings as JSON, hand them to subagents inline, and
treat the parent's `.clawpatch/` as disposable.

**How findings reach the subagents.** Subagents do not share the main
thread's `.clawpatch/` state and do not need Clawpatch installed in
their worktree at all. The main thread extracts each finding as JSON
and pastes it inline into the subagent's prompt:

```bash
# Main thread, in the research worktree:
clawpatch report --json --no-input --no-color > /tmp/cp.json

# Per finding selected for fan-out:
jq --arg id "<id>" '.items[] | select(.id == $id)' /tmp/cp.json \
  > /tmp/finding-<id>.json
```

The subagent's prompt then carries the finding inline. A minimal shape:

```
You are addressing a Clawpatch finding in this worktree.

FIRST, verify the finding is real. Clawpatch's provider produces false
positives — read the evidence at the cited path/lines and confirm the
bug actually exists before changing anything. If it's a false positive
or the recommendation would make things worse, STOP and report that
back instead of forcing a fix.

If it's real: implement the fix and prepare a commit + PR using this
repo's conventions (see AGENTS.md / CONTRIBUTING.md). Use the finding's
title, recommendation, reproduction, and evidence for the commit message
and PR body, and add the suggestedRegressionTest. The minimumFixScope
field lists the files you may touch — do not modify anything else, and
do not expand the fix beyond what the finding describes.

<paste contents of /tmp/finding-<id>.json>
```

The JSON is already structured and prose-ready — there's no need for a
per-finding markdown template. Most of the dispatch prompt's bulk is
**repo-specific glue** (commit conventions, PR template, test framework,
branch-naming rules), which lives in the host agent's normal context
(AGENTS.md, CLAUDE.md, repo files), not in the Clawpatch handoff. The
verify-first and stay-in-scope instructions are the load-bearing part —
they steer the subagent away from Clawpatch's two main fix pitfalls
(acting on a false positive, over-correcting beyond the finding).

### Flow (main thread)

1. `init → map → review → report --json` in a research worktree.
2. `jq '.items[]' /tmp/cp.json` — inspect findings; present a table
   (see *Presenting findings*).
3. Select which to fan out. **Add file-independence to the selection
   sort:** prefer findings whose `evidence[].path` and `minimumFixScope`
   don't overlap, so subagents can't conflict.
4. For each selected finding, extract its JSON (jq above) and spawn a
   subagent in a fresh worktree **branched from `main`** (not from the
   research worktree). Pass the JSON inline in the prompt.
5. Each subagent runs the host's normal edit / test / commit / push /
   PR flow. **It never calls `clawpatch`.**
6. The research worktree is disposable — discard it after findings are
   extracted.

### Sub-rules

- **Subagents do not share `.clawpatch/` with the main thread.**
  Passing the finding JSON inline is the entire handoff. Sharing state
  defeats the parallel-safety property of this mode.
- **Subagent worktrees branch from `main`, not from the research
  worktree.** The research worktree may carry experimental state;
  subagents start clean.
- **Selection is the agent's job, not Clawpatch's.** Rank by
  `severity desc → confidence desc → triage (confirmed-bug first) →
  file-overlap (independent first)`. Ignore the `next:` hint in
  Clawpatch's output.

### How this compares to full-cycle mode

| Concern                            | Scanner-only mode                    | Full-cycle mode                          |
| ---------------------------------- | ------------------------------------ | ---------------------------------------- |
| Concurrency                        | Parallel (independent worktrees)     | Sequential (`.clawpatch/` locks)         |
| Patch model                        | Host agent's model                   | Clawpatch provider's model               |
| Patch shaping / constraints        | Explicit subagent briefing           | Opaque Clawpatch prompt                  |
| Repo commit/PR conventions         | Subagent honors via host context     | Clawpatch generates generic body         |
| Validator gate                     | Subagent runs its own                | Clawpatch runs format/typecheck/lint/test |
| Finding traceability               | Finding ID in PR body / trailer      | Same                                     |

Full-cycle mode is faster for trivial fixes when its provider produces
the right patch first try. Scanner-only mode is more reliable
everywhere else, and is the only mode that scales to parallel work.

### After subagents finish: don't reconcile, discard

Subagents fix code without invoking `clawpatch`, so the parent's
`.clawpatch/` never learns that anything was fixed — findings stay
`status: "open"`, no `PatchAttempt` records get written. **This is a
non-issue, because the state is disposable.** Don't try to reconcile it:

- There is no documented "mark this finding fixed from outside"
  command. Hand-editing `findings/*.json` is forbidden (*Safety rules*).
- The audit trail of what shipped lives in git — PR bodies, commit
  trailers (`Clawpatch finding: <id>`), the issue tracker — not in
  `.clawpatch/`.
- After the subagent PRs are out, run the cleanup one-liner from the
  *Lifecycle* section (`[ -d .clawpatch ] && rm -r .clawpatch`). The
  next review cycle rebuilds state fresh.

The one exception worth offering: if the user explicitly wants an
independent "did the subagents actually fix it?" check, after their PRs
merge you can pull `main` and run `clawpatch revalidate --finding <id>`
per dispatched finding. Each is a fresh provider pass against the
now-fixed code; findings that still reproduce flip back to your
attention. This costs one provider call per finding and is purely
optional — it's a verification pass, not a state-bookkeeping
requirement.

## Full-cycle mode (cherry-pick → fix → PR handoff → batch)

Full-cycle mode runs `clawpatch fix` in the main thread: Clawpatch's
provider produces the patch, its validators gate it, and commits/PRs go
through `open-pr` or the agent's own workflow. It is the **non-default**
path (scanner-only is the default — see *Choose your mode*).

> **STOP — required reading.** Before running `clawpatch fix`,
> `revalidate`, `open-pr`, or any batch in this mode, **read
> [`references/full-cycle-mode.md`](references/full-cycle-mode.md) in
> full.** Do not act on the capsule summary below. That reference holds
> the mandatory pieces this summary omits: the clean-worktree
> precondition, the `fix` → `uncertain` → `revalidate` → `fixed`
> sequence, exit-6 worktree recovery, the Pattern A vs B PR-handoff
> decision and handoff-field table, the between-fix worktree-state
> contract, batch failure modes, and the one-fix-per-`.clawpatch/`
> concurrency rule. Skipping it skips safety steps. The summary exists
> only to confirm this is the mode you want — not to operate from.

Capsule summary (orientation only — not sufficient to act):

- **One fix:** clean worktree, then `clawpatch fix --finding <id>`. On
  success the finding goes to `uncertain`, **not** `fixed`; confirm with
  `clawpatch revalidate --finding <id>`.
- **PR:** either `clawpatch open-pr --patch <id> --draft` (Clawpatch
  writes the body) or hand the worktree to the agent's own commit/PR
  workflow. The full-cycle reference has the decision criteria and the
  finding-field → PR-body mapping.
- **Batch:** native (`fix --finding A --finding B`, or
  `fix --feature X --severity high`) when it works; otherwise a
  sequential loop with per-fix gates. Both, plus the worktree-state
  contract and failure-mode handling, are in the reference.
- **Never** run `clawpatch fix` in parallel on one `.clawpatch/`
  (feature locks serialize it; interleaved patches don't merge). For
  parallel work, use scanner-only mode instead.
- **Stop on the first surprise** — exit 6, an oversized diff (>300
  lines), or two consecutive `wont-fix` outcomes. Halt and report.

## Exit codes

Per spec:

| Code | Meaning                          | Agent action                                                                                  |
| ---: | -------------------------------- | --------------------------------------------------------------------------------------------- |
|    0 | Success                          | Continue.                                                                                     |
|    1 | Runtime failure                  | Show stderr to user; do not retry without diagnosing.                                         |
|    2 | Invalid usage or config          | The command was constructed wrong, or `config.json` is invalid. Re-read with `--help`.        |
|    3 | Dirty worktree blocks operation  | Ask user to commit/stash. **Never auto-pass `--force` to bypass.**                            |
|    4 | Provider auth/config failure     | Tell user to authenticate the configured provider. Stop the run.                              |
|    5 | Provider quota / rate-limit      | Sleep + retry once with backoff. After two failures, escalate and stop.                       |
|    6 | Tests / validation failed        | Worktree is dirty with the rejected patch. Run `git restore .`, show output, ask user.        |
|    7 | Lock conflict                    | Check `pgrep -f clawpatch`; if nothing running, `clawpatch clean-locks --yes`. Otherwise wait. |
|    8 | Malformed provider output        | Retry once. If it recurs, switch model with `--model` or escalate.                            |

## Safety rules

Non-negotiable.

1. **Never pass `--force` to `fix` or `open-pr` without explicit user
   approval.** `--force` overrides the dirty-worktree and failed-validation
   guards that are Clawpatch's whole point.
2. **Never run `clawpatch land` autonomously.** `land` merges the
   Clawpatch PR; it is destructive and irreversible. Only the user
   invokes it.
3. **Do not hand-edit `.clawpatch/*.json`.** State files are versioned and
   internal. Use Clawpatch commands (`triage`, `revalidate`, `fix`) to
   mutate state.
4. **Gitignore the whole `.clawpatch/` directory** (one line:
   `.clawpatch/`). See the *state directory* section above.
5. **Never combine `clawpatch fix` with subagent dispatch.** The two
   workflows are mutually exclusive:
   - If you're using subagents (parallel or otherwise), the main thread
     extracts each finding as JSON and the subagent does its own fix
     without invoking any `clawpatch` command. See *Scanner-only mode*.
   - If you're using `clawpatch fix`, run it sequentially in the main
     thread without subagents. See *Full-cycle mode*.

   Don't combine them. Sharing `.clawpatch/` across subagent processes
   risks silent wrong-worktree patch application (evidence paths
   resolve against `project.json.rootPath`, which was written for the
   main worktree). Copying state per subagent works mechanically but is
   choreography against undocumented assumptions. The clean answer is
   to pick one mode and stay in it.
6. **Confirm the target repo before any PR-creating call** (Clawpatch's
   `open-pr`, the agent's PR-creation skill, or a raw shell PR command).
   Clawpatch honors the current git remote; in a fork or worktree pointing
   at the wrong upstream, the PR opens in the wrong place.
7. **Pause the batch on the first surprise.** Validation failures,
   oversized diffs, consecutive `wont-fix` outcomes — all are signals the
   provider is misbehaving.

## Anti-patterns

- **Dumping `report --json` raw.** Build a table.
- **Reflexively escalating from smoke test to full sweep.** Three
  features routinely yields enough findings for hours of fix work. The
  default after a smoke test is "review findings → ship," not "run the
  full repo next." Surface the time/cost estimate (~30–60s per feature)
  to the user before any full-repo `review`.
- **Following the `next:` hint Clawpatch prints.** It's first-finding-
  encountered, not best-finding-to-fix. Use severity/confidence/triage
  to rank — see *Presenting findings*.
- **Running `review` without `--limit` on a first-time repo.** A
  500-feature repo burns hours of provider time. Smoke-test with
  `--limit 3` first; get user approval before going full-sweep.
- **Treating `uncertain` as failure.** A fix returning `uncertain` is the
  *expected* post-fix state; run `revalidate` to confirm `fixed`.
- **Re-running `init` reflexively.** `init` overwrites detection. If
  `.clawpatch/` exists, only re-run with `--force` after the user asks.
- **Using Clawpatch as a small-diff code-review tool.** The
  mapping/review/findings machinery is designed for whole-repo work; for
  a 20-line PR, use the harness's diff-review helper instead.
- **Fabricating finding IDs in output to the user.** They're hashes — copy
  from `report --json`, don't invent.
- **Skipping `--no-input` / `--no-color` and then complaining about
  parsing problems.** The two universal flags are not optional (the
  context-dependent ones — `--json`/`--quiet`/`--root`/`--yes` — are).

## Harness-portability notes

This skill is written to be useful across agent harnesses. The bash
recipes above run anywhere with a shell. A few notes for non-Claude-Code
hosts:

- The frontmatter (`name`, `description`) at the top is Claude Code's
  skill format. Other harnesses can ignore it; the markdown body is the
  portable content.
- `references/` holds two files. `finding-schema.md` is a supplementary
  deep dive (full record types, provider matrix, advanced jq) — load it
  when needed. `full-cycle-mode.md` is **required reading before any
  full-cycle `clawpatch fix`/`open-pr`/batch operation**, not optional —
  the inline full-cycle section is only a capsule pointer.
- Pattern B (PR handoff) deliberately does not prescribe how to make the
  commit or PR. The agent uses whatever commit-and-PR mechanism it
  already has — a higher-level skill, a project-specific command, or
  raw shell tools talking to whichever VCS host the repo lives on.
  This skill's job is to specify *what* finding context to hand off,
  not *how* to ship it.
- Where this skill says "ask the user", treat that as a harness-native
  prompt — chat message, terminal prompt, whatever your runtime provides.
- **Parallel scanner-only dispatch and language-server noise.** When the
  host harness dispatches subagents into sibling worktrees of the same
  repo, a long-running LSP in the parent worktree may attempt to lint
  files across all of them — Go's `gopls` is a known offender, producing
  spurious "internal package not allowed" and "undefined: X" diagnostics
  for files that aren't actually in scope. Treat the **subagent's own
  `go build` / `go test` / equivalent** as authoritative; ignore parent-
  worktree LSP diagnostics about sibling-worktree files during parallel
  dispatch.
