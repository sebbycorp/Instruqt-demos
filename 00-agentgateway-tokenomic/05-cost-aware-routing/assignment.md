---
slug: cost-aware-routing
id: 56m9nfd7erhf
type: challenge
title: Cost-Aware Routing
teaser: Cheap model by default; premium only when a request explicitly escalates.
notes:
- type: text
  contents: "# \U0001F500 Cost-Aware Routing\n\nThe cheapest token is the one you
    never send to an expensive model. Default\neveryone to a small model and let the
    gateway promote to a premium model **only\nwhen a request opts in** — no client
    gets to silently pick gpt-4.1.\n"
tabs:
- id: fuezxe4ceolm
  title: Terminal
  type: terminal
  hostname: server
- id: kdd0dvnowmto
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: 2qaps6kex9wz
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Cost-Aware Routing

![Cost-aware routing: cheap default, premium on x-priority](../assets/diagram-routing.png)

You've capped spend; now route smartly. Some requests genuinely need a frontier
model — but instead of trusting clients to choose (and pay) responsibly, make the
**gateway** decide: cheap by default, premium only when a request carries an explicit
`x-priority: high` header.

## Step 1 — Two models, two price points

Give the `llm` block **two models**: a premium one that only matches when the request
carries `x-priority: high`, and a catch-all default. In the **Editor**, set
`/root/agentgateway/config.yaml` to:

```yaml
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:////config/data/data.db"
  modelCatalog:
    - file: /config/costs/base-costs.json
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models:
  - name: "openai/*"
    matches:
    - headers:
      - name: x-priority
        value: { exact: high }
    provider: openAI
    params: { model: gpt-4.1, apiKey: "$OPENAI_API_KEY" }
  - name: "openai/*"
    provider: openAI
    params: { model: gpt-4.1-nano, apiKey: "$OPENAI_API_KEY" }
frontendPolicies:
  http:
    maxBufferSize: 33554432
```

```bash
docker run --rm -v /root/agentgateway:/config -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml --validate-only
docker restart agentgateway
```

Order matters: the **premium** model (more specific — it requires the header) is
listed first, so it's matched before the catch-all default.

## Step 2 — Prove it

The client asks for the **same** model both times (`openai/gpt-4.1-nano`). Only the
header changes. Default traffic → cheap model:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns **`gpt-4.1-nano`**. Now escalate with the header:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'x-priority: high' -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns **`gpt-4.1`**. Same endpoint, same request body — the **gateway** decided the
price tier from the header, overriding the model the client asked for.

## Step 3 — Why this saves real money

Most traffic is routine and runs fine on a small model at ~1/17th the cost. Reserve
the frontier model for the few requests that truly need it: your apps set
`x-priority: high` only on those — and they physically *cannot* reach gpt-4.1 any
other way.

> Next: bring **tool traffic** under the same gateway with a separate MCP server. ➡️
