# agent-skills

A personal, multi-skill collection of cross-platform AI agent skills
(`SKILL.md` format), installable à la carte through runtime-specific lanes:
the generic skills CLI (`npx skills add tmchow/agent-skills --skill <name>`,
works across Claude Code/Cursor/Codex and other Agent-Skills runtimes), Hermes
(GitHub directory identifiers via `hermes skills install owner/repo/path/to/skill`
or `/skills install owner/repo/path/to/skill`; raw `SKILL.md` URLs are valid only
for single-file fallback installs), and OpenClaw (ClawHub via
`openclaw skills install <slug>`). The Hermes path segment is the exact skill
directory relative to repo root — do not add `skills/` unless the skill actually
lives under a `skills/` directory. Document lanes side by side; never present
one as the only path.

This guide is for anyone (human or agent) editing the repo. Keep it accurate
when conventions change.

## Repo layout

**Every top-level directory is a skill, identified by a `SKILL.md` inside it
— with one exception: `_assets/`.** That's the rule installers use (they
find skills by looking for `SKILL.md`), so anything that isn't a skill must
not look like one at the root.

- `<skill-name>/SKILL.md` — required. The agent-facing instructions.
- `<skill-name>/README.md` — required. The human-facing landing page (below).
- `<skill-name>/references/` — optional. Deep material loaded on demand.
- `_assets/<skill-name>/` — optional. Docs-only images a skill links by raw
  URL (calibration examples, README embeds). Two reasons it lives here, not
  in the skill dir: installers copy the entire skill directory verbatim, so
  anything inside ships to every user; and the leading `_` is an impossible
  skill-name start (`[a-z0-9-]`), so it can never be mistaken for a skill.
  Namespace by skill (`_assets/illo/...`) so it scales without cluttering
  the root.
- Root `README.md` — the catalog: one row per skill.
- Root `LICENSE` — MIT.

## Adding or editing a skill

1. Directory name == the skill's `name:` frontmatter, lowercase kebab-case
   (`[a-z0-9-]`, no leading/trailing/double hyphens, 1–64 chars).
2. Write `SKILL.md` (frontmatter + body rules below).
3. Write `README.md` (human landing page — rules below).
4. Put deep or optional material in `references/`, not in `SKILL.md`.
5. Add a one-line row to the root `README.md` skill table.
6. Validate (below) before committing.

## SKILL.md frontmatter

Required: `name`, `description`, `version`.

- `name` — matches the directory; lowercase kebab-case.
- `description` — third person ("This skill should be used when…"), ≤1024
  characters, with **specific** trigger phrases. Make it tool/task-specific,
  not generic: a skill wrapping tool X must not trigger on bare "do
  X-category work" when X isn't named. Add an explicit "Do NOT use for …"
  clause when the skill could be confused with a broader category.
- `version` — semver; bump on meaningful change.

Per-runtime metadata is **optional and additive**. Unknown frontmatter fields
are ignored by runtimes that don't understand them, so these blocks are safe
to include side by side:

- `metadata.openclaw` — install directive, env vars, homepage, emoji.
- `metadata.hermes` — tags, category, required toolsets.

`metadata.hermes.category` must come from Hermes's standard set:

```text
apple, autonomous-ai-agents, browser, creative, data-science, devops, email,
gaming, github, mcp, media, mlops, note-taking, productivity, red-teaming,
research, security, smart-home, social-media, software-development, writing
```

Pick the closest existing category (e.g. an illustration skill → `creative`,
a code-review skill → `software-development`). Inventing a new category should
be rare and deliberate — only when the skill truly fits none of these.

**Verify every runtime-specific field against that runtime's own docs before
adding it. Do not fabricate frontmatter schemas** — a wrong field can break
the skill silently in that runtime. The same goes for any CLI/command syntax
quoted in a skill: confirm it against the tool's `--help`, don't guess.

## SKILL.md body

- Imperative/infinitive voice ("Run X", "Confirm Y"), not second person.
- Progressive disclosure: keep the body focused; move long schemas, advanced
  patterns, and edge cases to `references/`.
- **No duplication across files.** A fact lives in `SKILL.md` *or* a
  reference, never both — duplicated content drifts.
- When a reference is mandatory before acting, gate it explicitly in the body
  (a capsule summary + a "read `references/X.md` in full before …" stop).

## CLI-wrapper skills

For skills that wrap an external CLI/API tool, `SKILL.md` is not the man page.
Use it for agent operating judgment that live `--help` will not provide:

- when to choose the tool and when not to
- the mental model needed to avoid misuse
- where the actual underlying tool is installed from, including package name and source repo when relevant
- install/verify commands for the underlying tool
- a small set of canonical examples, not an exhaustive command list
- failure handling and verification discipline
- privacy, security, and side-effect boundaries

