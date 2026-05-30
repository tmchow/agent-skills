---
name: camofox-cloaked-browser
description: >-
  Use Camofox/Camoufox as an opt-in anti-detection browser server for agent
  workflows that need cloaked browsing. Covers npm/npx startup, OpenClaw plugin
  tools, REST API commands, session/tab workflow, environment variables,
  process-scoped CAMOFOX_URL for Hermes, and hard rules such as always sending
  userId and re-snapshotting after state-changing actions. Do not use for normal
  web search, text extraction, curl fetches, or ordinary browser automation.
version: 1.3.4
author: Trevin Chow
license: MIT-0
platforms: [macos, linux]
metadata:
  hermes:
    tags: [browser, camofox, camoufox, cloaking, anti-detection, npm, openclaw]
    category: browser
    related_skills: [hermes-agent]
    config:
      - key: camofox.base_url
        description: Local Camofox browser server base URL. Default is http://127.0.0.1:9377.
        default: "http://127.0.0.1:9377"
        prompt: Camofox server base URL
  openclaw:
    emoji: "🦊"
    os: [macos, linux]
    homepage: https://github.com/jo-inc/camofox-browser
    requires:
      bins: [node, npm, curl]
    install:
      - kind: node
        package: "@askjo/camofox-browser"
        bins: [camofox-browser]
    envVars:
      - name: CAMOFOX_PORT
        required: false
        description: Optional server port. Defaults to 9377. PORT is also honored.
      - name: PORT
        required: false
        description: Optional generic port override when CAMOFOX_PORT is not set.
      - name: CAMOFOX_BASE_URL
        required: false
        description: Optional shell/client convention used by this skill's raw REST examples. Defaults to http://127.0.0.1:9377.
      - name: CAMOFOX_USER_ID
        required: false
        description: Optional shell/client convention for raw REST examples. Always send a userId; defaults to agent1 in examples.
      - name: CAMOFOX_SESSION_KEY
        required: false
        description: Optional shell/client convention for raw REST examples. Prefer a sessionKey when creating tabs.
      - name: CAMOFOX_ACCESS_KEY
        required: false
        description: Optional global bearer token for all routes except /health. Use when exposing beyond localhost.
      - name: CAMOFOX_API_KEY
        required: false
        description: Optional bearer token used only for sensitive endpoints such as cookie import and traces. Not needed for normal browsing, snapshots, navigation, clicks, or typing.
      - name: CAMOFOX_CRASH_REPORT_ENABLED
        required: false
        description: Optional telemetry toggle. Set to false to disable anonymized crash/hang reports.
      - name: CAMOFOX_CRASH_REPORT_URL
        required: false
        description: Optional self-hosted crash-report endpoint override.
      - name: CAMOUFOX_EXECUTABLE
        required: false
        description: "Optional external Camoufox executable/bundle to avoid npm postinstall browser download. Aliases: CAMOUFOX_EXECUTABLE_PATH, CAMOFOX_EXECUTABLE_PATH."
      - name: CAMOUFOX_EXECUTABLE_PATH
        required: false
        description: Compatibility alias for CAMOUFOX_EXECUTABLE.
      - name: CAMOFOX_EXECUTABLE_PATH
        required: false
        description: Legacy compatibility alias for CAMOUFOX_EXECUTABLE.
      - name: CAMOFOX_URL
        required: false
        description: "Hermes-only client-side backend switch. Footgun: if set in Hermes global env/.env/gateway env, Hermes routes all browser calls in that process through Camofox. Do not persist or export globally; use only inline for a dedicated Hermes process."
---

# Camofox Cloaked Browser

## When to use

Use this skill only when the task needs Camofox/Camoufox specifically:

- user names Camofox, Camoufox, anti-detection browsing, cloaked browser, browser fingerprint spoofing, or stealth browsing
- a site is likely to block normal Playwright/Chrome automation
- the task needs stable accessibility refs from a Camofox browser server
- OpenClaw has the `camofox-browser` plugin/tools available

