---
slug: governance-budgets-models
id: g0j34nzxlg1d
type: challenge
title: Governance — Budgets & Approved Models
teaser: Enforce a token budget and pin traffic to an approved model. Stop the runaway
  spender.
notes:
- type: text
  contents: "# \U0001F6E1️ Governance\n\nVisibility tells you what happened. **Policy**
    stops it from happening. Two\npolicies — a token budget and an approved-model
    pin — and the surprise\nbill can't repeat the same way.\n"
tabs:
- id: bqrrq2hu93rs
  title: Terminal
  type: terminal
  hostname: server
- id: opmqddlpjloy
  title: Editor
  type: code
  hostname: server
  path: /root
- id: vtk7nyah145i
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Governance — Budgets & Approved Models

Now that every call is metered, attach guardrails to the `llm` block.

## Part A — A token budget (the spend cap)

In the **Editor**, add a `policies` block with `localRateLimit` under `llm:` in
`/root/config.yaml`:

```yaml
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  policies:                 # <-- add this block
    localRateLimit:
    - maxTokens: 50
      tokensPerFill: 50
      fillInterval: "1h"
      type: tokens
  models:
  - name: "openai/*"
    provider: openAI
    params:
      apiKey: "$OPENAI_API_KEY"
```

`type: tokens` counts **LLM tokens**, not requests. We use a tiny 50-token
bucket so you can trip it in one call (in production this would be, say, 2M
tokens/hour per team).

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

Now spend the budget:

```bash
for i in 1 2 3 4; do
  echo "req $i -> $(curl -s -o /dev/null -w '%{http_code}' http://localhost:4000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"Write a paragraph about budgets."}],"max_tokens":80}')"
done
```

The first call returns **200**; the rest return **429 Too Many Requests**. The
runaway agent is throttled — *before* it runs up the bill.

## Part B — Pin to an approved model (the model lever)

A frontier model costs ~17× more. Force everything to the approved cheap model,
no matter what the client asks for. Add `params.model` to your model:

```yaml
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4o-mini      # <-- pin the OUTGOING model, ignore what's requested
      apiKey: "$OPENAI_API_KEY"
```

```bash
agw-restart
```

Now ask for the **expensive** model and watch what actually runs:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4o","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

The response model comes back **`gpt-4o-mini`** — the expensive model is simply
unreachable. (The full `solve` config combines both policies; your edits match it.)

## Part C — How this scales

A 50-token bucket is a demo. In production you enforce budgets **per API key or
per team** with `remoteRateLimit` descriptors backed by a shared rate-limit
service — so `sales` and `eng` each get their own cap across every gateway
replica. Same idea, distributed.

> Two policies, and the runaway bill is structurally prevented. Next: stop
> over-paying by **routing cheap by default**. ➡️
