---
slug: price-every-request
id: x5fxek7pzgyv
type: challenge
title: Price Every Request
teaser: Load a model cost catalog so the gateway knows the real USD cost of every
  LLM call.
notes:
- type: text
  contents: "# \U0001F4B5 From tokens to dollars\n\nA budget in **USD** only works
    if the gateway can price a request. That's the job of the **model cost catalog**
    — per-1M-token rates for each provider and model, loaded into the data plane.\n\n```\nrequest
    ──▶ gateway ──▶ OpenAI\n              │\n              ├─ usage: 21 in / 57 out
    tokens\n              └─ catalog: gpt-4o-mini $0.15 in / $0.60 out per 1M\n                 →
    realized cost: $0.0000374\n```\n\nYou'll also create a dedicated `/budget-demo`
    route. Budgets attach to routes — an isolated route means you can trip a Block
    budget later **without touching** anything else using `/openai`.\n"
tabs:
- id: sevhunvoyefw
  title: Terminal
  type: terminal
  hostname: server
- id: zm4qwguk35yz
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Price Every Request

Budgets denominated in USD need a price list. Let's give the gateway one.

## Step 1: Create the model cost catalog

> **What's happening:** The catalog is a plain ConfigMap with per-1M-token USD rates. These match OpenAI's public pricing: gpt-4o at $2.50 in / $10.00 out, gpt-4o-mini at $0.15 in / $0.60 out. The data plane multiplies each response's token usage by these rates to compute the *realized* dollar cost of every request.

```bash,run
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-cost-catalog
  namespace: agentgateway-system
data:
  catalog.json: |
    {
      "providers": {
        "openai": {
          "models": {
            "gpt-4o": {
              "rates": { "input": "2.50", "output": "10.00" }
            },
            "gpt-4o-mini": {
              "rates": { "input": "0.15", "output": "0.60" }
            }
          }
        }
      }
    }
EOF
```

## Step 2: Load the catalog into the proxy

> **What's happening:** `EnterpriseAgentgatewayParameters` configures the data plane. The `modelCatalog.sources` field tells the proxy where to read pricing from. Attaching it to the Gateway via `infrastructure.parametersRef` scopes it to this gateway's proxy.

Create the parameters:

```bash,run
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: budget-params
  namespace: agentgateway-system
spec:
  modelCatalog:
    sources:
      - configMap:
          name: model-cost-catalog
          key: catalog.json
EOF
```

Attach it to the Gateway:

```bash,run
kubectl patch gateway agentgateway -n agentgateway-system --type=merge -p '
spec:
  infrastructure:
    parametersRef:
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
      name: budget-params
'
```

Wait for the proxy to roll out with the new config:

```bash,run
kubectl -n agentgateway-system rollout status deployment -l app.kubernetes.io/name=agentgateway --timeout=120s
```

## Step 3: Create the isolated /budget-demo route

> **What's happening:** This route proxies to the **same** OpenAI backend as `/openai`, but on its own path. Budgets are enforced per-route — so when you trip a Block budget on `/budget-demo` in challenge 4, `/openai` traffic is completely unaffected. That's your blast-radius control.

```bash,run
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: budget-demo
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /budget-demo
      backendRefs:
        - name: openai
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
EOF
```

## Step 4: Price a request by hand

> **What's happening:** Send a completion through the new route, then compute what it cost using the catalog rates. This is exactly the math the gateway now does internally on every request — and what USD budgets meter against.

```bash,run
curl -s "$GATEWAY_IP:8080/budget-demo" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "content": "Give me a 2-sentence haiku-style ode to cost control."}]
  }' | jq '{usage: .usage,
           cost_usd: ((.usage.prompt_tokens * 0.15 + .usage.completion_tokens * 0.60) / 1000000)}'
```

Fractions of a cent per request — but multiply by thousands of agents running 24/7 and it's real money. Time to put a budget on it.