Do **not** use it for ordinary web search, simple page fetches, static text extraction, or normal browser automation. Use the cheaper default stack unless cloaking is actually load-bearing.

If the user names another browser target explicitly — Browserbase, Selkies, a tailnet browser, normal Hermes browser, Chrome DevTools, etc. — stop and use that target/skill instead. Do not silently route through Camofox.

## Default target

Default local server:

```text
http://127.0.0.1:9377
```

Equivalent localhost URL is usually fine:

```text
http://localhost:9377
```

Prefer `127.0.0.1` in examples to avoid IPv6/localhost oddities.

There is no container target in this skill. Do not mention or rely on container names; this skill is about the npm server and OpenClaw plugin/API.

## Operating model

Camofox Browser is a Node server and OpenClaw plugin wrapper around Camoufox, a Firefox-based anti-detection browser.

Primary local startup:

```bash
npx -y @askjo/camofox-browser
# serves http://127.0.0.1:9377 by default
```

Alternative cloned-repo startup:

```bash
git clone https://github.com/jo-inc/camofox-browser
cd camofox-browser
npm install
npm start
```

Alternative global install:

```bash
npm install -g @askjo/camofox-browser
camofox-browser
```

`npm install` / `npx` downloads the Camoufox browser binary on first run via the package postinstall unless `CAMOUFOX_EXECUTABLE` points to an existing compatible Camoufox bundle. Expect roughly a few hundred MB for the browser payload.

## OpenClaw plugin mode

If OpenClaw has the upstream plugin installed, prefer the plugin tools over raw `curl` because they auto-manage `userId` from `ctx.agentId`, use `sessionKey`, and can auto-start the server.

Install shape:

```bash
openclaw plugins install @askjo/camofox-browser
# or whatever ClawHub install command the registry page currently shows
```

Useful OpenClaw CLI commands from the plugin:

```bash
openclaw camofox status
openclaw camofox start
openclaw camofox stop
openclaw camofox tabs
openclaw camofox configure
```

Plugin config shape shown by upstream:

```yaml
plugins:
  entries:
    camofox-browser:
      enabled: true
      config:
        port: 9377
        autoStart: true
        maxSessions: 5
        maxTabsPerSession: 3
        sessionTimeoutMs: 600000
        browserIdleTimeoutMs: 300000
        maxOldSpaceSize: 128
```

The upstream plugin exposes these core tools:

- `camofox_create_tab` — create tab; returns `tabId`
- `camofox_snapshot` — accessibility snapshot with refs and screenshot; primary observation tool
- `camofox_click` — click by ref or CSS selector
- `camofox_type` — type by ref or selector; optional `pressEnter`
- `camofox_navigate` — navigate by URL or search macro
- `camofox_scroll` — scroll page
- `camofox_screenshot` — screenshot only
- `camofox_close_tab` — close a tab
- `camofox_evaluate` — execute JS; gated by server auth middleware
- `camofox_list_tabs` — list tabs for the current user
- `camofox_import_cookies` — import Netscape cookies; use `CAMOFOX_API_KEY` for this sensitive endpoint

## Hard workflow rules

Always follow these rules when using Camofox:

1. Check `/health` before doing browser work.
2. Always send `userId` in raw REST calls.
3. Prefer `sessionKey` when creating tabs so task tabs group together.
4. Open or reuse a tab intentionally; do not spray new tabs.
5. Snapshot before selecting refs.
6. Re-snapshot after every state-changing action: click, type with submit, press, scroll, navigation, back, forward, refresh, JS evaluate that mutates state.
7. Element refs reset after navigation and may become stale after DOM changes.
8. Prefer refs from the latest snapshot over CSS selectors. Use selectors only when refs are unavailable or unstable.
9. Close tabs when done unless preserving the session is explicitly useful.
10. Do not claim Camofox is in use until both the server is healthy and the actual agent/client is pointed at it.

