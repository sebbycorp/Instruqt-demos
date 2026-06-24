---
slug: mcp-is-spend-too
id: aglc05mcpcfo
type: challenge
title: MCP Is Spend Too — Answer the CFO
teaser: Tool schemas are tokens. See the hidden MCP cost, then total the whole bill.
notes:
- type: text
  contents: |
    Agents don't just call models — they call tools over MCP, and every tool's name,
    description, and JSON schema is injected into the model's context as input tokens on
    every turn. Tool-heavy agents can quietly double their prompt size. Because your MCP
    server is proxied by the same gateway, that traffic is as visible as your LLM calls.
    This final lab makes the MCP cost concrete, then answers the question every CFO asks.
tabs:
- id: aglc05-term
  title: Terminal
  type: terminal
  hostname: server
- id: aglc05-edit
  title: Editor
  type: code
  hostname: server
  path: /root
- id: aglc05-ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# MCP Is Spend Too — Answer the CFO

The full stack is running: cost-aware LLM routing on `:4000` and the MCP tool server on
`:3000`, all in one gateway.

## Step 1 — List the tools (this is the hidden token cost)

Complete the MCP handshake and ask the server what tools it exposes. Every name and
schema you get back is what rides along in the model's context on each turn:

```bash
# 1) initialize and capture the MCP session id
SID=$(curl -s -D - -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')

# 2) signal initialized, then list the tools
curl -s -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -s -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | grep '^data:' | sed 's/^data: //' \
  | jq '{tool_count: (.result.tools|length), tool_names: [.result.tools[].name]}'
```

`server-everything` exposes a dozen-plus tools — every schema is input tokens on every
agent turn. Now it's visible at the gateway instead of hiding in your bill.

## Step 2 — Call a tool (it flows through the same gateway)

```bash
curl -s -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"echo","arguments":{"message":"MCP is spend too"}}}' \
  | grep '^data:' | sed 's/^data: //' | jq '.result'
```

## Step 3 — Answer the CFO

One query, the whole bill — total spend, spend by model, and call counts:

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

That's the answer to "what did our AI cost?" — broken down by model, in real time, from
a control plane you stood up in Docker in five short labs.

Click **Check** to finish the webinar. 🎉

## What you built

- Agentgateway OSS running **in Docker**, fronting OpenAI **and** an MCP tool server.
- **Per-request cost** in a queryable log and a UI — no month-end surprises.
- **Token budgets** that return 429 before spend runs away.
- **Cost-aware routing** — cheap by default, premium on demand.
- **MCP traffic** under the same lens and the same governance.
