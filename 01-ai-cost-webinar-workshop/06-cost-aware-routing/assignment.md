---
slug: cost-aware-routing
id: cm1oljriphtl
type: challenge
title: Cost-Aware Routing
teaser: Cheap model by default; premium only when a request explicitly escalates.
notes:
- type: text
  contents: "# \U0001F500 Cost-Aware Routing\n\nThe cheapest token is the one you
    never send to an expensive model. Default\neveryone to a small model and let the
    gateway promote to a premium model\n**only when a request opts in** — no client
    gets to silently pick gpt-4o.\n"
tabs:
- id: 95kjhyqqhydb
  title: Terminal
  type: terminal
  hostname: server
- id: 2wksoymhy5vx
  title: Editor
  type: code
  hostname: server
  path: /root
- id: mjpcpyg1jn2w
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

In the last challenge you pinned *everyone* to `gpt-4o-mini`. But some requests
genuinely need a frontier model. Instead of trusting clients to choose (and pay)
responsibly, make the **gateway** decide: cheap by default, premium only when a
request carries an explicit `x-priority: high` header.

## Step 1 — Two models, two price points

Give the `llm` block **two models**: a premium one that only matches when the
request carries `x-priority: high`, and a catch-all default. Paste this:

```bash
cat > /root/config.yaml <<'EOF'
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:///root/data/data.db"
  modelCatalog:
    - file: /root/costs/base-costs.json
llm:
  port: 4000
  policies:
    localRateLimit:
    - { maxTokens: 200000, tokensPerFill: 200000, fillInterval: "1h", type: tokens }
  models:
  - name: "openai/*"
    matches:
    - headers:
      - name: x-priority
        value: { exact: high }
    provider: openAI
    params: { model: gpt-4o, apiKey: "$OPENAI_API_KEY" }
  - name: "openai/*"
    provider: openAI
    params: { model: gpt-4o-mini, apiKey: "$OPENAI_API_KEY" }
frontendPolicies:
  http:
    maxBufferSize: 33554432
EOF
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

Order matters: the **premium** model (more specific — it requires the header) is
listed first, so it's matched before the catch-all default.

## Step 2 — Prove it

Default traffic → cheap model:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"anything","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns **`gpt-4o-mini`**. Now escalate with the header:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'x-priority: high' -H 'Content-Type: application/json' \
  -d '{"model":"anything","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns **`gpt-4o`**. Same endpoint, same client code — the **gateway** decided
the price tier.

## Step 3 — Why this saves real money

Most traffic is routine (summaries, classification, simple Q&A) and runs fine on
a small model at ~1/17th the cost. Reserve the frontier model for the few
requests that truly need it. Your apps set `x-priority: high` only on those — and
they physically *cannot* reach gpt-4o any other way.

> Next: tag and slice that spend so you can see exactly where it lands. ➡️
