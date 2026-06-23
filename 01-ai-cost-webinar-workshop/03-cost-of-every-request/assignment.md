---
slug: cost-of-every-request
id: zzrir3nvtmi0
type: challenge
title: The Cost of Every Request
teaser: Attach real USD cost and token counts to every call — and watch models differ
  by 17×.
notes:
- type: text
  contents: "# \U0001F4B5 The Cost of Every Request\n\nRouting isn't the point — **attribution**
    is. Give the gateway a price list\nand it stamps every request with token usage
    and **realized USD cost**, in\nreal time. Then you'll see why model choice is
    a budget decision.\n"
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
- id: ""
  title: Agentgateway UI
  type: service
  hostname: server
  port: 15000
  path: /ui
difficulty: ""
enhanced_loading: null
---

# The Cost of Every Request

The gateway already sees every token (OpenAI returns usage on each response).
Give it a **model cost catalog** and it converts tokens → dollars automatically.

## Step 1 — Look at the price list

```bash
cat /root/costs/catalog.json
```

Rates are **USD per 1 million tokens**, split into input and output — exactly how
providers bill. Note `gpt-4o` costs ~17× `gpt-4o-mini`.

## Step 2 — Wire the catalog into the gateway

In the **Editor**, add this block to the **top** of `/root/config.yaml` (above
`binds:`):

```yaml
config:
  modelCatalog:
  - file: /root/costs/catalog.json
```

Validate and restart (the `agw-restart` helper does both):

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

## Step 3 — Drive a call through the OpenAI CLI

The official `openai` CLI is already pointed at the gateway
(`OPENAI_BASE_URL=http://localhost:4000/v1`):

```bash
openai api chat.completions.create -m gpt-4o-mini -g user "Say hi in 3 words."
```

## Step 4 — See the cost in the gateway log

```bash
grep -oE 'gen_ai.usage.input_tokens=[^ ]+|gen_ai.usage.output_tokens=[^ ]+|agw.ai.usage.cost.total=[^ ]+' /root/agentgateway.log | tail -3
```

You'll see something like
`gen_ai.usage.input_tokens=14 gen_ai.usage.output_tokens=5 agw.ai.usage.cost.total=0.0000051`.
**That is a dollar figure on a single call** — something the provider dashboard
will never give you.

## Step 5 — The teaching moment: same answer, 17× the price

```bash
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' >/dev/null
grep 'agw.ai.usage.cost.total' /root/agentgateway.log | tail -2
```

Compare the two `cost.total` values: roughly `$0.0000051` (mini) vs `$0.000085`
(gpt-4o) for the *same* trivial answer. At a million calls a day, that gap is the
difference between a $5 bill and an $85 bill.

## Step 6 — It's a metric too

```bash
curl -s http://localhost:15020/metrics | grep cost_catalog
```

`agentgateway_cost_catalog_lookups_total{status="Exact",...}` confirms the gateway
priced the request. Point Prometheus/Grafana at `:15020` and cost becomes a
dashboard.

## Step 7 — Watch it in the UI

Open the **Agentgateway UI** tab (`:15000/ui`) and use the **Playground** to send
a chat request, then look at the request view — you'll see the model, token
counts, and cost for each call without touching the logs. This is the same data,
visualized.

> Now you can answer *"how much did that call cost?"* Next: how to **operate and
> troubleshoot** this gateway. ➡️
