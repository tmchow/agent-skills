---
name: chrome-devtools-axi
description: >-
  This skill should be used when the user names chrome-devtools-axi, asks to
  "execute npx -y chrome-devtools-axi", wants Chrome DevTools Protocol browser
  automation through the AXI CLI, or needs token-efficient accessibility
  snapshots, stale-ref-safe clicks/fills, console/network inspection,
  screenshots, Lighthouse, or performance traces from a Chrome session. Do NOT
  use for simple web search/static extraction, stealth/anti-detection browsing,
  or non-Chrome browser targets.
version: 0.1.0
author: Trevin Chow
license: MIT
platforms: [macos, linux, windows]
metadata:
  hermes:
    tags: [browser, chrome, devtools, cdp, accessibility, automation, axi, cli]
    category: browser
    requires_toolsets: [terminal]
  openclaw:
    emoji: "🧭"
    homepage: https://github.com/kunchenguid/chrome-devtools-axi
    requires:
      bins: [node, npx]
    install:
      - kind: node
        package: chrome-devtools-axi
        bins: [chrome-devtools-axi]
    envVars:
      - name: CHROME_DEVTOOLS_AXI_AUTO_CONNECT
        required: false
        description: Set to 1 to connect to the user's running Chrome 144+ via remote debugging instead of launching a new browser.
      - name: CHROME_DEVTOOLS_AXI_HEADED
        required: false
        description: Set to 1 to run Chrome in headed/visible mode.
      - name: CHROME_DEVTOOLS_AXI_CHROME_ARGS
        required: false
        description: Whitespace-separated Chrome flags forwarded to the browser; no shell-style quoting. Useful with headed GPU/WebGL/WebGPU debugging.
      - name: CHROME_DEVTOOLS_AXI_BROWSER_URL
        required: false
        description: Existing Chrome DevTools endpoint. Supports http(s):// browserUrl or ws(s):// WebSocket endpoint.
      - name: CHROME_DEVTOOLS_AXI_WS_HEADERS
        required: false
        description: JSON headers for authenticated ws(s) endpoints. Treat as secret-bearing and never print raw values.
      - name: CHROME_DEVTOOLS_AXI_USER_DATA_DIR
        required: false
        description: Persistent Chrome profile directory. Skips isolated mode; use only when persistent session state is intentional.
      - name: CHROME_DEVTOOLS_AXI_PORT
        required: false
        description: Local bridge server port. Defaults to 9224.
      - name: CHROME_DEVTOOLS_AXI_MCP_PATH
        required: false
        description: Absolute path to a chrome-devtools-mcp script; avoids slow npx bootstrap on cold systems.
      - name: CHROME_DEVTOOLS_AXI_BRIDGE_TIMEOUT_MS
        required: false
        description: Bridge readiness deadline in milliseconds. Defaults to 30000.
---

# Chrome DevTools AXI

## When to use

Use this skill when the task specifically needs `chrome-devtools-axi` or a
Chrome DevTools Protocol browser controlled from the terminal:

- the user says `chrome-devtools-axi`, `chrome-devtools-mcp`, AXI browser automation, or asks to run `npx -y chrome-devtools-axi`
- an agent needs a compact accessibility snapshot with actionable refs and command suggestions
- browser work needs console logs, network requests, screenshots, Lighthouse, heap snapshots, or performance traces in the same CLI surface
- the existing Hermes/browser stack is unavailable or the user explicitly prefers this tool

Do **not** use it for ordinary web search, curl-able pages, static extraction,
or stealth/anti-detection work. Use Camofox/Camoufox only when cloaking is
load-bearing. Use built-in browser tools when the task is already solved there
and the user did not name `chrome-devtools-axi`.

## Mental model

`chrome-devtools-axi` is an npm CLI wrapper around `chrome-devtools-mcp`. The
first command starts a persistent local bridge, then later invocations reuse the
same Chrome/DevTools session. Output is AXI/TOON-style: page metadata,
accessibility snapshots, element refs, and next-step suggestions.

State lives under `~/.chrome-devtools-axi/`, including the bridge PID and
snapshot-generation counter.

## Setup and verification

The skill only teaches the agent how to use the tool. The actual CLI is the npm
package `chrome-devtools-axi`; source repo:
https://github.com/kunchenguid/chrome-devtools-axi

Prefer `npx` for one-off use so the CLI is installed on demand:

```bash
npx -y chrome-devtools-axi --help
npx -y chrome-devtools-axi
```

Use a global install only when repeated shell use matters:

```bash
npm install -g chrome-devtools-axi
chrome-devtools-axi --help
```

Before relying on the tool, verify the live CLI rather than assuming docs are
current:

```bash
npx -y chrome-devtools-axi --version
npx -y chrome-devtools-axi --help
```

If the home view says `browser: no active session`, open a page:

```bash
npx -y chrome-devtools-axi open https://example.com
```

## Core workflow

1. Start from the current state or open a page:

   ```bash
   npx -y chrome-devtools-axi
   npx -y chrome-devtools-axi open https://example.com
   ```

2. Use the latest snapshot refs exactly as printed. They look like `@g<N>:...`
   and may include underscores, for example `@g4:1_3`:

   ```bash
   npx -y chrome-devtools-axi click @<uid>
   npx -y chrome-devtools-axi fill @<uid> "search text"
   npx -y chrome-devtools-axi press Enter
   ```

3. Re-snapshot after every state-changing action:

   ```bash
   npx -y chrome-devtools-axi snapshot
   ```

