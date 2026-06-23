---
slug: mcp-is-spend-too
id: esebezs0fzq6
type: challenge
title: MCP Is Spend Too
teaser: Bring agent tool traffic under the same gateway — and see why tools cost tokens.
notes:
- type: text
  contents: "# \U0001F50C MCP Is Spend Too\n\nAgents don't just call models — they
    call **tools** over MCP. Every tool\nschema and every tool result becomes tokens
    in the model's context. Put MCP\nbehind the same gateway so it shares the same
    logs, budgets, and control.\n"
tabs:
- id: watyjcfkrh05
  title: Terminal
  type: terminal
  hostname: server
- id: tydwe9v82pmm
  title: Editor
  type: code
  hostname: server
  path: /root
- id: nr5tmeg6jh3m
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# MCP Is Spend Too

**MCP** (Model Context Protocol) is how agents discover and call tools. The cost
catch: every tool's JSON schema is sent to the model as **input tokens**, and
every tool **result** comes back as input tokens on the *next* call. Ten tools
with verbose schemas can add thousands of input tokens to every single turn —
spend that's completely invisible if MCP traffic doesn't go through your gateway.

Agentgateway proxies MCP servers too, so LLM **and** tool traffic share one
control point.

## Step 1 — Add an MCP server on its own port

LLM lives on `:4000`. Give MCP its **own port, `:3000`** — `llm` and `mcp` must
use unique ports. In the **Editor**, add a top-level `mcp:` block to
`/root/config.yaml` (a sibling of `llm:`):

```yaml
mcp:
  port: 3000
  targets:
  - name: everything
    stdio:
      cmd: npx
      args: ["-y","@modelcontextprotocol/server-everything"]
```

This proxies the reference `server-everything` MCP server (started on demand via
`npx`) and exposes it at **`:3000`** — alongside your LLM gateway on `:4000`.

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

## Step 2 — Confirm MCP is live on :3000

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"c","version":"1"}}}'
```

`200` means the MCP server is proxied and answering.

## Step 3 — Explore MCP in the UI

Open the **Agentgateway UI** tab (`:15000/ui`) and find the **MCP / Tools
Playground**. List the tools exposed by `server-everything` and invoke one. Notice
how much tool metadata (names, descriptions, JSON schemas) the model would have to
carry as input tokens — that's the hidden cost of tool-heavy agents, now visible
and governable at the gateway.

## Step 4 — Why this matters for cost

- **Same logs** — MCP calls land in the same access log as LLM calls.
- **Same governance** — the same budget/policy machinery applies.
- **Same `config_dump`** — one place to see everything an agent can reach.

Without this, MCP tool traffic is yet another blind spot inflating your token
bill. With it, the tools your agents use are as visible and governable as the
models they call.

> You now control LLM and MCP traffic from one gateway. Next: issue **per-team
> virtual keys** so every call is authenticated and attributable. ➡️
