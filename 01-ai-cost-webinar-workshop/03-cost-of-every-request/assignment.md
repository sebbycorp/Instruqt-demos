---
slug: cost-of-every-request
id: zzrir3nvtmi0
type: challenge
title: The Cost of Every Request
teaser: See the real USD cost of every call — and why gpt-4o costs ~17× gpt-4o-mini.
notes:
- type: text
  contents: "# \U0001F4B5 The Cost of Every Request\n\nHere's the magic: you didn't
    *configure* any of this. The moment the gateway\ncame up, it started pricing every
    call in real USD and recording it.\n\nIn this challenge you'll watch a single
    request become a dollar figure — and\nsee why choosing gpt-4o over gpt-4o-mini
    is a budget decision, not a detail.\n"
tabs:
- id: ujaxcjanrprx
  title: Terminal
  type: terminal
  hostname: server
- id: mtwzbgcxq93i
  title: Editor
  type: code
  hostname: server
  path: /root
- id: vr2m01jsbyyl
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# The Cost of Every Request

Good news: you don't have to *configure* cost tracking — it's already on. Your
config loaded the default **`base-costs.json`** catalog (~846 models, USD per 1M
tokens) and a **request database**, so the gateway is already pricing and
recording every call. This challenge is about *seeing* it.

## Step 1 — Look at the price list

```bash
grep -A4 '"gpt-4o-mini"\|"gpt-4o"' /root/costs/base-costs.json | head
```

Rates are **USD per 1 million tokens**, split into input and output — exactly how
providers bill. Note `gpt-4o` costs ~17× `gpt-4o-mini`.

## Step 2 — Drive a couple of calls

Send a cheap and a premium request through the gateway. Note the **`openai/`**
prefix on the model — that's how the gateway's `openai/*` model matches:

```bash
# cheap model
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' | jq -r '.choices[0].message.content'

# premium model
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4o","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' | jq -r '.choices[0].message.content'
```

Each prints the model's short reply — the calls went through **your** gateway to
OpenAI and back, and were just priced and logged.

## Step 3 — See the cost, cleanly

Every call is priced and written to the database. Pull the last few as a tidy
table (no log-grepping):

```bash
sqlite3 -box /root/data/data.db \
  "SELECT gen_ai_request_model AS model, input_tokens AS in_tok, output_tokens AS out_tok,
          printf('\$%.7f', cost) AS cost_usd
   FROM request_logs ORDER BY completed_at DESC LIMIT 3;"
```

```
┌─────────────┬────────┬─────────┬────────────┐
│ model       │ in_tok │ out_tok │ cost_usd   │
├─────────────┼────────┼─────────┼────────────┤
│ gpt-4o-mini │ 14     │ 5       │ $0.0000051 │
└─────────────┴────────┴─────────┴────────────┘
```

**A clean dollar figure on every single call** — something the provider dashboard
never gives you. (It's in the access log too if you ever need it:
`grep cost /root/agentgateway.log`.)

## Step 4 — The teaching moment: same answer, 17× the price

```bash
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4o","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' >/dev/null

sqlite3 -box /root/data/data.db \
  "SELECT gen_ai_request_model AS model, printf('\$%.6f', cost) AS per_call,
          printf('\$%.2f', cost*1000000) AS per_1M_calls
   FROM request_logs ORDER BY completed_at DESC LIMIT 2;"
```

```
┌─────────────┬───────────┬──────────────┐
│ model       │ per_call  │ per_1M_calls │
├─────────────┼───────────┼──────────────┤
│ gpt-4o      │ $0.000085 │ $85.00       │
│ gpt-4o-mini │ $0.000005 │ $5.10        │
└─────────────┴───────────┴──────────────┘
```

Same trivial answer — **$5 vs $85 per million calls.** That ~17× gap is exactly
why "which model?" is a budget decision, not a detail.

## Step 5 — It's a metric too

```bash
curl -s http://localhost:15020/metrics | grep cost_catalog
```

`agentgateway_cost_catalog_lookups_total{status="Exact",...}` confirms the gateway
priced the request. Point Prometheus/Grafana at `:15020` and cost becomes a
dashboard.

## Step 6 — Watch it in the UI

Open the **Agentgateway UI** tab (`:15000/ui`) and use the **Playground** to send
a chat request, then look at the request view — you'll see the model, token
counts, and cost for each call without touching the logs. This is the same data,
visualized.

> Now you can answer *"how much did that call cost?"* Next: explore the whole
> gateway in its **web UI**. ➡️