Avoid full command tables, flag catalogs, exit-code catalogs, or examples likely
to drift. Stable, high-value commands and flags *do* belong in the skill when
they are key functionality or encode best-practice scenarios the agent would
otherwise miss. Favor scenario-shaped guidance over exhaustive reference docs:
"For X situation, run/consider Y, then verify Z." Tell agents to run the live
tool help (`<tool> --help`, subcommand help, or upstream docs) for exact syntax
around anything uncommon or version-sensitive. If quoting syntax anyway, verify
it against current help during the PR.

Use placeholders that agents cannot accidentally copy as stale literals. For
example, prefer `@<uid>` plus "copy refs exactly as printed" over fake browser
refs that look real but may be invalid.

## Per-skill README.md

The human-facing landing page. GitHub renders it when someone browses the
skill directory, and it's what a person reads to decide whether to install.
It is **not** the agent instructions — that's `SKILL.md`. Include:

- One-paragraph what-it-is, in human framing.
- **Prerequisites** — external tools, accounts, or credentials the skill
  needs. This is the highest-value section; `SKILL.md` buries it.
- Install commands for *this* skill: the generic skills-CLI one-liner
  (`npx skills add tmchow/agent-skills --skill <name>`) first, then Hermes
  CLI/slash-command install and OpenClaw ClawHub install. For Hermes docs in
  this repo, prefer the GitHub directory identifier (`owner/repo/path/to/skill`)
  over a raw `SKILL.md` URL so multi-file skills install correctly. Use raw
  `SKILL.md` URLs only as a single-file fallback. If the ClawHub slug is not
  known yet, mark it provisional and update it after publish before merge.
- A few capability bullets and a link to the upstream tool/API.
- An explicit line: "SKILL.md is the agent-facing instructions; you don't
  need to read it to use the skill."

**Do not** restate the workflow, schema, exit codes, or step-by-step
procedure from `SKILL.md` — that duplicates the agent doc and drifts. Keep the
README to slow-changing metadata (purpose, prerequisites, install).

## Scanner-safe skills (security and size budgets)

Skills here install from a community source, so security scanners judge
them at their most hostile reading — Hermes's `skills_guard` hard-blocks an
install (no `--force` override) on patterns that merely *look* like
exfiltration. These rules come from getting `illo` from a DANGEROUS verdict
to SAFE; build new skills to them from the start:

- **Never read secrets from the environment.** A community skill that reads
  a secret-shaped env var (`*_API_KEY`, `*_TOKEN`, …) scans as a critical
  exfiltration primitive regardless of what the code does with the value.
  Hermes's `required_environment_variables` frontmatter (Secure Setup on
  Load) does **not** exempt the read: as of June 2026 its scanner flags the
  `os.environ.get` even when the variable is declared (tested — verdict
  DANGEROUS, install blocked). This is deliberate, not a scanner bug: the
  environment is a shared namespace, so a community skill reading a
  secret-shaped var can harvest a key the user set for *other* tools
  (Secure Setup only prompts for missing vars — pre-existing values flow
  with no consent), and a declaration would just be consent-washing.
  Secure Setup is therefore de facto reserved for Hermes's trusted tier
  (openai/anthropics/huggingface/NVIDIA skills). Revisit only if Hermes
  adds per-skill scoped secret provisioning that isn't the shared env. When a scanner flags a pattern, fix it by **removal,
  not renaming** — renaming a variable (or switching to a synonym API) to
  dodge the regex is scanner evasion, and scanners say so.
- **Don't take secrets as CLI flags either.** Command-line arguments leak
  into process listings, shell history, and agent transcripts. The one
  scan-clean credential channel is a config file written by a **user-run**
  `init` (hidden `getpass` prompt, file mode 600). On machines with a
  persistent home that's runtime-agnostic — every runtime reads the same
  file.
