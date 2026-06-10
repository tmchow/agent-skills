# agent-skills

A personal, multi-skill collection of cross-platform AI agent skills
(`SKILL.md` format), installable à la carte through runtime-specific lanes.
This repo currently targets Hermes and OpenClaw first: Hermes installs from raw
`SKILL.md` URLs via `hermes skills install` or `/skills install`, while OpenClaw
installs published skills from ClawHub via `openclaw skills install <slug>`.
Generic Agent Skills installers may work too, but do not document them as the
only path.

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
- install/verify commands
- a small set of canonical examples, not an exhaustive command list
- failure handling and verification discipline
- privacy, security, and side-effect boundaries

Avoid full command tables, flag catalogs, exit-code catalogs, or examples likely
to drift. Tell agents to run the live tool help (`<tool> --help`, subcommand
help, or upstream docs) for exact syntax. If quoting syntax anyway, verify it
against current help during the PR.

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
- Runtime-specific install commands for *this* skill: Hermes CLI/slash-command install and OpenClaw ClawHub install. If the ClawHub slug is not known yet, mark it provisional and update it after publish before merge.
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

## Validate before committing

- Frontmatter parses as valid YAML.
- `name` == directory name, lowercase kebab-case.
- `description` ≤ 1024 characters.
- Any install/command syntax in `README.md` or `SKILL.md` is real — checked
  against the tool's `--help`, not guessed.
- Installer syntax is runtime-specific. For Hermes, verify against `hermes skills --help` and `/skills`; for OpenClaw, verify against ClawHub publish/install output and `openclaw skills --help`. A sibling `README.md` is for humans and must never be required for the skill to run.
