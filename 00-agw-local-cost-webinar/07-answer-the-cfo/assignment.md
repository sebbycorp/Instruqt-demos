---
slug: answer-the-cfo
id: fsyfypm5crfd
type: challenge
title: Answer the CFO
teaser: MCP tool schemas are tokens too. See the whole bill — LLM and tools — in one
  query.
notes:
- type: text
  contents: |
    The full stack is running: cost-aware LLM routing on :4000 and your MCP tool server on
    :3000, one gateway. Agents don't just call models — every MCP tool's name and schema
    rides along as input tokens on each turn. Because that traffic is proxied too, it's as
    visible as your LLM calls. This final lab makes the MCP cost concrete, then answers the
    question every CFO asks.
tabs:
- id: 3p0pohxt46gj
  title: Terminal
  type: terminal
  hostname: server
- id: 15yy4mj7jq7z
  title: Editor
  type: code
  hostname: server
  path: /root
- id: 4eoj1kymic2y
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Answer the CFO

## Step 1 — Your tools are tokens (list them)

```bash
SID=$(curl -s -D - -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')

curl -s -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -s -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | grep '^data:' | sed 's/^data: //' | jq '{tools: [.result.tools[].name]}'
```

Every tool name, description, and schema there is injected into the model's context on
**each** turn — now visible at the gateway instead of hiding in your bill.

## Step 2 — Answer the CFO

One query, the whole bill — total spend, spend by model, call counts:

```bash
sqlite3 -box /root/data/data.db \
"SELECT gen_ai_request_model AS model,
        COUNT(*)            AS calls,
        SUM(input_tokens)   AS in_tok,
        SUM(output_tokens)  AS out_tok,
        printf('\$%.6f', SUM(cost)) AS spend_usd
 FROM request_logs
 GROUP BY gen_ai_request_model
 ORDER BY SUM(cost) DESC;"

echo '--- grand total ---'
sqlite3 /root/data/data.db \
"SELECT printf('Total spend: \$%.6f across %d calls', SUM(cost), COUNT(*)) FROM request_logs;"
```

That's the answer to "what did our AI cost?" — by model, in real time, from a control
plane you stood up in Docker. Click **Check** to finish. 🎉

## What you built

- Agentgateway OSS in **Docker**, fronting OpenAI **and** your own MCP server.
- **Per-request cost** in a queryable log and a UI — no month-end surprise.
- **Your own MCP server** in a few lines of Python, proxied by the gateway.
- **A client app** using the standard OpenAI SDK + MCP client through one endpoint.
- **Token budgets** that return 429, and **cost-aware routing** by header.
