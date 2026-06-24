---
slug: govern-spend
id: aglc03govern
type: challenge
title: Govern the Spend
teaser: A budget you can't enforce isn't a budget. Cap tokens and pin the model.
notes:
- type: text
  contents: |
    Seeing cost is half the job; the other half is stopping runaway spend before it
    happens. The gateway can enforce a **token budget** per time window and **pin** the
    model so nobody can quietly switch to an expensive one. Requests over budget get a
    clean HTTP 429 instead of a surprise invoice.
tabs:
- id: aglc03-term
  title: Terminal
  type: terminal
  hostname: server
- id: aglc03-edit
  title: Editor
  type: code
  hostname: server
  path: /root
- id: aglc03-ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Govern the Spend

Your model is already pinned to the cheap `gpt-4.1-nano` (the `openai/*` route forces
`params.model: gpt-4.1-nano`, so a client asking for `gpt-4.1` still gets nano). Now add
a hard token budget.

## Step 1 — Add a token budget

In the **Editor**, open `/root/config.yaml` and add a `localRateLimit` to the existing
`llm.policies` block (alongside `cors`). This caps total tokens per hour:

```yaml
llm:
  port: 4000
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

`type: tokens` counts **input + output** tokens. Token counts are only known once a
request completes, so the cap applies to the *next* request after the budget is spent —
that request fails fast with **429**.

## Step 2 — Apply it

```bash
agw-validate
agw-restart
```

## Step 3 — Spend past the budget

Fire several requests in a row. The first one or two succeed (`200`); once ~300 tokens
are gone, the rest are rejected with `429`:

```bash
for i in $(seq 1 8); do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Write a detailed paragraph about cloud cost governance."}],"max_tokens":120}')
  echo "request $i -> $code"
done
```

You should see `200`s flip to `429`s. That 429 is the gateway refusing to let spend run
away. Click **Check** once the budget is enforcing.

> In production you'd set this per team or per virtual key — but the mechanism is exactly
> what you just configured.