4. Inspect browser diagnostics when debugging app behavior:

   ```bash
   npx -y chrome-devtools-axi console --type error --limit 20
   npx -y chrome-devtools-axi network --type fetch --limit 20
   npx -y chrome-devtools-axi network-get <id> --response-file /tmp/response.json
   ```

5. Stop the bridge when preserving the browser session is not useful:

   ```bash
   npx -y chrome-devtools-axi stop
   ```

## Ref discipline

Refs include a generation prefix such as `@g1:1_7`. Treat refs as scoped to the
latest accessibility snapshot.

Hard rules:

- Pass refs back exactly as printed, including `@` and the generation prefix.
- Re-snapshot after navigation, click, form submit, scroll, dialog handling,
  upload, drag, JS mutation, page selection, or resize.
- If a command fails with `STALE_REF`, do not retry the same ref blindly.
  Snapshot again, find the new ref, then retry.
- Prefer refs over CSS selectors or DOM guessing. Use `eval` only when the
  accessibility surface cannot expose the needed state.

## Command usage policy

Do not treat this skill as the CLI reference. For exact flags and subcommands,
run the live help first:

```bash
npx -y chrome-devtools-axi --help
```

Use this skill for durable agent judgment: when to choose the tool, how to
sequence work, which commands are high leverage, how to verify actions, and
which modes are privacy-sensitive.

## Best-practice scenarios

**Explore or verify a page.** Open the URL, snapshot it, then verify with URL,
title, DOM, or screenshot evidence before reporting success:

```bash
npx -y chrome-devtools-axi open https://example.com
npx -y chrome-devtools-axi snapshot
npx -y chrome-devtools-axi eval "document.title"
npx -y chrome-devtools-axi screenshot /tmp/page.png
```

**Interact with a page or form.** Use refs from the latest snapshot exactly as
printed, then re-snapshot after each state-changing action:

```bash
npx -y chrome-devtools-axi click @<uid>
npx -y chrome-devtools-axi fill @<uid> "value"
npx -y chrome-devtools-axi press Enter
npx -y chrome-devtools-axi snapshot
```

**Debug a web app.** Check console and network before guessing. Save large
request/response bodies to files instead of dumping them into chat:

```bash
npx -y chrome-devtools-axi console --type error --limit 20
npx -y chrome-devtools-axi network --type fetch --limit 20
npx -y chrome-devtools-axi network-get <id> --response-file /tmp/response.json
```

**Use existing Chrome or logged-in state.** Prefer a fresh isolated browser.
Connect to an existing browser/profile only when the task requires it and the
user has authorized that scope:

```bash
CHROME_DEVTOOLS_AXI_BROWSER_URL=http://127.0.0.1:9222 npx -y chrome-devtools-axi snapshot
CHROME_DEVTOOLS_AXI_AUTO_CONNECT=1 npx -y chrome-devtools-axi snapshot
CHROME_DEVTOOLS_AXI_USER_DATA_DIR="$HOME/.chrome-devtools-axi/profile" npx -y chrome-devtools-axi open https://example.com
```

**Handle slow cold starts.** If repeated commands are slow because the bridge is
bootstrapping `chrome-devtools-mcp` through `npx`, use the CLI help's current
`CHROME_DEVTOOLS_AXI_MCP_PATH` recipe rather than inventing a path.

**Use visual/GPU-sensitive pages.** For WebGL/WebGPU/GPU debugging, use headed
mode and Chrome flags from the live help. Do not persist broad Chrome flags
globally.

For `run`, `emulate`, Lighthouse, performance traces, heap snapshots, or less
common flags, consult `--help`/upstream docs at the moment of use instead of
copying examples from memory.

## Existing Chrome, headed mode, and profiles

Use environment variables inline for a single command or process when possible.
Do not persist them globally unless the user explicitly wants that behavior.
Treat persistent profiles as privacy-sensitive. Do not browse authenticated
accounts, export cookies, or inspect private content unless the user explicitly
authorizes that scope.

## Diagnostics and performance

For app debugging, prefer the CLI's console/network commands before guessing.
Only run Lighthouse, traces, and heap snapshots when they materially help; they
are slower and noisier than simple snapshots, console checks, and network lists.

## Safety and privacy

- Do not run `setup hooks` unless the user explicitly asks to install ambient
  agent hooks. It mutates local agent configuration.
- Do not print `CHROME_DEVTOOLS_AXI_WS_HEADERS`; it may contain bearer tokens.
- Do not use a persistent `CHROME_DEVTOOLS_AXI_USER_DATA_DIR` casually; it can
  carry login state and browsing history.
- Do not claim a browser action succeeded until a fresh snapshot, URL/title,
  screenshot, console/network result, or DOM evaluation verifies it.
- Close pages or stop the bridge when done unless keeping state is useful for
  the user's next step.

## Common failure handling

- **No active session:** run `open <url>`.
- **Stale ref:** run `snapshot`, find the new ref, retry once.
- **Slow first command:** the bridge may be bootstrapping `chrome-devtools-mcp`
  through `npx`. If repeated cold starts hurt, globally install
  `chrome-devtools-mcp` and use `CHROME_DEVTOOLS_AXI_MCP_PATH` as described in
  `chrome-devtools-axi --help`.
- **Wrong tab:** run `pages`, then `selectpage <id>` and snapshot.
- **Need the user's normal Chrome:** use `CHROME_DEVTOOLS_AXI_AUTO_CONNECT=1`
  only when remote debugging is enabled and connecting to that browser is
  intentional.
