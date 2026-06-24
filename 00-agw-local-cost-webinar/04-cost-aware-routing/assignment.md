---
slug: cost-aware-routing
id: f86rf7tky2tk
type: challenge
title: Cost-Aware Routing
teaser: Cheap by default, premium only when it's worth it — decided at the gateway.
notes:
- type: text
  contents: |
    Not every request deserves your most expensive model. The gateway can route by a
    request header: default traffic goes to cheap `gpt-4.1-nano`, while requests that
    opt in with `x-priority: high` get premium `gpt-4.1`. The caller never sees an API
    key or a model list — routing and the resulting cost are decided centrally.
tabs:
- id: 3qpnrz9earsx
  title: Terminal
  type: terminal
  hostname: server
- id: nn9ujeydtrnc
  title: Editor
  type: code
  hostname: server
  path: /root
- id: k96sgwhmffiy
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Cost-Aware Routing

Right now `/root/config.yaml` has one route: everything → `gpt-4.1-nano`. Add a premium
route in front of it that only matches when the request carries `x-priority: high`.

## Step 1 — Add a header-matched premium route

Order matters: the first matching model wins, so the premium route must come **before**
the catch-all. Replace the `llm.models` list with:

```yaml
  models:
  - name: "openai/*"
    matches:
    - headers:
      - name: x-priority
        value: { exact: high }
    provider: openAI
    params:
      model: gpt-4.1
      apiKey: "$OPENAI_API_KEY"
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
```

## Step 2 — Apply it

```bash
agw-validate
agw-restart
```

## Step 3 — Watch the routing decision

Same model name in the request body, two different backends depending on the header:

```bash
echo "default  -> $(curl -s http://localhost:4000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"hi"}],"max_tokens":6}' | jq -r .model)"

echo "priority -> $(curl -s http://localhost:4000/v1/chat/completions \
  -H 'x-priority: high' -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"hi"}],"max_tokens":6}' | jq -r .model)"
```

Default should report `gpt-4.1-nano`; priority should report `gpt-4.1`.

## Step 4 — See the cost gap

```bash
sqlite3 -box /root/data/data.db \
"SELECT gen_ai_request_model AS asked,
        printf('\$%.2f', cost*1000000) AS per_1M_calls
 FROM request_logs ORDER BY completed_at DESC LIMIT 6;"
```

The premium calls cost ~20× the cheap ones per the catalog (`gpt-4.1` \$2/\$8 vs
`gpt-4.1-nano` \$0.10/\$0.40 per 1M tokens). You just made that a per-request decision.

Click **Check** when both routes resolve correctly.
