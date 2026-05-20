# Full-cycle mode reference

This is the complete reference for **full-cycle mode** — where the main
thread runs `clawpatch fix` against its own `.clawpatch/` state, lets
Clawpatch's provider produce the patch, and gates it through Clawpatch's
format → typecheck → lint → test validators.

**Read this file in full before running any `clawpatch fix`,
`revalidate`, `open-pr`, or batch operation.** The summary in `SKILL.md`
is only a pointer; the worktree-state contract, validation-failure
handling, PR-handoff decision, and concurrency rules below are mandatory
for safe operation and are not repeated inline. Acting on the SKILL.md
summary alone will skip required safety steps.

Full-cycle mode is the **non-default** path. Confirm it's the right mode
first: it's correct only when the work is single-developer / sequential,
the repo's commit/PR conventions are minimal, the configured provider is
trusted to produce the patch, and Clawpatch's validator gate is wanted.
If any of parallel work, strict repo PR conventions, explicit patch-shape
control, or a stronger host-agent model apply, use scanner-only mode
instead (see `SKILL.md`, *Choose your mode*).

## Cherry-picking a single fix

The fix loop assumes a clean worktree by default. Confirm before invoking:

```bash
git status --porcelain   # must be empty (apart from .clawpatch/ contents)
clawpatch fix --finding <id> --no-input --no-color --yes
```

What happens: Clawpatch reads the finding, asks the provider for a patch
plan, applies it to the worktree, runs the configured validators in
order — `format` → `typecheck` → `lint` → `test` — and records the result
as a `PatchAttempt` linked to the finding.

**On success the finding moves to `uncertain`, not `fixed`.** Validation
passed, but a second confirmation pass is needed:

```bash
clawpatch revalidate --finding <id> --no-input --no-color
# or, equivalently, by patch / by feature:
clawpatch revalidate --patch <patchAttemptId> --no-input --no-color
clawpatch revalidate --feature <featureId>   --no-input --no-color
```

If `fix` exits non-zero, consult the *Exit codes* table in `SKILL.md` —
do not blindly retry. Exit 6 (validation failed) in particular means the
patch landed but tests/lint/typecheck rejected it; the worktree is now
dirty and needs a `git restore .` (or a manual fix) before the next
attempt.

`fix --dry-run` shows what would happen without applying anything. Use it
when the user is undecided.

## PR handoff (Pattern A or B)

After `fix` succeeds the worktree contains uncommitted changes. Pick one
of two paths.

### Pattern A: Clawpatch-native `open-pr`

Use when **all** of these are true:

- The user wants the finding metadata (evidence, reasoning, validation log)
  preserved verbatim in the PR body — `open-pr` generates a richer body than
  a hand-written commit message.
- The repo uses GitHub and `gh` CLI is installed + authenticated.
- One finding per PR is acceptable.

```bash
clawpatch open-pr --patch <patchAttemptId> --draft --base main --no-input
```

`--draft` is the default agent posture; promote to ready-for-review only
after the user reads the diff and PR body. `--base <branch>` targets a
different base if `main` isn't right. `open-pr` commits only the files
recorded in the patch attempt (per `PatchAttempt.filesChanged`), not
unrelated worktree changes, and creates a branch named for the finding.

`open-pr` refuses to open a PR for a failed-validation patch unless
`--force` is passed — **never pass `--force` without explicit user
approval**.

### Pattern B: Hand off to the agent's own commit/PR workflow

Use when **any** of these are true:

- The fix is part of a bundled PR (Clawpatch `open-pr` is one patch per PR).
- The repo's PR workflow needs steps Clawpatch doesn't do — issue claim,
  PR template, queue labels, conventional-commit prefix, etc.
- Not a GitHub repo (GitLab, Bitbucket, Gerrit, codeberg, self-hosted git, …).
- The agent has a preferred commit/PR skill, command, or tool of its own.

