# vhs-terminal-recorder

Agent-facing skill for using [Charmbracelet VHS](https://github.com/charmbracelet/vhs) to create reproducible terminal recordings from `.tape` files. It helps agents author clean demos, validate tapes, render GIF/MP4/WebM output, and avoid common mistakes like recording local secrets, relying on ambient shell state, or treating `vhs validate` as an execution test.

## Prerequisites

- The `vhs` CLI installed and available on `PATH`
- VHS runtime dependencies for rendering in the current environment, especially `ttyd` and `ffmpeg`
- Any CLI programs shown in a tape, declared with `Require`

No API key is required for local rendering. Publishing to `vhs.charm.sh` is a public upload and should be done only when intended.

## Install VHS

This skill does not bundle VHS. The common macOS/Linux install path from upstream is:

```bash
brew install vhs
```

For Windows, Arch, Nix, Docker, Debian/RPM packages, or Go source installs, use the upstream installation docs: https://github.com/charmbracelet/vhs#installation

## Install

### Any Agent Skills runtime (skills CLI)

Install only this skill from the repo:

```bash
npx skills add tmchow/agent-skills --skill vhs-terminal-recorder
```

Add `--global` to install it at the user level instead of the current project:

```bash
npx skills add tmchow/agent-skills --skill vhs-terminal-recorder --global
```

### Hermes

```bash
hermes skills install tmchow/agent-skills/vhs-terminal-recorder
```

From an interactive Hermes session:

```text
/skills install tmchow/agent-skills/vhs-terminal-recorder
/reload-skills
/skill vhs-terminal-recorder
```

### OpenClaw

After this skill is published to ClawHub, install it with:

```bash
openclaw skills install vhs-terminal-recorder
```

ClawHub page, after publish: https://clawhub.ai/tmchow/vhs-terminal-recorder

## Capabilities

- Author deterministic `.tape` files for terminal demos
- Validate tapes before rendering
- Render GIF, MP4, and WebM outputs
- Use `Hide` / `Show`, `Require`, `Wait+Screen`, explicit shell/theme/size settings, and small fixtures effectively
- Debug failing tapes by separating parse errors from command/runtime failures
- Handle `vhs record`, `vhs themes`, and `vhs publish` with the right caution

SKILL.md is the agent-facing instructions; you don't need to read it to use the skill.
