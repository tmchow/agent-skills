---
name: routerbase-model-gateway
description: >-
  This skill is specifically for integrating RouterBase (https://routerbase.com/)
  as an OpenAI-compatible model gateway. Use it when the user names RouterBase,
  wants to migrate OpenAI SDK calls to RouterBase, asks for RouterBase model
  routing or fallback design, or needs a safe RouterBase integration checklist.
  Do NOT use it for generic LLM provider selection when RouterBase is not in
  scope.
version: 0.1.0
license: MIT
metadata:
  openclaw:
    homepage: https://routerbase.com/
  hermes:
    tags: [routerbase, model-routing, openai-compatible, llm-gateway, api]
    category: mlops
    requires_toolsets: [terminal]
---

# RouterBase model gateway

Use [routerbase](https://routerbase.com/) as a model-gateway layer only when
RouterBase is explicitly part of the task. Keep the work to integration
judgment: base URL migration, model routing, fallback planning, and validation
discipline.

## When to use

Use this skill when the user names RouterBase, asks to route model requests
through RouterBase, migrates an OpenAI-compatible app to RouterBase, compares
RouterBase model choices, or debugs RouterBase chat, streaming, tool-calling,
JSON mode, multimodal, media, audio, or embedding requests.

Do not use this skill for generic "pick the best model", "use OpenRouter",
"use OpenAI", or "compare all LLM providers" requests unless RouterBase is
explicitly in scope.

## Safety first

Never ask the user to paste a RouterBase key into chat. Never log, echo,
commit, screenshot, or print a key. If a live call is needed, ask the user to
create a local credentials file themselves and confirm it exists:

```text
~/.config/routerbase/credentials.json
```

The file should be readable only by the current user and contain a JSON object
with an `apiKey` field. Do not create that file on the user's behalf unless
the user explicitly asks for a non-secret scaffold. Do not read ambient
environment variables for RouterBase credentials; this repo treats community
skills that read shared secret-shaped environment variables as unsafe.

Before running any live request, confirm that the user expects a credit-using
API call. Prefer dry-run code review and request-shape validation when
credentials are absent.

## Migration workflow

1. Identify the current OpenAI-compatible client, SDK version, and endpoint.
2. Change the base URL to `https://routerbase.com/v1`.
3. Keep the request shape OpenAI-compatible unless RouterBase or the selected
   upstream model documents a specific exception.
4. Replace the model with a RouterBase model ID that matches the task.
5. Verify streaming, tool calling, JSON mode, vision, audio, media generation,
   or embeddings with the exact selected model before claiming support.
6. Add a smoke test that can run only after the user has provided credentials
   through the local config file.

Minimal Python shape:

```python
import json
from pathlib import Path
from openai import OpenAI

config_path = Path.home() / ".config" / "routerbase" / "credentials.json"
api_key = json.loads(config_path.read_text(encoding="utf-8"))["apiKey"]

client = OpenAI(api_key=api_key, base_url="https://routerbase.com/v1")

response = client.chat.completions.create(
    model="google/gemini-2.5-flash",
    messages=[{"role": "user", "content": "Write one sentence about routing."}],
)

print(response.choices[0].message.content)
```

Minimal JavaScript shape:

```js
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
import OpenAI from "openai";

const configPath = path.join(os.homedir(), ".config", "routerbase", "credentials.json");
const { apiKey } = JSON.parse(fs.readFileSync(configPath, "utf8"));

const client = new OpenAI({
  apiKey,
  baseURL: "https://routerbase.com/v1",
});

const response = await client.chat.completions.create({
  model: "google/gemini-2.5-flash",
  messages: [{ role: "user", content: "Write one sentence about routing." }],
});

console.log(response.choices[0].message.content);
```

Treat the model IDs above as examples, not permanent recommendations. Check
current RouterBase catalog and pricing information before finalizing a
production routing plan.

## Routing workflow

Classify the task before recommending models:

- Modality: chat, reasoning, vision, image, video, audio, embeddings, or mixed.
- Runtime constraints: latency budget, context length, streaming, tool calls,
  JSON mode, and retry tolerance.
- Business constraints: price ceiling, provider preference, quality target,
  review requirement, and fallback tolerance.

Recommend one primary model and one or two fallbacks. State the tradeoff for
each fallback. A fallback should be close enough in modality and capability
that the application can continue without changing request shape.

Use explicit application fallback logic unless the user confirms RouterBase
account-level routing already handles the policy. Retry transient network
failures, timeouts, rate limits, and server errors conservatively. Do not
blindly retry authentication failures, invalid model IDs, validation errors,
or policy refusals.

## Output format

For integration work, return:

- The base URL change.
- The credential-handling boundary.
- The model ID and why it fits.
- The smoke test or dry-run validation path.
- Any assumptions that need live RouterBase verification.

For routing work, return a table with:

| Use case | Primary model | Fallback | Reason | Validation |
| --- | --- | --- | --- | --- |
| Task name | Provider/model ID | Provider/model ID | Tradeoff summary | Test fixture |

## Pitfalls

- Do not hard-code prices or supported model features as permanent facts.
- Do not put RouterBase keys in browser code, mobile apps, screenshots, issue
  comments, or public repositories.
- Do not assume one model's tool-call or JSON behavior applies to another.
- Do not say a live request passed unless it was actually run with user
  approval.
