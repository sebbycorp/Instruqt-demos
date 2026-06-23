---
slug: virtual-keys
id: doybwiiiixus
type: challenge
title: Per-Team Virtual Keys
teaser: Issue your own gateway keys per team — the real provider key never leaves
  the gateway.
notes:
- type: text
  contents: "# \U0001F511 Per-Team Virtual Keys\n\nRight now the gateway holds one
    real OpenAI key and accepts anyone who can\nreach :4000. In production you issue
    **your own** keys — one per team — and the\ngateway validates them. The real provider
    key never leaves the gateway, and\nevery call is attributable to the team that
    made it.\n"
tabs:
- id: 3xrnkpjxn0zz
  title: Terminal
  type: terminal
  hostname: server
- id: ygvfvwyuktlh
  title: Editor
  type: code
  hostname: server
  path: /root
- id: vfxojhwunhxo
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Per-Team Virtual Keys

A **virtual key** is a credential *you* mint and hand to a team. Clients send the
virtual key; the gateway checks it, then uses its own `backendAuth` key to call
OpenAI. Teams never see the real key, you can revoke a team's access by deleting
one line, and the key's `metadata` (team, owner) rides along for attribution.

## Step 1 — Add a virtual-key gate

Add a listener-level `apiKey` policy with one key per team. Paste the full config:

```bash
cat > /root/config.yaml <<'EOF'
config:
  adminAddr: "0.0.0.0:15000"
  modelCatalog:
  - file: /root/costs/catalog.json
  logging:
    fields:
      add:
        cost_usd: "llm.cost.total"
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    policies:
      apiKey:
        mode: strict          # reject any request without a valid virtual key
        keys:
        - key: "sk-team-sales-DEMO"
          metadata: { team: sales, owner: "alice@example.com" }
        - key: "sk-team-eng-DEMO"
          metadata: { team: eng, owner: "bob@example.com" }
    routes:
    - name: premium-route
      matches:
      - path: { pathPrefix: /v1 }
        headers:
        - name: x-priority
          value: { exact: high }
      policies:
        backendAuth: { key: "$OPENAI_API_KEY" }
        ai: { overrides: { model: gpt-4o } }
      backends: [{ ai: { name: openai, provider: { openAI: {} } } }]
    - name: default-route
      matches:
      - path: { pathPrefix: /v1 }
      policies:
        backendAuth: { key: "$OPENAI_API_KEY" }
        ai: { overrides: { model: gpt-4o-mini } }
      backends: [{ ai: { name: openai, provider: { openAI: {} } } }]
- port: 3000
  listeners:
  - protocol: HTTP
    routes:
    - name: mcp-route
      backends:
      - mcp:
          targets:
          - name: everything
            stdio: { cmd: npx, args: ["-y","@modelcontextprotocol/server-everything"] }
EOF
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

The virtual-key gate is on the **`:4000` (LLM)** listener; MCP keeps its own
`:3000` bind from the previous challenge.

## Step 2 — No key, no service

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"x","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

Returns **401** — `mode: strict` rejects unkeyed traffic. Shadow AI can't even
connect.

## Step 3 — A team's virtual key works

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H 'Authorization: Bearer sk-team-sales-DEMO' \
  -H 'Content-Type: application/json' \
  -d '{"model":"x","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
```

Returns a normal response (`gpt-4o-mini`) — and the gateway used the **real**
OpenAI key, which the sales team never saw. Check the log:

```bash
grep -oE 'apiKey.name=[^ ]+|apiKey.owner=[^ ]+|cost_usd=[^ ]+' /root/agentgateway.log | tail -3
```

Every call now carries the **team and owner** — the live version of the
attribution you're about to do across the fleet.

## Step 4 — Revoke in one line

Delete a team's key from the `keys:` list and `agw-restart` — that team is cut
off instantly, with zero impact on anyone else and no provider-key rotation.

> Last stop: put all of this together and **answer the CFO**. ➡️
