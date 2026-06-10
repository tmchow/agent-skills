# chrome-devtools-axi

Agent-facing browser automation skill for [`chrome-devtools-axi`](https://github.com/kunchenguid/chrome-devtools-axi), an AXI-style CLI wrapper around Chrome DevTools MCP. It gives agents compact accessibility snapshots, stale-ref-safe interactions, page screenshots, console/network inspection, and performance tooling from a reusable Chrome DevTools session.

## Prerequisites

- Node.js 20+
- `npx` or npm global installs
- Chrome/Chromium availability through `chrome-devtools-mcp`
- Optional: a DevTools endpoint if connecting to an existing browser with `CHROME_DEVTOOLS_AXI_BROWSER_URL`

No API key is required. `CHROME_DEVTOOLS_AXI_WS_HEADERS` can contain secrets for authenticated remote WebSocket endpoints; do not paste or publish those values.

## Install

### Hermes

```bash
hermes skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/chrome-devtools-axi/SKILL.md
```

From an interactive Hermes session:

```text
/skills install https://raw.githubusercontent.com/tmchow/agent-skills/main/chrome-devtools-axi/SKILL.md
/reload-skills
/skill chrome-devtools-axi
```

### OpenClaw

After this skill is published to ClawHub, install it with:

```bash
openclaw skills install chrome-devtools-axi
```

ClawHub page, after publish: https://clawhub.ai/tmchow/chrome-devtools-axi

### CLI

Use `npx` for one-off browser work:

```bash
npx -y chrome-devtools-axi --help
npx -y chrome-devtools-axi open https://example.com
```

Or install globally for repeated shell use:

```bash
npm install -g chrome-devtools-axi
chrome-devtools-axi --help
```

## Capabilities

- Navigate and capture compact accessibility snapshots with element refs
- Click, fill, type, press keys, upload files, handle dialogs, drag, hover, and scroll
- Manage tabs/pages and viewport/device emulation
- Capture screenshots
- Inspect console messages and network requests
- Run Lighthouse, performance traces, and heap snapshots when debugging
- Connect to an existing Chrome/DevTools endpoint or launch a fresh browser session

SKILL.md is the agent-facing instructions; you don't need to read it to use the skill.
