# camofox-cloaked-browser

Agent skill for using [Camofox/Camoufox](https://github.com/jo-inc/camofox-browser) as an opt-in anti-detection browser server when normal automation is likely to get blocked.

This skill is intentionally **npm-first**. It does not include a container workflow.

## Prerequisites

- Node.js 22+
- npm / npx
- curl for raw REST examples
- Optional: OpenClaw with the upstream `@askjo/camofox-browser` plugin installed

## Install this skill

### Hermes

Install from the raw `SKILL.md` URL:

```bash
hermes skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/camofox-cloaked-browser/SKILL.md
```

From an interactive Hermes CLI session, use the slash command path:

```text
/skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/camofox-cloaked-browser/SKILL.md
/reload-skills
/skill camofox-cloaked-browser
```

Use `/reload-skills` if installing into an already-running session; then load it with `/skill camofox-cloaked-browser` when needed.

### OpenClaw

Install from ClawHub:

```bash
openclaw skills install camofox-cloaked-browser
```

ClawHub page: https://clawhub.ai/tmchow/camofox-cloaked-browser

## Default Camofox target

```text
http://127.0.0.1:9377
```

## Start Camofox locally

```bash
npx -y @askjo/camofox-browser
```

Or from a clone:

```bash
git clone https://github.com/jo-inc/camofox-browser
cd camofox-browser
npm install
npm start
```

The first install/run downloads the Camoufox browser binary unless `CAMOUFOX_EXECUTABLE` points to an existing compatible bundle.

## What the skill teaches agents

- When Camofox is actually warranted and when to use cheaper tools instead
- OpenClaw plugin tools and CLI commands: `openclaw camofox status/start/stop/tabs/configure`
- Raw REST API commands for tabs, navigation, snapshots, clicks, typing, scrolling, screenshots, links/images, JS evaluation, structured extraction, and cleanup
- Hard workflow rules: check `/health`, always send `userId`, prefer `sessionKey`, snapshot before refs, re-snapshot after state changes
- Environment variables for npm/server mode, auth, telemetry, browser binary overrides, proxy settings, and Hermes `CAMOFOX_URL`
- Hermes-specific gotcha: globally visible `CAMOFOX_URL` routes Hermes browser calls through Camofox for that process

## Important defaults

- Default base URL: `http://127.0.0.1:9377`
- No auth is needed for local loopback development unless sensitive endpoints are used
- Set `CAMOFOX_ACCESS_KEY` if exposing beyond localhost
- Set `CAMOFOX_API_KEY` for cookie import
- Set `CAMOFOX_CRASH_REPORT_ENABLED=false` to disable upstream crash/hang telemetry