## REST API quick commands

Set base variables:

```bash
BASE="${CAMOFOX_BASE_URL:-http://127.0.0.1:9377}"
USER_ID="${CAMOFOX_USER_ID:-agent1}"
SESSION_KEY="${CAMOFOX_SESSION_KEY:-task1}"
```

If `CAMOFOX_ACCESS_KEY` or `CAMOFOX_API_KEY` is set, include auth where required:

```bash
AUTH_HEADER=()
if [ -n "${CAMOFOX_ACCESS_KEY:-}" ]; then
  AUTH_HEADER=(-H "Authorization: Bearer ${CAMOFOX_ACCESS_KEY}")
elif [ -n "${CAMOFOX_API_KEY:-}" ]; then
  AUTH_HEADER=(-H "Authorization: Bearer ${CAMOFOX_API_KEY}")
fi
```

### Health

```bash
curl -fsS "$BASE/health"
```

### Create tab

```bash
TAB_ID="$(curl -fsS -X POST "$BASE/tabs" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"sessionKey\":\"$SESSION_KEY\",\"url\":\"https://example.com\"}" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["tabId"])')"
printf 'TAB_ID=%s\n' "$TAB_ID"
```

### Navigate

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/navigate" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"url\":\"https://example.com\"}"
```

Search macro example:

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/navigate" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"macro\":\"@google_search\",\"query\":\"site:example.com pricing\"}"
```

Known macros include:

- `@google_search`
- `@youtube_search`
- `@amazon_search`
- `@reddit_search`
- `@wikipedia_search`
- `@twitter_search`
- `@yelp_search`
- `@spotify_search`
- `@netflix_search`
- `@linkedin_search`
- `@instagram_search`
- `@tiktok_search`
- `@twitch_search`

### Snapshot

```bash
curl -fsS "$BASE/tabs/$TAB_ID/snapshot?userId=$USER_ID"
```

With screenshot and pagination offset:

```bash
curl -fsS "$BASE/tabs/$TAB_ID/snapshot?userId=$USER_ID&includeScreenshot=true&offset=0"
```

### Click

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/click" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"ref\":\"e1\"}"
```

Selector fallback:

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/click" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"selector\":\"button[type=submit]\"}"
```

### Type

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/type" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"ref\":\"e2\",\"text\":\"hello world\"}"
```

Type and submit:

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/type" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"ref\":\"e2\",\"text\":\"query\",\"pressEnter\":true}"
```

### Press key

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/press" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"key\":\"Enter\"}"
```

### Wait

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/wait" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"timeout\":3000}"
```

### Scroll

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/scroll" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"direction\":\"down\",\"amount\":700}"
```

### Back / forward / refresh

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/back"    -H 'Content-Type: application/json' "${AUTH_HEADER[@]}" -d "{\"userId\":\"$USER_ID\"}"
curl -fsS -X POST "$BASE/tabs/$TAB_ID/forward" -H 'Content-Type: application/json' "${AUTH_HEADER[@]}" -d "{\"userId\":\"$USER_ID\"}"
curl -fsS -X POST "$BASE/tabs/$TAB_ID/refresh" -H 'Content-Type: application/json' "${AUTH_HEADER[@]}" -d "{\"userId\":\"$USER_ID\"}"
```

### Links / images / screenshot

```bash
curl -fsS "$BASE/tabs/$TAB_ID/links?userId=$USER_ID&limit=50"
curl -fsS "$BASE/tabs/$TAB_ID/images?userId=$USER_ID&limit=50"
curl -fsS "$BASE/tabs/$TAB_ID/screenshot?userId=$USER_ID" --output screenshot.png
```

### Evaluate JavaScript

