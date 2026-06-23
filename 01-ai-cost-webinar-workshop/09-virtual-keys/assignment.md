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
virtual key; the gateway checks it, then uses its own provider key to call
OpenAI. Teams never see the real key, you can revoke a team's access by deleting
one line, and the key's `metadata` (team, owner) rides along for attribution.

## Step 1 — Add a virtual-key gate

Add an `apiKey` policy under `llm.policies` (alongside your rate limit), with one
key per team:

```yaml
llm:
  port: 4000
  policies:
    apiKey:                    # <-- add this
      mode: strict             # reject any request without a valid virtual key
      keys:
      - key: "sk-team-sales-DEMO"
        metadata: { team: sales, owner: "alice@example.com" }
      - key: "sk-team-eng-DEMO"
        metadata: { team: eng, owner: "bob@example.com" }
    localRateLimit:
    - { maxTokens: 200000, tokensPerFill: 200000, fillInterval: "1h", type: tokens }
  models:
    # ... your two models from the routing challenge stay unchanged ...
```

```bash
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

The gate is on the **`llm`** frontend (`:4000`); MCP keeps its own `:3000` port.

## Step 2 — No key, no service

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

Returns **401** — `mode: strict` rejects unkeyed traffic. Shadow AI can't even
connect.

## Step 3 — A team's virtual key works

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H 'Authorization: Bearer sk-team-sales-DEMO' \
  -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model
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
