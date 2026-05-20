# agent-skills

A personal collection of cross-platform AI agent skills ([`SKILL.md`](https://agentskills.io/specification)
format). Each skill is self-contained — install the ones you want and skip the
rest. No per-platform plugin to maintain; the skills work anywhere the Agent
Skills standard is supported (Claude Code, OpenClaw, Hermes, Cursor, OpenCode,
GitHub Copilot CLI, and others).

## Install

Skills install with [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# List what's in the repo without installing
npx skills add tmchow/agent-skills --list

# Install one skill (user-level, all detected agents)
npx skills add tmchow/agent-skills --skill clawpatch --global

# Install everything
npx skills add tmchow/agent-skills --all
```

Update installed skills by name, or all at once:

```bash
npx skills update clawpatch   # one skill, by its installed name
npx skills update             # everything
```

## Skills

| Skill | What it does |
| ----- | ------------ |
| [`clawpatch`](clawpatch/) | Drive the [Clawpatch](https://clawpatch.ai) CLI for automated AI code review and per-finding fixes — scanner-only subagent dispatch (parallel) or full-cycle `clawpatch fix` loops, with wire-format JSON parsing, mode selection, and exit-code/pitfall handling. |

## Layout

Each skill lives in its own top-level directory with a `SKILL.md` (agent
instructions) and a `README.md` (human landing page — what it does,
prerequisites, install) at its root. Deep material can optionally go in a
`references/` subdir, but skills should stay thin and defer to the wrapped
tool's own `--help`/output rather than re-document it:

```
agent-skills/
├── clawpatch/
│   ├── SKILL.md          # agent-facing instructions
│   └── README.md         # human landing page
└── <next-skill>/
    ├── SKILL.md
    └── README.md
```

Pull the whole repo or a single skill with `--skill <name>`.

Adding a skill? See [`AGENTS.md`](AGENTS.md) for the conventions (frontmatter
rules, the SKILL.md-vs-README split, validation).

## Compatibility

Every skill declares the two required fields (`name`, `description`) plus
`version`, so it works across any Agent-Skills-compatible runtime. Unknown
frontmatter fields are ignored per the standard, which lets individual skills
add runtime-specific metadata without breaking others — for example,
`clawpatch` carries `metadata.openclaw` (npm install directive, optional env
vars) and `metadata.hermes` (tags, required toolset) blocks that only their
respective runtimes read.

## License

[MIT](LICENSE) © Trevin Chow
