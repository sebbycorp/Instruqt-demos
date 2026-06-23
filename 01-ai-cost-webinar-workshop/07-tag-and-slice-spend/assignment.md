---
slug: tag-and-slice-spend
id: ""
type: challenge
title: Tag & Slice Spend
teaser: Use CEL on llm.cost.total to tag expensive calls and split metrics by cost tier.
notes:
- type: text
  contents: |
    # 🏷️ Tag & Slice Spend

    The gateway already knows the realized USD cost of every call. With a one-line
    **CEL** expression you can tag each request (`expensive: true/false`) and add a
    `cost_tier` label to your metrics — so "high-cost traffic" becomes something
    you can filter logs by and chart in Grafana.
tabs:
- id: ""
  title: Terminal
  type: terminal
  hostname: server
- id: ""
  title: Editor
  type: code
  hostname: server
  path: /root
- id: ""
  title: Agentgateway UI
  type: service
  hostname: server
  port: 15000
  path: /ui
difficulty: ""
enhanced_loading: null
---

# Tag & Slice Spend

`llm.cost.total` is a **CEL** value available on every priced request. You can
compute new log fields and metric labels from it — no external pipeline needed.

## Step 1 — Add cost-based fields

Add `logging` and `metrics` blocks under the existing `config:` in
`/root/config.yaml` (paste the whole file to be safe):

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
        expensive: "llm.cost.total > 0.0001"
  metrics:
    fields:
      add:
        cost_tier: "llm.cost.total > 0.0001 ? 'high' : 'low'"
binds:
- port: 4000
  listeners:
  - protocol: HTTP
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
EOF
agentgateway -f /root/config.yaml --validate-only
agw-restart
```

## Step 2 — Generate a cheap and an expensive call

```bash
curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"x","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' >/dev/null
curl -s http://localhost:4000/v1/chat/completions -H 'x-priority: high' -H 'Content-Type: application/json' \
  -d '{"model":"x","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' >/dev/null
```

## Step 3 — See the tags in the log

```bash
grep -oE 'cost_usd=[^ ]+|expensive=[^ ]+' /root/agentgateway.log | tail -4
```

The cheap call shows `expensive=false`; the gpt-4o call shows `expensive=true`.
You can now `grep expensive=true` to find every costly request.

## Step 4 — Slice metrics by cost tier

```bash
curl -s http://localhost:15020/metrics | grep -oE 'cost_tier="[a-z]+"' | sort | uniq -c
```

You'll see both `cost_tier="low"` and `cost_tier="high"` series. In Grafana this
becomes a single query — *what fraction of spend is high-tier?* — across every
team and model.

> Next: agents don't only call models — they call **tools**. ➡️
