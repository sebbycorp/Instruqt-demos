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
    stops it from happening. Two\nlevers — a token budget and an approved-model pin
    — and the surprise bill\ncan't repeat the same way.\n"
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

Visibility told you *what* happened. **Policy** stops it from happening. You'll
attach two levers to the `llm` block, then watch each one work.

## Set the config

In the **Editor**, make your `llm` block look like this — one `policies:` block
holding both `cors` and a `localRateLimit`, and a `params.model` pin on the model.
Paste it over the `llm:` section of `/root/config.yaml`:

```yaml
llm:
  port: 4000
  policies:
    cors:                       # allow cross-origin browser/API clients
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
    localRateLimit:             # lever 1 — the spend cap
    - maxTokens: 50
      tokensPerFill: 50
      fillInterval: "1h"
      type: tokens
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano       # lever 2 — pin every request to the approved model
      apiKey: "$OPENAI_API_KEY"
```

> ⚠️ Note there's a **single** `policies:` key with `cors` and `localRateLimit`
> nested under it — not two `policies:` keys.

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

---

## Lever 1 — Approved-model pin

- **What:** `params.model` forces every request to one model, ignoring whatever
  model the client asked for.
- **Why:** the frontier model (`gpt-4.1`) is **~20× pricier** than `gpt-4.1-nano`.
  Pinning removes that line item entirely — no app or agent can opt into the
  expensive model, even by accident.
- **See it:** ask for the *expensive* model and watch what actually runs:

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns **`gpt-4.1-nano`** — the expensive model is simply unreachable.

---

## Lever 2 — Token budget

- **What:** `localRateLimit` with `type: tokens` is a bucket of N tokens per
  window. When it's empty, further requests get **429 Too Many Requests**.
- **Why:** a runaway agent or a stuck batch job gets throttled *before* it runs
  up the bill — the #1 cause of surprise spend. (We use a tiny 50-token bucket so
  you can trip it in one call; in production this might be 2M tokens/hour.)
- **See it:** spend the budget:

```bash
for i in 1 2 3 4; do
  echo "req $i -> $(curl -s -o /dev/null -w '%{http_code}' http://localhost:4000/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Write a paragraph about budgets."}],"max_tokens":80}')"
done
```

The first call returns **200**; the rest return **429**. Throttled before the bill.

---

## How this scales in production

- **What:** budgets **per API key or per team**, not one global bucket.
- **Why:** `sales` and `eng` each need their own cap — and it has to hold across
  every gateway replica, not just one process.
- **How:** swap `localRateLimit` for **`remoteRateLimit`** with descriptors (keyed
  on the virtual key or a team header), backed by a shared rate-limit service.
  Same idea you just saw, enforced cluster-wide.

> Two levers, and the runaway bill is structurally prevented. Next: stop
> *over-paying* by **routing cheap by default**. ➡️
