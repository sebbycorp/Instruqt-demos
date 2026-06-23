---
slug: cost-aware-routing
id: ""
type: challenge
title: Cost-Aware Routing
teaser: Cheap model by default; premium only when a request explicitly escalates.
notes:
- type: text
  contents: |
    # 🔀 Cost-Aware Routing

    The cheapest token is the one you never send to an expensive model. Default
    everyone to a small model and let the gateway promote to a premium model
    **only when a request opts in** — no client gets to silently pick gpt-4o.
tabs:
- id: ""
  title: Terminal
  type: terminal
  hostname: server
- id: ""
  title: Editor
  type: code
  hostname: server
  path: /root
- id: ""
  title: Agentgateway UI
  type: service
  hostname: server
  port: 15000
  path: /ui
difficulty: ""
enhanced_loading: null
---

# Cost-Aware Routing

In the last challenge you pinned *everyone* to `gpt-4o-mini`. But some requests
genuinely need a frontier model. Instead of trusting clients to choose (and pay)
responsibly, make the **gateway** decide: cheap by default, premium only when a
request carries an explicit `x-priority: high` header.

## Step 1 — Two routes, two price points

Replace the single route in `/root/config.yaml` with two — a header-matched
**premium** route and a catch-all **default** route. Paste this into the
**Terminal**:

```bash
cat > /root/config.yaml <<'EOF'
config:
  adminAddr: "0.0.0.0:15000"
  modelCatalog:
  - file: /root/costs/catalog.json
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: premium-route
      matches:
      - path: { pathPrefix: /v1 }
        headers:
        - name: x-priority
          value: { exact: high }
      policies:
        backendAuth: { key: "$OPENAI_API_KEY" }
        ai: { overrides: { model: gpt-4o } }
      backends: [{ ai: { name: openai, provider: { openAI: {} } } }]
    - name: default-route
      matches:
      - path: { pathPrefix: /v1 }
      policies:
        backendAuth: { key: "$OPENAI_API_KEY" }
        ai: { overrides: { model: gpt-4o-mini } }
      backends: [{ ai: { name: openai, provider: { openAI: {} } } }]
EOF
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

Route order matters: the **premium** route (more specific — it requires the
header) is listed first, so it's matched before the catch-all default.

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
