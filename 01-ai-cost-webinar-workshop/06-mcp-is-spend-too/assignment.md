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

## Step 1 — Add an MCP route

In the **Editor**, add a second route to `/root/config.yaml` (a sibling of
`openai-route`, under the same listener's `routes:`):

```yaml
    - name: mcp-route
      matches:
      - path:
          pathPrefix: /mcp
      backends:
      - mcp:
          targets:
          - name: everything
            stdio:
              cmd: npx
              args: ["-y","@modelcontextprotocol/server-everything"]
```

This proxies the reference `server-everything` MCP server (started on demand via
`npx`) and exposes it at **`:4000/mcp`** — right next to your LLM route.

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

## Step 2 — Confirm both routes are live

```bash
curl -s http://localhost:15000/config_dump | jq '.binds[].listeners[].routes[]?.name? // empty'
```

You should see **both** `openai-route` and `mcp-route`. One gateway, both traffic
types.

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

> You now control LLM and MCP traffic from one gateway. Last stop: **answer the
> CFO** with real numbers. ➡️
