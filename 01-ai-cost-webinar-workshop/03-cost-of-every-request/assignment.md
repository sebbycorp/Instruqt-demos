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

## Step 2 — Drive a call through the OpenAI CLI

Point the official `openai` CLI at the gateway and send a chat request:

```bash
export OPENAI_BASE_URL=http://localhost:4000/v1
openai api chat.completions.create -m gpt-4o-mini -g user "Say hi in 3 words."
```

You'll see the model's short reply printed (e.g. `Hello there, friend!`) — that's
the call going through **your** gateway to OpenAI and back. (New terminals get
`OPENAI_BASE_URL` automatically; we export it here so the current shell uses it.)

## Step 3 — See the cost in the gateway log

```bash
grep -oE 'gen_ai.usage.input_tokens=[^ ]+|gen_ai.usage.output_tokens=[^ ]+|agw.ai.usage.cost.total=[^ ]+' /root/agentgateway.log | tail -3
```

You'll see something like
`gen_ai.usage.input_tokens=14 gen_ai.usage.output_tokens=5 agw.ai.usage.cost.total=0.0000051`.
**That is a dollar figure on a single call** — something the provider dashboard
will never give you.

That same call was also **written to the database**:

```bash
sqlite3 -box /root/data/data.db "SELECT gen_ai_request_model, input_tokens, output_tokens, cost FROM request_logs ORDER BY completed_at DESC LIMIT 3;"
```

Every request the gateway handles is now a row you can query — we'll come back to
this in the final challenge.

## Step 4 — The teaching moment: same answer, 17× the price

```bash
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' >/dev/null
grep 'agw.ai.usage.cost.total' /root/agentgateway.log | tail -2
```

Compare the two `cost.total` values: roughly `$0.0000051` (mini) vs `$0.000085`
(gpt-4o) for the *same* trivial answer. At a million calls a day, that gap is the
difference between a $5 bill and an $85 bill.

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

> Now you can answer *"how much did that call cost?"* Next: how to **operate and
> troubleshoot** this gateway. ➡️
