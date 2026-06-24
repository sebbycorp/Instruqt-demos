---
slug: cost-of-every-request
id: 2bt2uh5urdwm
type: challenge
title: The Cost of Every Request
teaser: Every call now lands in a request log with its exact dollar cost. Go read
  it.
notes:
- type: text
  contents: |
    The gateway is running and fronting OpenAI. Because you loaded a cost catalog and a
    SQLite request log, the gateway prices every single call — input tokens, output
    tokens, and dollars — the instant it completes. No exporter, no spreadsheet. In this
    lab you generate a little traffic and read the cost straight out of the log and the UI.
tabs:
- id: 3oxi5yt11uqs
  title: Terminal
  type: terminal
  hostname: server
- id: lxrjmt1afkbs
  title: Editor
  type: code
  hostname: server
  path: /root
- id: qhzohy4rsn1x
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# The Cost of Every Request

The gateway from Lab 1 is already running (LLM `:4000`, MCP `:3000`, UI `:15000`) and
the request log started fresh for this lab.

## Step 1 — Generate some traffic

Send a handful of completions of different sizes through the gateway:

```bash
for p in "Write a haiku about cost control" \
         "List three benefits of an AI gateway" \
         "Explain token-based pricing in two sentences" \
         "Summarize why MCP traffic costs money"; do
  curl -s http://localhost:4000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d "{\"model\":\"openai/gpt-4.1-nano\",\"messages\":[{\"role\":\"user\",\"content\":\"$p\"}],\"max_tokens\":120}" \
    | jq -r '.choices[0].message.content' >/dev/null && echo "ok: $p"
done
```

## Step 2 — Read the cost out of the log

Each row is one request, priced from the catalog:

```bash
sqlite3 -box /root/data/data.db \
"SELECT gen_ai_request_model AS model,
        input_tokens AS in_tok,
        output_tokens AS out_tok,
        printf('\$%.7f', cost) AS cost_usd
 FROM request_logs
 ORDER BY completed_at DESC
 LIMIT 10;"
```

That `cost_usd` column is computed as
`(input_tokens × input_rate + output_tokens × output_rate) / 1,000,000` using the rates
in `/root/costs/base-costs.json`.

## Step 3 — Total it up

```bash
sqlite3 -box /root/data/data.db \
"SELECT COUNT(*) AS calls,
        SUM(input_tokens)  AS in_tok,
        SUM(output_tokens) AS out_tok,
        printf('\$%.6f', SUM(cost)) AS total_usd
 FROM request_logs;"
```

## Step 4 — See it in the UI

Open the **Agentgateway UI** tab and find the **Costs** / **Requests** view. The same
numbers show up as charts you can hand to a stakeholder.

Click **Check** once you have priced requests in the log.

> The point: the moment traffic flows through the gateway, spend is no longer a
> month-end surprise — it's a column you can query in real time.
