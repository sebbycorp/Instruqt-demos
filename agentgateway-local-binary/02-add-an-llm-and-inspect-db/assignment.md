---
slug: add-an-llm-and-inspect-db
id: 0ijxtb9p8jy5
type: challenge
title: Add an LLM & Inspect the SQLite DB
teaser: Route a model through the gateway, then read the request back out of SQLite.
notes:
- type: text
  contents: |
    # 🌐 Add an LLM & Inspect the Database

    Point the gateway at OpenAI, send a call, then open the external SQLite file
    directly to see the request the gateway logged.
tabs:
- id: term-llm02
  title: Terminal
  type: terminal
  hostname: server
- id: edit-llm02
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: ui-llm02
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Add an LLM & Inspect the SQLite DB

## Step 1 — Add an OpenAI model

In the **Editor**, open `/root/agentgateway/config.yaml` and replace the empty
`models: []` with:

```yaml
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
```

`$OPENAI_API_KEY` is read from the environment — clients never see the real key.

## Step 2 — Validate and reload the gateway

There's no daemon manager here — reload = stop the process and start it again. Real
commands, no aliases:

```bash
agentgateway -f /root/agentgateway/config.yaml --validate-only

pkill -f "agentgateway -f /root/agentgateway/config.yaml"
sleep 1
nohup agentgateway -f /root/agentgateway/config.yaml > /root/agentgateway/agw.log 2>&1 &
sleep 4
```

## Step 3 — Send a call through the gateway

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' \
  | jq -r '.choices[0].message.content'
```

## Step 4 — Read it back out of the external SQLite DB

The gateway logged that call to `/data/data.db`. Query the file directly:

```bash
# row count
sqlite3 /data/data.db "SELECT COUNT(*) AS requests FROM request_logs;"

# the most recent call: model, status, tokens, cost
sqlite3 -header -column /data/data.db \
 "SELECT gen_ai_request_model AS model, http_status AS status,
         total_tokens AS tokens, printf('\$%.6f', cost) AS cost
  FROM request_logs ORDER BY completed_at DESC LIMIT 3;"
```

**What you'll see:** your `gpt-4.1-nano` call with a `200`, token counts, and a realized
USD cost — read straight from the external SQLite database the gateway is writing to.
Because the DB is just a file at `/data/data.db`, you can point any SQLite tool at it.

> 🎉 You ran Agentgateway as a plain binary, backed it with an external SQLite
> database, routed an LLM through it, and inspected the logged request directly in the
> DB — no Docker, no Kubernetes.
