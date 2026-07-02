---
slug: audit-the-spend
id: fiimuzl57fbl
type: challenge
title: Audit the Spend
teaser: Create a USD budget in Audit mode — see overspend without blocking anyone.
notes:
- type: text
  contents: "# \U0001F440 Audit before you block\n\nRolling out hard spend limits
    on day one is how you break production agents. The safe pattern:\n\n1. **Audit**
    first — meter real spend against a budget, log when it's exceeded, block nothing\n2.
    **Block** later — once you trust the numbers\n\nAn `EnterpriseAgentgatewayBudget`
    holds one or more budget entries. Each entry has:\n\n- `limit` — an amount in
    `USD` or `Tokens`\n- `window` — the reset period (e.g. `Day`)\n- `onBudgetExceeded`
    — `Audit` or `Block`\n\nBudgets don't do anything until a **policy** turns enforcement
    on for a route. That separation lets platform teams own budgets while app teams
    own routes.\n"
tabs:
- id: v5fyjlmqjspx
  title: Terminal
  type: terminal
  hostname: server
- id: klf9y9987swj
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Audit the Spend

Create a **$5/day Audit budget** on the `/budget-demo` route. It watches the money — and never blocks.

## Step 1: Create the budget

> **What's happening:** This budget has a single entry, `demo-usd-audit`. The gateway meters the *realized USD cost* of every request on enforced routes — computed from token usage and your model cost catalog — against the $5 daily limit. `Audit` mode means exceeding the limit is logged, never blocked.

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayBudget
metadata:
  name: budget-demo
  namespace: agentgateway-system
spec:
  budgets:
    - name: demo-usd-audit
      limit:
        unit: USD
        amount: 5
      window:
        unit: Day
      onBudgetExceeded: Audit
EOF
```

## Step 2: Turn enforcement on for the route

> **What's happening:** The `entBudgetEnforcement` policy activates budget metering on its target — here, only the `budget-demo` HTTPRoute. `discovery.namespaces.from: Same` means only Budget resources in this namespace apply. Note what's *not* targeted: the `/openai` route has no budget policy at all.

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: budget-demo-enforcement
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: budget-demo
  traffic:
    entBudgetEnforcement:
      discovery:
        namespaces:
          from: Same
EOF
```

## Step 3: Send metered traffic

> **What's happening:** Every one of these requests is now priced against the catalog and counted toward `demo-usd-audit`. Because the mode is `Audit`, requests flow normally — you're building a real picture of spend with zero risk to callers.

```bash
for i in {1..3}; do
  curl -s "$GATEWAY_IP:8080/budget-demo" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"gpt-4o-mini\",
      \"messages\": [{\"role\": \"user\", \"content\": \"Reply with exactly: metered request $i\"}]
    }" | jq -r '.choices[0].message.content'
done
```

All three succeed — an Audit budget never says no.

## Step 4: Inspect the budget

> **What's happening:** The budget is a live Kubernetes resource. Its spec is the contract; the gateway's counters (in ext-cache) track consumption against it per window.

```bash
kubectl get enterpriseagentgatewaybudget budget-demo -n agentgateway-system -o yaml | grep -A12 'spec:'
```

The gateway now *knows* what this route spends per day. In the next challenge you'll make it *care* — with a Block budget you'll trip live.
