---
slug: build-an-mcp-server
id: twpwv7uh2e55
type: challenge
title: Build an MCP Server
teaser: Write a tiny MCP server in Python, run it, and proxy it through the gateway.
notes:
- type: text
  contents: |
    Agents don't just call models — they call tools over MCP. Here you'll build your own
    MCP server in Python (a few lines with FastMCP), run it as a container, and add it to
    the gateway config. Now both your LLM and your tool traffic share one control point.
tabs:
- id: bp2dprpsmanc
  title: Terminal
  type: terminal
  hostname: server
- id: y4ikvovjve5c
  title: Editor
  type: code
  hostname: server
  path: /root
- id: jurf2abm9dma
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Build an MCP Server

## Step 1 — Write the server

In the **Editor**, open `/root/mcp-server/server.py` and make it expose two tools. The
`FastMCP` helper turns plain Python functions into MCP tools — the docstring and type
hints become the tool schema the model sees:

```python
from mcp.server.fastmcp import FastMCP

# Bind to all interfaces on :8000 so the gateway container can reach us by name.
mcp = FastMCP("solo-tools", host="0.0.0.0", port=8000)

@mcp.tool()
def estimate_tokens(text: str) -> int:
    """Roughly estimate the number of tokens in a piece of text (~4 chars per token)."""
    return max(1, len(text) // 4)

@mcp.tool()
def estimate_cost(input_tokens: int, output_tokens: int) -> str:
    """Estimate the USD cost of a gpt-4.1-nano call from token counts."""
    cost = (input_tokens * 0.10 + output_tokens * 0.40) / 1_000_000
    return f"${cost:.6f}"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

## Step 2 — Run it as a container

```bash
mcp-up          # docker run your server.py on agw-net, listening on :8000/mcp
mcp-logs        # (optional) watch it start; Ctrl-C to stop tailing
```

`mcp-up` runs your code in the pre-built `mcp-python:local` image and names the
container `mcp-custom`, reachable on the Docker network at `http://mcp-custom:8000/mcp`.

## Step 3 — Add it to the gateway config

In `/root/config.yaml`, add a top-level `mcp:` block (a sibling of `llm:`) that proxies
your server on port `:3000`:

```yaml
mcp:
  port: 3000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      exposeHeaders: ["Mcp-Session-Id"]
  targets:
  - name: solo-tools
    mcp:
      host: http://mcp-custom:8000/mcp
```

Then validate and reload:

```bash
agw-validate
agw-reload
```

## Step 4 — List your tools through the gateway

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
  | grep '^data:' | sed 's/^data: //' \
  | jq '{tools: [.result.tools[].name]}'
```

You should see `estimate_tokens` and `estimate_cost`. Your own MCP server is now proxied
right alongside OpenAI. Click **Check**.

> Edited `server.py`? Re-run `mcp-up` to restart it, then `agw-reload`.
