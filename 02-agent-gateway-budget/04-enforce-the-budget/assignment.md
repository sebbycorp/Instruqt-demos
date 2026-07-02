---
slug: enforce-the-budget
id: 7gcpofz73t1a
type: challenge
title: Enforce the Budget
teaser: A budget you can't enforce isn't a budget. Add a Block entry, trip it live,
  get a clean 429.
notes:
- type: text
  contents: "# \U0001F6D1 Block mode\n\nAudit told you what's being spent. **Block**
    puts a hard ceiling on it:\n\n- Requests within budget → proxied normally\n- Requests
    over budget → **HTTP 429**, no call to OpenAI, no cost incurred\n- The counter
    resets when the window rolls over\n\nThe demo budget is tiny on purpose — **2000
    tokens/day** — so a few chat completions trip it while you watch.\n\nAnd because
    enforcement is scoped to the `/budget-demo` route, tripping it can't hurt anything
    else. That's the whole point of the isolated route you built in challenge 2.\n"
tabs:
- id: v5566xxw2lfl
  title: Terminal
  type: terminal
  hostname: server
- id: wdl0lmy9iy0c
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Enforce the Budget

Add a **Block** entry to the budget and trip it — live.

## Step 1: Add the Block entry

> **What's happening:** You're updating the existing budget to hold two entries. `demo-usd-audit` keeps watching dollars. The new `demo-token-block` caps the route at 2000 tokens per day — and this one says no. In real life this would be sized generously; here it's small enough to trip in a couple of requests.

```bash,run
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
    - name: demo-token-block
      limit:
        unit: Tokens
        amount: 2000
      window:
        unit: Day
      onBudgetExceeded: Block
EOF
```

The enforcement policy from challenge 3 already covers this route — new budget entries in the namespace are discovered automatically.

## Step 2: Burn the budget

> **What's happening:** Each request asks for a long answer (~600+ tokens), and each response's total tokens count against the 2000/day limit. Watch the HTTP status codes: 200s while there's budget left, then a 429 the moment it's exhausted — the gateway refuses *before* calling OpenAI, so a blocked request costs exactly $0.

```bash,run
for i in {1..6}; do
  CODE=$(curl -s -o /tmp/resp.json -w "%{http_code}" "$GATEWAY_IP:8080/budget-demo" \
    -H "content-type: application/json" \
    -d '{
      "model": "gpt-4o-mini",
      "messages": [{"role": "user", "content": "Explain Kubernetes in detail, about 400 words."}]
    }')
  TOKENS=$(jq -r '.usage.total_tokens // "-"' /tmp/resp.json)
  echo "request $i → HTTP $CODE (tokens: $TOKENS)"
  [ "$CODE" = "429" ] && break
done
```

You should see a few `HTTP 200`s and then:

```nocopy
request N → HTTP 429 (tokens: -)
```

That's the budget doing its job — a clean, machine-readable rejection instead of a surprise invoice at month-end.

## Step 3: Prove the blast radius is contained

> **What's happening:** The Block budget is enforced only where the policy targets it: the `budget-demo` route. The `/openai` route has no budget policy — its traffic flows untouched. This is how you protect experimental or untrusted workloads without risking the ones that pay the bills.

```bash,run
curl -s -o /dev/null -w "/budget-demo → HTTP %{http_code}\n" "$GATEWAY_IP:8080/budget-demo" \
  -H "content-type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "ping"}], "max_tokens": 5}'

curl -s -o /dev/null -w "/openai      → HTTP %{http_code}\n" "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "ping"}], "max_tokens": 5}'
```

```nocopy
/budget-demo → HTTP 429
/openai      → HTTP 200
```

## 🎉 That's the feature

You just gave an AI platform hard spend limits in four steps: **catalog → route → audit → block**. Where to go next:

- **Dimensions** (same v2026.6.3 release) let budgets scope per user, team, or agent — one CRD, per-subject counters.
- Size Block budgets from what Audit mode measures, not guesses.
- Keep budgeted routes isolated per workload, exactly like `/budget-demo`.
