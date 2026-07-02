---
slug: gateway-is-running
id: mhiggqyjdr1m
type: challenge
title: The Gateway Is Running
teaser: Verify Enterprise AgentGateway v2026.6.3 and send your first LLM request through
  it.
notes:
- type: text
  contents: "# \U0001F680 Enterprise AgentGateway v2026.6.3\n\nThis release introduces
    **Budgets & Dimensions** — native AI spend control at the gateway.\n\nRate limits
    count *requests*. But requests aren't what costs money with LLMs — **tokens and
    dollars** are. A single request can cost $0.0001 or $5 depending on the model
    and payload.\n\nThe new `EnterpriseAgentgatewayBudget` CRD lets you set limits
    denominated in **USD or tokens**, per time window, with two enforcement modes:\n\n-
    **Audit** — track and log overspend, never block\n- **Block** — return HTTP 429
    until the window resets\n\nFirst, let's make sure the gateway is up and passing
    LLM traffic.\n"
tabs:
- id: jjx9uzccdeas
  title: Terminal
  type: terminal
  hostname: server
- id: lp4r9i1fezgh
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# The Gateway Is Running

The track setup installed Enterprise AgentGateway **v2026.6.3** with an OpenAI backend and an `/openai` route. Let's verify everything and send a request through it.

## Step 1: Verify the components

> **What's happening:** The *control plane* (`enterprise-agentgateway`) watches Kubernetes for configuration changes and programs the *data plane* (`agentgateway`) — the proxy that actually handles LLM traffic. Shared extensions like `ext-cache` (Redis) hold the counters that budgets and rate limits rely on.

```bash
kubectl get pods -n agentgateway-system
```

You should see `enterprise-agentgateway` (control plane), `agentgateway` (data plane proxy), and the shared extensions (`ext-auth-service`, `rate-limiter`, `ext-cache`) all `Running`.

## Step 2: Meet the new Budget CRD

> **What's happening:** v2026.6.3 ships a new Custom Resource Definition: `EnterpriseAgentgatewayBudget`. It's a first-class Kubernetes resource — your AI spend policy lives in version-controlled YAML, just like the rest of your gateway config.

```bash
kubectl api-resources | awk 'NR==1 || /enterpriseagentgateway\.solo\.io|agentgateway\.dev/'
```

Spot the star of this lab:

```bash
kubectl explain enterpriseagentgatewaybudget.spec.budgets --recursive | head -20
```

Note the fields: `limit` (unit: `USD` or `Tokens`), `window`, and `onBudgetExceeded` (`Audit` or `Block`).

## Step 3: Verify the Gateway

> **What's happening:** The `Gateway` resource is a standard Gateway API object — the single entry point for all LLM traffic. A systemd port-forward exposes it on `localhost:8080`.

```bash
kubectl get gateway -n agentgateway-system
kubectl get httproute -n agentgateway-system
```

The `agentgateway` Gateway should be `PROGRAMMED: True` with an `/openai` HTTPRoute attached.

## Step 4: Send your first LLM request

> **What's happening:** The gateway proxies this to OpenAI, injecting the API key server-side from a Kubernetes Secret — clients never hold provider credentials. Note the `usage` block in the response: input and output token counts. That's the raw material budgets are built on.

```bash
curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "In one sentence: why do tokens, not requests, drive LLM cost?"}
    ]
  }' | jq '{answer: .choices[0].message.content, usage: .usage}'
```

You get an answer — and a token count. Next: teach the gateway what those tokens **cost**.
