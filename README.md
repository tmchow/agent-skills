# agent-skills

A personal collection of cross-platform AI agent skills ([`SKILL.md`](https://agentskills.io/specification)
format). Each skill is self-contained — install the ones you want and skip the
rest. No per-platform plugin to maintain; the skills work anywhere the Agent
Skills standard is supported.

This repo currently targets **Hermes** and **OpenClaw** first. Installation is
runtime-specific; do not assume one installer command works everywhere.

## Install

### Hermes

Hermes can install a skill directly from a `SKILL.md` URL:

```bash
hermes skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/camofox-cloaked-browser/SKILL.md
hermes skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/clawpatch/SKILL.md
```

From an interactive Hermes CLI session, use the slash command equivalent:

```text
/skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/camofox-cloaked-browser/SKILL.md
/reload-skills
/skill camofox-cloaked-browser
```

Notes:

- `/skills` is the Hermes interactive slash command for search/install/inspect/manage.
- `/reload-skills` makes newly installed skills visible in the current session.
- `/skill <name>` loads an installed skill into the current session.

### OpenClaw

After a skill is published to ClawHub, install it with:

```bash
openclaw skills install <slug>
```

Expected slug for this skill is likely:

```bash
openclaw skills install camofox-cloaked-browser
```

Treat that as provisional until the ClawHub upload returns the canonical slug.
Before merging this PR, update this README and the skill README with the actual
published slug.

### Vercel Agent Skills / other runtimes

This repo may also be consumable by generic Agent Skills tooling, but those
commands are not the primary install path for Hermes/OpenClaw. If documenting
another runtime, add it as a separate lane rather than replacing the Hermes and
OpenClaw instructions.

## Skills

| Skill | What it does |
| ----- | ------------ |
| [`camofox-cloaked-browser`](camofox-cloaked-browser/) | Use [Camofox/Camoufox](https://github.com/jo-inc/camofox-browser) selectively for cloaked browser automation — npm/npx startup, OpenClaw plugin tools, raw REST commands, session/tab workflow, environment variables, process-scoped Hermes `CAMOFOX_URL`, and snapshot-after-action discipline. |
| [`clawpatch`](clawpatch/) | Drive the [Clawpatch](https://clawpatch.ai) CLI for automated AI code review and per-finding fixes — scanner-only subagent dispatch (parallel) or full-cycle `clawpatch fix` loops, with wire-format JSON parsing, mode selection, and exit-code/pitfall handling. |

## Layout

Each skill lives in its own top-level directory with a `SKILL.md` (agent
instructions) and a `README.md` (human landing page — what it does,
prerequisites, install) at its root. Deep material can optionally go in a
`references/` subdir, but skills should stay thin and defer to the wrapped
tool's own `--help`/output rather than re-document it:

```
agent-skills/
├── camofox-cloaked-browser/
│   ├── SKILL.md          # agent-facing instructions
│   └── README.md         # human landing page
├── clawpatch/
│   ├── SKILL.md          # agent-facing instructions
│   └── README.md         # human landing page
└── <next-skill>/
    ├── SKILL.md
    └── README.md
```

Adding a skill? See [`AGENTS.md`](AGENTS.md) for the conventions (frontmatter
rules, the SKILL.md-vs-README split, validation).

## Compatibility

Every skill declares the two required fields (`name`, `description`) plus
`version`, so it works across any Agent-Skills-compatible runtime. Unknown
frontmatter fields are ignored per the standard, which lets individual skills
add runtime-specific metadata without breaking others — for example,
`clawpatch` carries `metadata.openclaw` (npm install directive, optional env
vars) and `metadata.hermes` (tags, required toolset) blocks, while
`camofox-cloaked-browser` carries OpenClaw optional env-var declarations plus
Hermes skill config for the local Camofox base URL and explicitly scoped
Hermes-only `CAMOFOX_URL` guidance.

## License

[MIT](LICENSE) © Trevin Chow
