# agent-skills

A personal, multi-skill collection of cross-platform AI agent skills
(`SKILL.md` format), installable à la carte through runtime-specific lanes:
the generic skills CLI (`npx skills add tmchow/agent-skills --skill <name>`,
works across Claude Code/Cursor/Codex and other Agent-Skills runtimes),
Hermes (raw `SKILL.md` URLs via `hermes skills install` or `/skills
install`), and OpenClaw (ClawHub via `openclaw skills install <slug>`).
Document lanes side by side; never present one as the only path.

This guide is for anyone (human or agent) editing the repo. Keep it accurate
when conventions change.

## Repo layout

- `<skill-name>/SKILL.md` — required. The agent-facing instructions.
- `<skill-name>/README.md` — required. The human-facing landing page (below).
- `<skill-name>/references/` — optional. Deep material loaded on demand.
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
  (`npx skills add tmchow/agent-skills --skill <name>`) first, then the Hermes
  CLI/slash-command install and OpenClaw ClawHub install. If the ClawHub slug
  is not known yet, mark it provisional and update it after publish before
  merge.
- A few capability bullets and a link to the upstream tool/API.
- An explicit line: "SKILL.md is the agent-facing instructions; you don't
  need to read it to use the skill."

**Do not** restate the workflow, schema, exit codes, or step-by-step
procedure from `SKILL.md` — that duplicates the agent doc and drifts. Keep the
README to slow-changing metadata (purpose, prerequisites, install).

## License guidance

Default new skills to `license: MIT` in frontmatter because the repository root
license is MIT. Use `MIT-0` or another per-skill license only when there is a
clear reason, such as intentionally removing attribution requirements, matching
an upstream asset/license constraint, or documenting a third-party-derived work
that cannot honestly be represented as MIT. If using a non-default license,
state the reason in the PR body and keep any required notices with the skill.

## illo character packs

The `illo` skill has a companion content repo,
[tmchow/illo-characters](https://github.com/tmchow/illo-characters). When
creating or editing character packs anywhere (that repo, a user's local
`~/.config/illo/characters/`, or skill docs/examples): **every character pack
name is globally unique** — names are the selection keys agents use ("use
anvil"), and illo-characters' `index.json` is the ecosystem registry. Check
it before naming a character. Reserved names: `blot` (ships with the skill),
`illo`, and the look names (`riso`, `blueprint`, `woodcut`, `pixel`, `clay`,
`manila`, `chalk`, `phosphor`, `enamel`, `gouache`).

## illo looks (style definitions)

Styles split deliberately across the two repos: a character pack carries only
a style **name** (its `Style:` line); the style **definition** — the ~3 KB
prompt-block file in `illo/references/styles/<name>.md` — lives here, because
style files are engine interface (their sections slot into
`references/prompt-recipe.md` and must evolve with it) and shared
infrastructure (one fix improves every pack that references the look). The
consequences:

- **Adding a character** never touches this repo — packs reference looks by
  name and ship entirely through illo-characters.
- **Adding a look** is a PR here: a new `references/styles/<name>.md` in the
  established format (prompt blocks, palette mapping, character treatment,
  labels, QA deltas), plus updating every enumeration of the look library
  (SKILL.md body + references list, `character-builder.md` interview +
  reserved names, the illo README "Looks" table, and the reserved names
  above). The SKILL.md description states only the look *count* ("ten
  bundled print looks") — bump the number, never enumerate names there. New look names must not collide with any existing character
  pack name in illo-characters' `index.json` — they become reserved both ways.
- **No PR needed to experiment**: a custom style file at
  `~/.config/illo/styles/<name>.md` works immediately for that user; promote
  it here once it proves out.

## Validate before committing

- Frontmatter parses as valid YAML.
- `name` == directory name, lowercase kebab-case.
- `description` ≤ 1024 characters.
- Any install/command syntax in `README.md` or `SKILL.md` is real — checked
  against the tool's `--help`, not guessed.
- Installer syntax is runtime-specific. For the skills CLI, verify against `npx -y skills --help`; for Hermes, verify against `hermes skills --help` and `/skills`; for OpenClaw, verify against ClawHub publish/install output and `openclaw skills --help`. A sibling `README.md` is for humans and must never be required for the skill to run.
