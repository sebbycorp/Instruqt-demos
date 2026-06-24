---
slug: stand-up-the-gateway
id: 1vpt5bunqhfy
type: challenge
title: Stand Up the Gateway in Docker
teaser: Run Agentgateway in a container and put one chokepoint in front of OpenAI
  and MCP.
notes:
- type: text
  contents: |
    Every LLM call and every MCP tool call your apps make is spend. If that traffic
    goes straight to OpenAI, you have no chokepoint to measure or control it. In this
    lab you run Agentgateway OSS **in Docker**, front OpenAI on `:4000` and an MCP tool
    server on `:3000`, and turn on cost tracking — all on this one machine.
tabs:
- id: pzsvlvgy2plp
  title: Terminal
  type: terminal
  hostname: server
- id: xgouifys5oj8
  title: Editor
  type: code
  hostname: server
  path: /root
- id: 5dl3b8e1c8ft
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Stand Up the Gateway in Docker

![All in Docker on one VM: a gateway container fronts OpenAI and an MCP tool server](../assets/diagram-architecture.png)

Your apps talk to OpenAI directly today, so nobody can see or shape the spend.
Let's fix that with a single container in front of everything.

The pieces are already on this box:

- A Docker network `agw-net` and an **MCP tool server** container (`mcp-everything`,
  reachable on the network at `http://mcp-everything:3001/mcp`) are already running.
- The **cost catalog** is at `/root/costs/base-costs.json` (USD per 1M tokens).
- A skeleton config is at `/root/config.yaml`, open in the **Editor** tab.
- Helper commands are loaded in your shell: `agw-validate`, `agw-up`, `agw-restart`,
  `agw-logs`.

You just need to wire the config and start the gateway container.

## Step 1 — Add the OpenAI backend

In the **Editor**, open `/root/config.yaml`. Replace the empty `llm.models` list so
the gateway forwards OpenAI traffic. Anything a client asks for (`openai/...`) maps to
the cheap `gpt-4.1-nano` for now:

```yaml
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
```

## Step 2 — Add the MCP server

Add a top-level `mcp:` block (a sibling of `llm:`) that proxies the tool server on its
own port, `:3000`:

```yaml
mcp:
  port: 3000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      exposeHeaders: ["Mcp-Session-Id"]
  targets:
  - name: everything
    mcp:
      host: http://mcp-everything:3001/mcp
```

The `config.database` and `config.modelCatalog` lines at the top of the file are
already there — that's what turns on the SQLite request log and per-request cost.

## Step 3 — Validate and start the container

From the **Terminal**:

```bash
agw-validate          # parses /root/config.yaml, exits 0 if valid
agw-up                # docker run the gateway: LLM :4000, MCP :3000, UI :15000
```

`agw-up` runs the container image `cr.agentgateway.dev/agentgateway:v1.3.1` with your
config, the cost catalog, and the request-log volume mounted in.

## Step 4 — Prove it routes LLM traffic

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' \
  | jq -r '.choices[0].message.content'
```

A sentence back means OpenAI traffic is flowing through your container.

## Step 5 — Prove MCP is proxied

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"c","version":"1"}}}'
```

`200` means the MCP server is proxied alongside your LLM gateway.

Open the **Agentgateway UI** tab (`:15000`) to see the dashboard. Click **Check** when
both endpoints answer.

> Stuck? `agw-logs` tails the container. The **Check** button will tell you exactly
> what's missing.