This endpoint is auth-gated by the server middleware. Use only when needed.

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/evaluate" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"expression\":\"document.title\"}"
```

### Structured extract

```bash
curl -fsS -X POST "$BASE/tabs/$TAB_ID/extract" \
  -H 'Content-Type: application/json' \
  "${AUTH_HEADER[@]}" \
  -d "{\"userId\":\"$USER_ID\",\"schema\":{\"type\":\"object\",\"properties\":{\"title\":{\"type\":\"string\"}}}}"
```

### List and close tabs

```bash
curl -fsS "$BASE/tabs?userId=$USER_ID"
curl -fsS -X DELETE "$BASE/tabs/$TAB_ID?userId=$USER_ID" "${AUTH_HEADER[@]}"
```

### Delete session data

This endpoint is auth-gated.

```bash
curl -fsS -X DELETE "$BASE/sessions/$USER_ID" "${AUTH_HEADER[@]}"
```

## Environment variables

### Server/client target

- `CAMOFOX_PORT`: server port; default `9377`
- `PORT`: generic port fallback if `CAMOFOX_PORT` is unset
- `CAMOFOX_BASE_URL`: not an upstream server env var; useful shell convention for scripts in this skill
- `CAMOFOX_URL`: Hermes-specific browser-backend switch; **do not set/export globally**; see below

### Auth and sensitive operations

- `CAMOFOX_ACCESS_KEY`: global bearer token for all routes except `/health`; use when anything can reach the server beyond loopback
- `CAMOFOX_API_KEY`: optional bearer token used only for sensitive endpoints such as cookie import/traces; not needed for normal browsing, snapshots, navigation, clicks, or typing
- `CAMOFOX_ADMIN_KEY`: protects `/stop` when configured

For local development with neither key set, upstream allows loopback requests in non-production mode. Do not expose that to a network. That would be the kind of convenience feature that becomes an incident report.

### Storage / runtime tuning

- `CAMOFOX_COOKIES_DIR`: cookie import source directory; default `~/.camofox/cookies`
- `CAMOFOX_PROFILE_DIR`: profile directory; default `~/.camofox/profiles`
- `CAMOFOX_TRACES_DIR`: trace directory; default `~/.camofox/traces`
- `CAMOFOX_TRACES_MAX_BYTES`: default 50MB
- `CAMOFOX_TRACES_TTL_HOURS`: default 24
- `MAX_CONCURRENT_PER_USER`: default 3
- `MAX_SESSIONS`: default 50
- `MAX_TABS_PER_SESSION`: default 10
- `MAX_TABS_GLOBAL`: default 50
- `SESSION_TIMEOUT_MS`: default 600000 in current code
- `TAB_INACTIVITY_MS`: default 300000
- `BROWSER_IDLE_TIMEOUT_MS`: default 300000
- `NAVIGATE_TIMEOUT_MS`: default 25000
- `BUILDREFS_TIMEOUT_MS`: default 12000
- `NATIVE_MEM_RESTART_THRESHOLD_MB`: default 300
- `BROWSER_RSS_RESTART_THRESHOLD_MB`: default 1500

### Browser binary

- `CAMOUFOX_EXECUTABLE`: external Camoufox executable/bundle; skips bundled download when valid
- `CAMOUFOX_EXECUTABLE_PATH`: compatibility alias
- `CAMOFOX_EXECUTABLE_PATH`: legacy alias
- `CAMOFOX_SKIP_DOWNLOAD=1|true`: skip postinstall browser download; only use if an executable is provided another way

### Proxy / GeoIP

- `PROXY_STRATEGY`
- `PROXY_PROVIDER`
- `PROXY_HOST`
- `PROXY_PORT`
- `PROXY_PORTS`
- `PROXY_USERNAME`
- `PROXY_PASSWORD`
- `PROXY_BACKCONNECT_HOST`
- `PROXY_BACKCONNECT_PORT`
- `PROXY_COUNTRY`
- `PROXY_STATE`
- `PROXY_CITY`
- `PROXY_ZIP`
- `PROXY_SESSION_DURATION_MINUTES`

### Telemetry

Upstream crash/hang telemetry is enabled unless disabled:

```bash
CAMOFOX_CRASH_REPORT_ENABLED=false
```

Use that for privacy-conservative local runs unless the user wants upstream crash reporting.

## Hermes-specific `CAMOFOX_URL` footgun

This section is load-bearing for Hermes agents. Read it before setting environment variables.

In Hermes, `CAMOFOX_URL` is not just a harmless client hint. It is a browser-backend switch. If `CAMOFOX_URL` is visible in the Hermes process environment, Hermes considers Camofox mode enabled for that process and normal Hermes browser calls can be forced through Camofox.

Hard rules for Hermes:

- **Do not** add `CAMOFOX_URL` to `~/.hermes/.env`.
- **Do not** export `CAMOFOX_URL` in shell profiles such as `~/.zshrc`, `~/.bashrc`, launchd plists, Docker env, or service env.
- **Do not** put `CAMOFOX_URL` in the Hermes gateway environment unless the explicit intent is for the entire gateway process to use Camofox for browser calls.
- **Do not** use `hermes config set ... CAMOFOX_URL ...`; this is runtime state, not durable config.
- Use `CAMOFOX_URL` only as an inline, one-process env var for a dedicated cloaked Hermes run:

```bash
CAMOFOX_URL="http://127.0.0.1:9377" hermes chat -q 'Use cloaked Camofox browsing for this task: <task>'
```

- After removing an accidentally global `CAMOFOX_URL`, restart the affected Hermes CLI/gateway process; running processes keep their old environment.

Safer default for Hermes agents: run the Camofox server normally, store the stable server URL as `skills.config.camofox.base_url` or `CAMOFOX_BASE_URL` for REST examples, and **do not set `CAMOFOX_URL` at all** unless launching a dedicated cloaked Hermes process.

This routing claim is Hermes-specific. For OpenClaw, use the upstream plugin config/tools unless you have verified equivalent env semantics.

## Verification checklist

Before claiming success:

1. Server health responds:

```bash
curl -fsS "${CAMOFOX_BASE_URL:-http://127.0.0.1:9377}/health"
```

2. OpenClaw plugin mode, if applicable:

```bash
openclaw camofox status
```

3. Raw REST mode: create a tab, snapshot it, close it.

4. Hermes mode: verify `CAMOFOX_URL` is absent from global Hermes env and present only on the dedicated cloaked Hermes process, if Hermes process routing is intentionally being used.

5. Cleanup: close tabs or stop the managed server if the task started it only for one job.

## Common mistakes

- Using Camofox when normal search/extract tools are cheaper and sufficient.
- Forgetting `userId` in raw REST calls.
- Clicking stale refs after navigation or DOM changes.
- Setting/exporting `CAMOFOX_URL` globally in Hermes (`~/.hermes/.env`, gateway env, shell profile, service env) and accidentally routing all browser calls through Camofox.
- Assuming `CAMOFOX_API_KEY` is needed for normal browser work; it is only for sensitive endpoints such as cookie import/traces.
- Exposing a no-auth local development server beyond loopback.
- Treating MCP tool names from a separate MCP wrapper as if they are the upstream OpenClaw plugin tools. The MCP wrapper may expose many more tools; this skill is centered on upstream `@askjo/camofox-browser` plus its REST API.

## Output format when using this skill

Report:

- runtime: OpenClaw plugin, raw REST, Hermes, or other
- base URL: server URL used
- startup: npx/package, cloned repo npm start, OpenClaw autoStart, or existing server
- auth: no auth/local loopback, `CAMOFOX_ACCESS_KEY`, or `CAMOFOX_API_KEY` for sensitive endpoints
- user/session: `userId`, `sessionKey`
- tab: `tabId` used/closed/preserved
- verification: health + snapshot/action result summary
- cleanup: tabs closed and whether server was left running
