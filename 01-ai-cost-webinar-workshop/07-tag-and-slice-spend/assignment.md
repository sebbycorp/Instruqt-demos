---
slug: tag-and-slice-spend
id: 6ati5cvb227q
type: challenge
title: Tag & Slice Spend
teaser: Use CEL on llm.cost.total to tag expensive calls and split metrics by cost
  tier.
notes:
- type: text
  contents: "# \U0001F3F7️ Tag & Slice Spend\n\nThe gateway already knows the realized
    USD cost of every call. With a one-line\n**CEL** expression you can tag each request
    (`expensive: true/false`) and add a\n`cost_tier` label to your metrics — so \"high-cost
    traffic\" becomes something\nyou can filter logs by and chart in Grafana.\n"
tabs:
- id: ms0cvommpjfe
  title: Terminal
  type: terminal
  hostname: server
- id: pmekesvfzosb
  title: Editor
  type: code
  hostname: server
  path: /root
- id: focswpre7xck
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Tag & Slice Spend

`llm.cost.total` is a **CEL** value available on every priced request. You can
compute new log fields and metric labels from it — no external pipeline needed.

## Step 1 — Add cost-based fields

Add `logging` and `metrics` blocks under the existing `config:` in
`/root/config.yaml` (your `llm:` models from the last challenge stay as-is):

```yaml
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:///root/data/data.db"
  modelCatalog:
    - file: /root/costs/base-costs.json
  logging:                                       # <-- add
    fields:
      add:
        cost_usd: "llm.cost.total"
        expensive: "llm.cost.total > 0.0001"
  metrics:                                        # <-- add
    fields:
      add:
        cost_tier: "llm.cost.total > 0.0001 ? 'high' : 'low'"
```

Then validate and restart:

```bash
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