Clawpatch's job ends when the worktree contains validated changes and the
patch attempt is recorded. From there, hand off the work to whatever
commit-and-PR mechanism the agent already has — a higher-level skill, a
project-specific PR command, the harness's `git` and VCS-host integration,
or raw shell tools. **This skill does not prescribe how the commit and PR
get made; it only specifies what to hand off.**

The handoff content the next workflow needs from the finding:

| From the finding                  | Where it usually goes                          |
| --------------------------------- | ---------------------------------------------- |
| `title`                           | Commit subject / PR title                      |
| `recommendation`                  | PR body — the *what to do*                     |
| `reasoning`                       | PR body — the *why it's broken*                |
| `reproduction`                    | PR body — the *how to observe*                 |
| `evidence[].path`/`startLine`/`endLine` | PR body — the *where*                    |
| `whyTestsDoNotAlreadyCoverThis`   | PR body — the test-gap analysis                |
| `suggestedRegressionTest`         | New test the agent should add                  |
| `minimumFixScope`                 | Files the fix must stay within                 |
| `category`, `severity`, `triage`  | Labels or PR-body context                      |
| `id`                              | Commit trailer / PR body, for traceability     |

When composing the commit / PR body, lean on the finding's pre-structured
content — `title`, `recommendation`, `reasoning`, `reproduction`,
`evidence`, `minimumFixScope`. They are already prose-ready; there's no
need for a per-finding markdown template. (For subagent dispatch where
the subagent does the formatting itself, pass the raw JSON — see
*Scanner-only mode* in `SKILL.md` for the paste-the-JSON dispatch pattern.)

Decision rule when ambiguous: ask the user. "Pattern A keeps Clawpatch's
full finding context in the PR body and ships one PR per finding;
Pattern B is the repo's normal PR flow and can bundle multiple
findings. Which one?"

## Batch fix orchestration

`clawpatch fix` supports native batching (verify against
`clawpatch fix --help` — the surface has shifted between releases):

```bash
# Multiple --finding flags in one fix run:
clawpatch fix --finding <id1> --finding <id2> --finding <id3> --no-input --no-color --yes
# All findings on a feature, filtered by severity:
clawpatch fix --feature <featureId> --severity high --no-input --no-color --yes
```

When native batching is available, prefer it — Clawpatch handles
validator sequencing and locks internally and produces fewer but larger
patch attempts.

When any of the following are needed, drive a sequential loop instead:

- A separate commit per finding so reviewers can see one fix per commit.
- One PR per finding instead of one bundled PR.
- A user-approval gate between fixes (the most common reason).
- Selective revalidation after each fix, with the option to halt the
  batch if any single fix produces a surprising diff.

**If a native batched `fix` invocation fails with exit 2** (invalid
usage), the installed Clawpatch version may not support the multi- or
by-feature batching shape. Fall back to a sequential loop.

### Sequential loop variant A: one PR per finding

Each finding gets its own branch and PR. Best for findings touching
unrelated subsystems, repos that prefer small PRs, and using Clawpatch
`open-pr` (which preserves full finding context in the PR body).

```bash
BASE=main
FINDINGS=(<id1> <id2> <id3>)   # ordered, user-provided

git fetch origin "$BASE"

for id in "${FINDINGS[@]}"; do
  git checkout "$BASE" && git pull --ff-only
  git checkout -b "clawpatch/$id"

  if ! clawpatch fix --finding "$id" --no-input --no-color --yes; then
    echo "fix failed for $id (exit $?) — stopping batch"
    break
  fi

  # Show the user the diff. Wait for approval. Then:
  clawpatch open-pr --patch "$id" --draft --base "$BASE" --no-input
done
```

Do **not** run this loop unattended. The whole point of using Clawpatch
is human-in-the-loop control; an unattended loop defeats it. Pause after
each iteration, surface the diff and PR URL, and wait for the user.

### Sequential loop variant B: bundled PR

