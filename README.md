# agent-skills

A personal collection of cross-platform AI agent skills ([`SKILL.md`](https://agentskills.io/specification)
format). Each skill is self-contained — install the ones you want and skip the
rest. No per-platform plugin to maintain; the skills work anywhere the Agent
Skills standard is supported (Claude Code, OpenClaw, Hermes, Cursor, OpenCode,
GitHub Copilot CLI, and others).

## Install

### With `npx skills` ([Vercel](https://github.com/vercel-labs/skills))

```bash
# List what's in the repo without installing
npx skills add tmchow/agent-skills --list

# Install one skill (user-level, all detected agents)
npx skills add tmchow/agent-skills --skill clawpatch --global

# Install everything
npx skills add tmchow/agent-skills --all
```

### With `gh skills` (GitHub CLI, preview)

```bash
gh skill preview tmchow/agent-skills clawpatch   # inspect first
gh skill install tmchow/agent-skills clawpatch   # install
```

## Skills

| Skill | What it does |
| ----- | ------------ |
| [`clawpatch`](clawpatch/) | Drive the [Clawpatch](https://clawpatch.ai) CLI for automated AI code review and per-finding fixes — scanner-only subagent dispatch (parallel) or full-cycle `clawpatch fix` loops, with wire-format JSON parsing, mode selection, and exit-code/pitfall handling. |

## Layout

Each skill lives in its own top-level directory with a `SKILL.md` (agent
instructions) and a `README.md` (human landing page — what it does,
prerequisites, install) at its root, plus any supporting material under
`references/`:

```
agent-skills/
├── clawpatch/
│   ├── SKILL.md          # agent-facing instructions
│   ├── README.md         # human landing page
│   └── references/
│       ├── finding-schema.md
│       └── full-cycle-mode.md
└── <next-skill>/
    ├── SKILL.md
    └── README.md
```

Installers can pull the whole repo or a single skill by name (`--skill <name>`
for `npx skills`, or the skill name as the second argument for `gh skill`).

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
