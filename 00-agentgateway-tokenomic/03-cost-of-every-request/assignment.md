---
slug: cost-of-every-request
id: 61ctkrsshamm
type: challenge
title: The Cost of Every Request
teaser: See the real USD cost of every call — and why gpt-4.1 costs ~20× gpt-4.1-nano.
notes:
- type: text
  contents: "# \U0001F4B5 The Cost of Every Request\n\nYou didn't *configure* this
    — the moment the gateway came up it started pricing\nevery call in real USD and
    recording it. In this challenge you'll watch a single\nrequest become a dollar
    figure, and see why choosing gpt-4.1 over gpt-4.1-nano is\na budget decision,
    not a detail.\n"
tabs:
- id: pfdjffahct5v
  title: Terminal
  type: terminal
  hostname: server
- id: iay834bqhgxq
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: ovcfnorshoel
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# The Cost of Every Request

Cost tracking is already on. Your config loaded the default **`base-costs.json`**
catalog (USD per 1M tokens) and the request database, so the gateway is already
pricing and recording every call. This challenge is about *seeing* it.

![Two models, two price points: the model name picks the tier; the gateway prices every call](../assets/diagram-cost-models.png)

## Step 1 — Look at the price list

```bash
grep -A4 '"gpt-4.1-nano"\|"gpt-4.1"' /root/agentgateway/costs/base-costs.json | head
```

Rates are **USD per 1M tokens**, split into input and output. Note `gpt-4.1`
($2.00 / $8.00) costs **~20×** `gpt-4.1-nano` ($0.10 / $0.40).

## Step 2 — Offer both a premium and a cheap model

To compare costs, the gateway needs to serve **both**. Add a premium model matched by
name (`openai/gpt-4.1` → real `gpt-4.1`) alongside the cheap catch-all (`openai/*` →
`gpt-4.1-nano`). In the **Editor**, set `/root/agentgateway/config.yaml` to:

```yaml
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:////config/data/data.db"
  modelCatalog:
    - file: /config/costs/base-costs.json
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models:
  - name: "openai/gpt-4.1"                # asked for by name → real gpt-4.1
    provider: openAI
    params: { model: gpt-4.1, apiKey: "$OPENAI_API_KEY" }
  - name: "openai/*"                       # everything else → cheap nano
    provider: openAI
    params: { model: gpt-4.1-nano, apiKey: "$OPENAI_API_KEY" }
frontendPolicies:
  http:
    maxBufferSize: 33554432
```

Validate and reload:

```bash
docker run --rm -v /root/agentgateway:/config -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml --validate-only
docker restart agentgateway
```

The gateway picks the model by **matching the requested name**: `openai/gpt-4.1` hits
the specific entry; anything else falls through to the `openai/*` default.

## Step 3 — Drive a cheap call and a premium call

```bash
# cheap model
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' | jq -r '.choices[0].message.content'

# premium model
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' | jq -r '.choices[0].message.content'
```

## Step 4 — See the cost, cleanly

Every call is priced and written to the external SQLite database. Pull the last few:

```bash
sqlite3 -box /root/agentgateway/data/data.db \
  "SELECT gen_ai_request_model AS model, input_tokens AS in_tok, output_tokens AS out_tok,
          printf('\$%.7f', cost) AS cost_usd
   FROM request_logs ORDER BY completed_at DESC LIMIT 3;"
```

**A clean dollar figure on every call** — and the model column shows what *actually
ran* (the gateway logs the served model, not the name you typed).

## Step 5 — The teaching moment: same answer, 20× the price

```bash
sqlite3 -box /root/agentgateway/data/data.db \
  "SELECT gen_ai_request_model AS model, printf('\$%.6f', MAX(cost)) AS per_call,
          printf('\$%.2f', MAX(cost)*1000000) AS per_1M_calls
   FROM request_logs
   WHERE gen_ai_request_model IN ('gpt-4.1','gpt-4.1-nano')
   GROUP BY gen_ai_request_model ORDER BY MAX(cost) DESC;"
```

Same trivial answer — but roughly **$3–4 vs ~$68 per million calls**. That ~20× gap
is exactly why "which model?" is a budget decision.

## Step 6 — Correct the price with an override

List price isn't *your* price. Agentgateway lets you **layer catalogs** — later files
win — so you override only what you need. Write a negotiated `gpt-4.1` rate (40% off)
and layer it on top of the base:

```bash
cat > /root/agentgateway/costs/overrides.json <<'EOF'
{ "providers": { "openai": { "models": {
  "gpt-4.1": { "rates": { "input": "1.20", "output": "4.80" } } } } } }
EOF

# add the overrides file AFTER the base in modelCatalog (later wins)
sed -i 's#    - file: /config/costs/base-costs.json#    - file: /config/costs/base-costs.json\n    - file: /config/costs/overrides.json#' /root/agentgateway/config.yaml

docker run --rm -v /root/agentgateway:/config -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml --validate-only
docker restart agentgateway
```

Re-run the premium call, then compare its cost to the earlier one:

```bash
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1","messages":[{"role":"user","content":"Say hi in 3 words."}],"max_tokens":20}' >/dev/null

sqlite3 -box /root/agentgateway/data/data.db \
  "SELECT gen_ai_request_model AS model, printf('\$%.2f', cost*1000000) AS per_1M_calls
   FROM request_logs WHERE gen_ai_request_model='gpt-4.1' ORDER BY completed_at DESC LIMIT 2;"
```

Two `gpt-4.1` rows, newest first: the **top** is your just-made call at the negotiated
rate (~$40.80/1M), the **bottom** an earlier call at list price (~$68.00/1M). Same
model, same prompt — the override cut the price ~40% with no base-catalog edit.

## Step 7 — See it as a chart

Open the **Agentgateway UI** tab → left nav → **Costs**. The same numbers you just
queried, as **charts** — spend by model, no SQL required. Re-run a curl from Step 3
and refresh to watch spend tick up. This is the view you'd hand a manager.

> Now you can answer *"how much did that call cost?"* Next: cap runaway spend with a
> token budget. ➡️
