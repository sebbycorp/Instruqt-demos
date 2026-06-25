---
slug: govern-the-spend
id: obyekbrijesh
type: challenge
title: Govern the Spend
teaser: A budget you can't enforce isn't a budget. Cap tokens at the gateway.
notes:
- type: text
  contents: "# \U0001F6E1️ Govern the Spend\n\nSeeing cost is half the job; the other
    half is stopping runaway spend before it\nhappens. The gateway can enforce a token
    budget per time window — requests over\nbudget get a clean HTTP 429 instead of
    a surprise invoice.\n"
tabs:
- id: fgdad03bytjn
  title: Terminal
  type: terminal
  hostname: server
- id: pejgien7jdex
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: oge0n6qbz9py
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
`/root/agentgateway/config.yaml` (alongside `cors`):

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
fails fast with **429**. (A real budget would be far larger; 300 keeps the demo quick.)

## Step 2 — Validate and reload

```bash
docker run --rm -v /root/agentgateway:/config -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml --validate-only
docker restart agentgateway
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

Watch the `200`s flip to `429`s. That 429 is the gateway refusing to let spend run
away.

> In production you'd scope this **per team or per virtual key** (you'll add per-team
> keys at the end of the track) — same mechanism, finer granularity.

> Next: instead of capping everyone, route cheap by default and pay for the premium
> model only when a request explicitly asks for it. ➡️
