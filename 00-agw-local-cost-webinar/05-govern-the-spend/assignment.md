---
slug: govern-the-spend
id: 2wex6skwjybv
type: challenge
title: Govern the Spend
teaser: A budget you can't enforce isn't a budget. Cap tokens at the gateway.
notes:
- type: text
  contents: |
    Seeing cost is half the job; the other half is stopping runaway spend before it
    happens. The gateway can enforce a token budget per time window — requests over
    budget get a clean HTTP 429 instead of a surprise invoice.
tabs:
- id: wxx1qbgl5ixl
  title: Terminal
  type: terminal
  hostname: server
- id: nrgp69fqh7nf
  title: Editor
  type: code
  hostname: server
  path: /root
- id: bmoifxngn5kg
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Govern the Spend

## Step 1 — Add a token budget

In the **Editor**, add a `localRateLimit` to the existing `llm.policies` block in
`/root/config.yaml` (alongside `cors`):

```yaml
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
    localRateLimit:
    - maxTokens: 300
      tokensPerFill: 300
      fillInterval: 1h
      type: tokens
```

`type: tokens` counts **input + output** tokens. Counts are known only once a request
completes, so the cap applies to the *next* request after the budget is spent — which
fails fast with **429**.

## Step 2 — Apply it

```bash
agw-validate
agw-reload
```

## Step 3 — Spend past the budget

```bash
for i in $(seq 1 8); do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Write a detailed paragraph about cloud cost governance."}],"max_tokens":120}')
  echo "request $i -> $code"
done
```

Watch the `200`s flip to `429`s. That 429 is the gateway refusing to let spend run away.
Click **Check** once the budget is enforcing.

> In production you'd scope this per team or per virtual key — same mechanism.