- **Ephemeral cloud workspaces bridge via the platform's secrets, in the
  setup hook.** Claude Code web, Codex cloud, and CI have no interactive
  prompt and no persistent home; their native secret mechanism is the
  workspace env. Keep the skill's *code* env-free anyway, and put the
  bridge in the environment's setup hook (Codex setup script, devcontainer
  `postCreateCommand`, a CI step): a one-liner that materializes the config
  file from the workspace secret (see illo's README, "Cloud & CI"). Scanned
  prose documenting that shell one-liner passes `skills_guard` — the
  exfiltration patterns target code reads (`os.environ`, `printenv`,
  curl/wget interpolation), not a `printf` redirect in docs. The consent
  line that makes this safe: a **platform-provisioned workspace secret** is
  deliberate and scoped to that workspace, so an agent may seed the config
  from it once; an **ambient env var on a personal machine** proves nothing
  about intent (it may belong to other tools) — never copy it.
- **Keep credentials out of frontmatter.** No `envVars:`-style declarations
  for secrets; credential setup belongs in body text as something the user
  runs themselves. Agents must not enter, paste, print, or store a user's
  key.
- **Budget the installed bundle: ≤ 1 MB total, no file over 256 KB.** Keep
  docs-only assets (calibration examples, screenshots) **outside the skill
  directory** — in a sibling `<skill>-examples/` dir — and link them by raw
  GitHub URL. A `.skillignore` is not enough: the scanner honors it but
  installers copy the whole skill directory verbatim, so ignored files still
  ship and bloat every install. Compress only what must ship.
- **Re-verify any compressed asset that is a functional input — by running
  it, on every backend.** Format support differs per provider: Azure's
  image API rejects WebP reference images ("Only JPEG and PNG are
  supported") while Google's and xAI's accept them. Prefer JPEG/PNG for
  images sent to third-party APIs, and after recompressing, make the real
  call against each supported model/provider and inspect the output — file
  size and local rendering prove nothing about API acceptance or fidelity.
- **Prefer stdlib over subprocess.** Scanners flag subprocess execution and
  most uses have a stdlib equivalent (`webbrowser.open` instead of shelling
  out to `open`/`xdg-open`).
- **Pin every install command** quoted in docs or code
  (`pip install 'PyYAML==6.0.2'`, `npx -y tool@1.2.3`) — unpinned installs
  scan as supply-chain risk and drift anyway.
- **Binary assets need a Hermes repair preflight.** Some Hermes versions
  corrupt binary files (images, fonts, models) when installing multi-file
  skills from GitHub — binaries get decoded as text; text files survive.
  The pattern (see illo): a generated manifest `assets/checksums.txt`
  (`<sha256>  <pin-commit>  <relpath>`, written by
  `.github/scripts/regen_asset_checksums.py`, kept current automatically by
  the `asset-checksums` workflow — PRs touching assets get the regenerated
  manifest committed back to their branch); a generic, never-edited
  `scripts/repair-hermes-assets.sh` that verifies each asset and re-downloads
  mismatches from the immutable raw URL its pin commit implies; a magic-byte
  check in the engine's preflight (`doctor`) so *every* runtime detects
  corruption; and a SKILL.md section scoping the repair run to Hermes Agent
  only (`bash ${HERMES_SKILL_DIR}/scripts/repair-hermes-assets.sh`). The
  script is checksum-gated, so it's harmless anywhere. **Remove the preflight
  once Hermes ships its installer fix** — the detection in `doctor` can stay.

## License guidance

Default new skills to `license: MIT` in frontmatter because the repository root
license is MIT. Use `MIT-0` or another per-skill license only when there is a
clear reason, such as intentionally removing attribution requirements, matching
an upstream asset/license constraint, or documenting a third-party-derived work
that cannot honestly be represented as MIT. If using a non-default license,
state the reason in the PR body and keep any required notices with the skill.

## illo has moved

The `illo` skill's canonical home is now
[tmchow/illo-skill](https://github.com/tmchow/illo-skill) — development,
version bumps, and ClawHub publishing all happen there, and it is
deliberately absent from this repo's publish registry. The copy in `illo/`
is frozen for backwards compatibility: existing installs reference this
repo's paths, and `_assets/illo/` raw URLs on `main` are baked into
already-installed copies — so keep `illo/` and `_assets/illo/` present on
`main`; never delete or restructure them. The illo-specific conventions
that used to live in this file (character packs, looks) moved to the new
repo's AGENTS.md, which is the authoritative version.

## Publishing to ClawHub

ClawHub publishing is **opt-in per skill, automatic per merge**. The
registry is `.github/clawhub-publishable.txt` — one directory name per line,
living beside the publish workflow on purpose (one file to read to know what
ships). A skill is published if and only if it is listed there; opting in is
a deliberate act done via PR.

- **On merge to main**, `.github/workflows/publish-clawhub.yml` publishes
  every opted-in skill the push touched — but only when the skill's
  `SKILL.md` `version:` is **new** on ClawHub. A touched skill without a
  version bump is skipped quietly (docs-only merges stay green); bump
  `version:` in the same PR whenever a change should ship.
- **Manual dispatch** publishes one named skill and is strict: an
  already-published version fails loudly. Use it for first publishes and
  re-runs. Inputs: `skill` (required), `changelog` (optional, defaults to a
  sha-stamped message).
- Auth comes from the `CLAW_TOKEN` repository secret (a ClawHub API token).

**Consistency rule:** a skill whose README documents the OpenClaw install
lane should be opted into the registry, and every registry entry's README
should document that lane — the two lists must not drift apart. After a
skill's **first** publish, replace its README's provisional ClawHub slug
note and update the root catalog (per the per-skill README rules above).

## Validate before committing

- Frontmatter parses as valid YAML.
- `name` == directory name, lowercase kebab-case.
- `description` ≤ 1024 characters.
- Any install/command syntax in `README.md` or `SKILL.md` is real — checked
  against the tool's `--help`, not guessed.
- Installer syntax is runtime-specific. For the skills CLI, verify against `npx -y skills --help`; for Hermes, verify against `hermes skills --help` and `/skills`; for OpenClaw, verify against ClawHub publish/install output and `openclaw skills --help`. A sibling `README.md` is for humans and must never be required for the skill to run.