All findings land on one feature branch. Best for thematically tied
findings ("fix all the null-deref bugs"), findings that touch the same
files, or repos where each new PR has real overhead (template, queue
labels, CI cost).

```bash
BASE=main
BRANCH="clawpatch/batch-$(date +%Y%m%d-%H%M)"
FINDINGS=(<id1> <id2> <id3>)

git checkout "$BASE" && git pull --ff-only
git checkout -b "$BRANCH"

for id in "${FINDINGS[@]}"; do
  if ! clawpatch fix --finding "$id" --no-input --no-color --yes; then
    echo "fix failed for $id (exit $?)"
    git restore .                  # discard partial patch
    break
  fi

  git add -A
  TITLE=$(jq -r --arg id "$id" \
    '.items[] | select(.id == $id) | .title' \
    /tmp/clawpatch-findings.json)
  git commit -m "fix: $TITLE

Clawpatch finding: $id"
done

# After the loop, hand off the branch (with its per-finding commits) to
# the agent's commit-and-PR workflow. This skill stops at the branch
# boundary — pushing and opening the PR is the harness's responsibility,
# using whatever mechanism it has for the repo's VCS host.
```

## Worktree-state contract

Between any two sequential `clawpatch fix` invocations the agent MUST
guarantee:

1. `git status --porcelain` is empty apart from `.clawpatch/` writes the
   tool itself made.
2. The current branch is the one the next fix should land on.
3. No stale `clawpatch` processes are holding feature locks.

If all three cannot be guaranteed, stop and surface the state instead of
plowing through.

## Failure modes during a batch

Handle these explicitly rather than blindly retrying.

### Exit 6 (validation failed) mid-batch

The worktree is dirty with a rejected patch. Run `git restore .` to clean
it, then show the validation output to the user and ask: skip this finding
and continue, or abort the batch? Do not retry the same finding silently —
the same validator will reject the same patch.

### `wont-fix` outcome after the fix attempt

Clawpatch records this in the finding's history. The worktree should
already be clean. The provider is signaling that the finding's
recommendation doesn't translate into a safe patch. **Two consecutive
`wont-fix` outcomes** = stop the batch and tell the user. The provider is
rejecting the chosen findings; pushing through wastes time.

### Unexpectedly large diff (>~300 lines)

The provider may be over-correcting. Show the user the diff and ask
before continuing. Common causes: the provider mistook a small fix for a
broader refactor, or ignored the finding's `minimumFixScope`.

### Lock conflict (exit 7) mid-batch

Most likely a stray `clawpatch` process. Check `pgrep -f clawpatch`. If
nothing is running, `clawpatch clean-locks --yes` clears the locks
directory. If something *is* running, wait — do not kill another agent's
in-flight run.

## Concurrency

Do not parallelize `clawpatch fix` on the same `.clawpatch/` state
directory. Feature locks make it safe-but-noisy at best; in the worst
case two patches racing on the same file produce a result with no useful
merge resolution. One fix at a time per repo. (For parallel fixes, use
scanner-only mode — see `SKILL.md`.) Separate repos with separate
`.clawpatch/` directories are independent and safe to run concurrently.

## Choosing a pattern with the user

When asking which pattern to use, frame it as a tradeoff:

> N findings selected. Three options:
>
> - **Native batched fix** — single `clawpatch fix` invocation handling
>   all findings, one patch attempt, one diff to review. Fastest when
>   the findings are uncontroversial.
> - **Sequential, one PR per finding** — each finding lands as its own
>   PR with Clawpatch's full finding context in the body. Best when
>   reviewers will merge them independently.
> - **Sequential, bundled PR** — one branch, one PR carrying all
>   commits, with a hand-written body summarizing the theme. Best when
>   findings are related or PR overhead is real.
>
> Which one?

Do not pick for them. Repo conventions (PR template, queue labels,
review culture) are not visible from inside Clawpatch and dominate the
right choice.
