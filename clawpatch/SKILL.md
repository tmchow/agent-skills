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
  those belong to a different tool.
version: 0.1.4
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
        description: Override the default review/fix provider (codex). One of codex, claude, cursor, grok, opencode, pi, acpx, mock.
      - name: CLAWPATCH_MODEL
        required: false
        description: Provider-specific model string (e.g. opencode/big-pickle, anthropic/claude-sonnet-4).
      - name: CLAWPATCH_REASONING_EFFORT
        required: false
        description: Provider reasoning level (none/minimal/low/medium/high/xhigh).
      - name: CLAWPATCH_RPM
        required: false
        description: Cap provider calls started per 60-second window.
  hermes:
    tags: [code-review, clawpatch, cli, bug-fixing, automation]
    category: software-development
    requires_toolsets: [terminal]
---

# Clawpatch — agent workflow

> Clawpatch (`openclaw/clawpatch`, https://clawpatch.ai) is an agent-driven
> CLI: it reviews a repo into structured findings and can apply validated
> per-finding fixes. It needs Node.js 22+ and a provider CLI (codex by
> default); `fix` refuses dirty source worktrees by default, while
> `review`/`ci`/`revalidate` can include dirty code when explicitly scoped.
>
> **This skill is judgment + orchestration, not a CLI reference.** For exact
> flags, JSON fields, and exit codes, trust `clawpatch <cmd> --help` and the
> live `--json` output over anything written here — Clawpatch is pre-1.0 and
> its surface drifts, so a written copy will mislead. What this skill carries
> is what `--help` can't tell you: when to reach for it, how to fix findings
> in parallel, and the traps.

## When to use

Use Clawpatch when the user names it, or wants whole-repo automated review
that yields locatable findings plus optional validated fixes.

Don't use it for: a small diff already under review (a whole-repo review cycle
is overkill — use a diff-review tool); a bug the user already understands and
just wants written; or when no provider CLI (`codex`/`claude`/`cursor`/`grok`/
`opencode`/`pi`/`acpx`, or `mock` for tests) is installed — `clawpatch doctor`
fails fast, so say so and stop.

## Setup

`clawpatch doctor` verifies the install and the provider. The published CLI
requires Node.js 22+. If `clawpatch` is missing, install the current published
package (`npm install -g clawpatch@0.7.2`). The provider (codex by default) is
the user's to install and authenticate — don't run login flows on their behalf.

Gitignore `.clawpatch/` early to keep status, commits, and PR diffs clean.
Clawpatch allows its own state directory to change during runs; `fix`'s dirty
guard is for source worktree changes, not `.clawpatch/` churn. Add the single
line `.clawpatch/` before review/fix when the repo doesn't already ignore it,
but remember `.gitignore` is tracked — confirm with the user before editing it.

## Review → findings

The pipeline is `init → map → review → report`. `init` and heuristic `map` are
local and cheap; `review` is the expensive step (~30–60s per feature with
codex) and runs with a bounded worker pool by default.

For a first whole-repo pass, `clawpatch ci` runs that pipeline in one call and
writes the same `.clawpatch/` findings the staged steps would — so
"`ci` → pick a fix path" is a fine default. Drop to the staged commands only
when you need to **sample before sweeping** a large or paid provider
(`review --limit 3`, below), **scope to a slice** (`review --since <ref>`), or
**resume an existing review** as code moves (`revalidate` / `review --since`,
below). Either way `ci` **stops at findings — it never fixes or opens PRs**;
fixing stays the paths under "Choose how to fix."

Run `init && map` directly — they're **idempotent and non-destructive**, so an
existing `.clawpatch/` just gets refreshed (`init` no-ops without `--force`;
heuristic `map` re-classifies). No scan, no `--force`. `map` writes one JSON
file per **feature** — the subsystem-sized unit (a package, a command, a
service) that both review and scoping operate on — into `.clawpatch/features/`.
If deterministic mapping is too shallow, `clawpatch map --source auto` can add
read-only provider-assisted slices; use `--source agent` only when you
deliberately want provider mapping even if heuristics found enough.

State persists in `.clawpatch/` and there's no natural point where the agent
deletes it, so findings — and their `fixed`/`wont-fix`/`uncertain` statuses —
carry across runs. **That persistence is intentional: Clawpatch owns resume
and freshness, so use its commands instead of hand-managing or wiping
findings.** When code has moved since the last review:

- `clawpatch revalidate --since <ref>` (or `--all`) re-checks existing
  findings against the current code and updates their statuses — this is the
  CLI's de-stale mechanism.
- `clawpatch review --since <ref>` reviews only the features changed since
  `<ref>`, including changed features regardless of previous review status,
  without a full re-review.

Prefer those over trusting stale statuses *or* wiping. Wiping
(`[ -d .clawpatch ] && rm -r .clawpatch` — guarded, never `rm -rf`) is a last
resort for genuinely corrupt state, not the freshness tool — it discards
resume context and costs a full re-review. (There's no `--resume` flag;
"resume" just means the on-disk state is still there.)
Use `clawpatch triage --finding <id> --status <status> --note <text>` to mark
false positives or decisions; it preserves the finding and appends history
instead of rewriting JSON by hand.

On a large or unfamiliar repo, **smoke-test first** (`review --limit 3`) and
treat that as often-sufficient — surface the time/cost before committing to a
full sweep (`ci` or `review`). For cost control on broader runs, `--jobs`
defaults to roughly half the CPU cores (max 10); lower it or use
`--rate-limit-per-minute` / `CLAWPATCH_RPM` when provider spend or quotas
matter.

By default review sees only **committed** code; `--include-dirty` (on `review`,
`ci`, and `revalidate`) pulls uncommitted worktree changes into scope — reach
for it to review in-progress work before it's committed. It's a review-scope
switch, not a `fix` dirty-tree override (Safety).

Use `clawpatch review --mode deslopify` when the user wants slop,
maintainability, or performance cleanup. Keep default review mode for ordinary
bug, security, API-contract, data-loss, and release-risk review.

### Scoping the review

The switches above scope *which code* review sees; this scopes *which features*
it spends on. `review` has no path filter — you scope it by **selecting
features**. Each `.clawpatch/features/*.json` carries an opaque `featureId` (a
hash like `feat_library_…`, **not** a `1..N` index — passing a number silently
matches nothing and the run reviews zero features) plus its `title` and
`ownedFiles[].path`. To review only what the user named ("just the Go code",
"only the auth package", "ignore the docs"), translate paths → ids yourself,
then pass them. For more than a one-off id, prefer `--feature-list`: it reads
one feature id per line, keeps first-seen order, and de-duplicates:

```
for f in .clawpatch/features/*.json; do
  jq -r '[.featureId,.title,((.ownedFiles//[])|map(.path)|join(","))]|@tsv' "$f"
done                                   # read the table, pick ids, then:
printf '%s\n' <id> <id> > /tmp/clawpatch-features.txt
clawpatch review --feature-list /tmp/clawpatch-features.txt
```

Repeated `--feature <id>` still works for short ad hoc selections. Confirm
mutual exclusions in `clawpatch review --help` before mixing feature selection
with `--since`, `--project`, or dirty-worktree scoping.

`--limit <n>` is the *cheapness* knob (first N features in map order) — right
for a smoke test, useless for targeting an area; reach for `--feature` when the
user names a subsystem. All Clawpatch state (features, findings, reports) is
plain JSON under `.clawpatch/`, so read it directly when a `--json` payload is
too large to scan or a wrapper truncates it.

Read findings from `report --json`. The full flag set is `clawpatch report
--help`; two non-obvious traps it won't flag for you:

- `--json` writes the JSON to **stdout**; `--output` writes **Markdown**.
  Capture JSON by redirecting stdout (`> findings.json`), not `--output`.
- In 0.7.2 the shape is `{ "findings": <count>, "total": <count>,
  "output": <path|null>, "items": [ … ], "results": [ … ] }`. `findings` is a
  deprecated count, not the array; read `.items[]` (`.results[]` is an alias).
  Field names still drift from prose docs (the id is `id`; evidence uses
  `path`/`startLine`/`endLine`). **Probe before parsing:** `jq 'keys,
  .items[0]' findings.json`.

Each finding already carries fix-ready prose — `title`, `reasoning`,
`reproduction`, `recommendation`, `suggestedRegressionTest`,
`minimumFixScope` — ready to drop into a fix prompt or PR body. Ignore the
`next:` hint Clawpatch prints; rank findings yourself (severity → confidence
→ triage), and never auto-fix without the user's pick (providers emit false
positives).

## Choose how to fix: scanner-only vs full-cycle

**Scanner-only (default, and the only safe parallel path).** Use Clawpatch
purely as the reviewer; the host agent fixes findings with its own tooling.
Required when the user wants fixes in parallel, the repo has strict commit/PR
conventions, you want control over each patch, or the host model beats the
provider's.

**Full-cycle.** Let `clawpatch fix` produce and validate the patch. Fine for
single-developer, sequential work in a low-ceremony repo where you trust the
provider and want its format/typecheck/lint/test gate.

The two are mutually exclusive — never run `clawpatch fix` from dispatched
subagents (see Safety).

### Scanner-only dispatch

1. `init → map → review`, then `report --json > /tmp/findings.json`, in a
   throwaway worktree.
2. Pick findings to fan out; prefer ones whose `minimumFixScope`/evidence
   paths don't overlap, so subagents can't collide.
3. Per finding: extract its JSON (`jq '.items[] | select(.id==$id)'`) and
   dispatch a subagent in a **fresh worktree branched from `main`** (not the
   review worktree). Paste the finding JSON inline in its prompt.
4. The subagent fixes with the host's normal edit/test/commit/PR tooling and
   **never calls `clawpatch`**. Tell it to confirm the finding is real before
   changing anything (false positives) and to stay within `minimumFixScope`
   (no over-correcting).
5. The review worktree and its `.clawpatch/` are disposable — discard after.
   Don't reconcile fixed-status back into `.clawpatch/`; the audit trail is
   git (PRs, commit trailers).

### Full-cycle

On a clean worktree, `clawpatch fix --finding <id>`. On success the finding
goes to `uncertain`, **not** `fixed` — confirm with `clawpatch revalidate
--finding <id>`. Then ship with `clawpatch open-pr --patch <patchAttemptId> --draft`
(Clawpatch writes a finding-rich PR body) or hand the diff to your own commit/PR
flow. Check flags with `clawpatch fix --help`. `fix` has no dirty-tree
`--force`; clean or stash source changes first, or use `--dry-run` only for a
preview. Never run two
`fix`es in parallel on one `.clawpatch/` (feature locks serialize them;
interleaved patches don't merge). Stop and report on the first surprise:
validation failure (exit 6 → `git restore .`), an oversized diff, or repeated
`wont-fix`.

## Safety & pitfalls

- **Never `clawpatch open-pr --force` without explicit user approval** — it
  overrides the failed-validation guard. `fix` has no `--force`; keep the clean
  source-worktree preflight intact. Clawpatch does **not** merge/land PRs today
  (there's no `land` command); if a merge/land command ever appears, treat it
  the same way — never run it without approval.
- **Never combine subagents with `clawpatch fix`.** Sharing `.clawpatch/`
  across subagents risks patches landing in the wrong worktree (evidence
  paths resolve against the original `rootPath`); copying state per subagent
  is fragile. Pick one mode and stay in it.
- **Don't blind-retry a non-zero exit.** Codes are typed (3 = dirty tree,
  4 = provider auth, 6 = validation failed, 7 = lock). On lock failures, try
  `clawpatch clean-locks --stale-only` first; use unfiltered `clean-locks` only
  after confirming no local or remote review process is active. `clawpatch
  --help` / the spec has the full set.
- **Treat this doc as fallible.** Verify any flag, field, or exit-code detail
  against `clawpatch <cmd> --help` and live `--json` output — they're the
  source of truth; this skill is the workflow around them.
