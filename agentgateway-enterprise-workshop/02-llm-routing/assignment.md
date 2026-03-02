---
slug: llm-routing
id: xnmpczhy75k7
type: challenge
title: LLM Routing & Observability
teaser: Route your first LLM request through the gateway and explore metrics and traces.
notes:
- type: text
  contents: "# \U0001F310 LLM Routing\n\nThe core value of an AI gateway: **one entry
    point for all your LLM traffic**.\n\nInstead of every agent calling OpenAI directly
    with its own API key, they all go through the gateway. You get:\n\n- Centralized
    credential management\n- Per-model metrics and token tracking\n- Distributed traces
    showing prompt → response\n- A single place to add policies later\n\nLet's route
    traffic to OpenAI and see the observability.\n"
tabs:
- id: rfoqyrehs9zo
  title: Terminal
  type: terminal
  hostname: server
- id: qjxngsqzpfjd
  title: Code Editor
  type: code
  hostname: server
  path: /root
- id: ptw0xxk0ldit
  title: Grafana
  type: service
  hostname: server
  port: 3000
- id: ylwlqnwkfj7c
  title: Solo UI
  type: service
  hostname: server
  port: 4000
difficulty: ""
enhanced_loading: null
---

# LLM Routing & Observability

Time to route your first LLM request through the gateway. You'll create an OpenAI backend and HTTPRoute, send a request, and see it show up in Grafana.

## Step 1: Create the OpenAI Backend and Route

> **What's happening:** You're creating two Kubernetes resources that work together. The `AgentgatewayBackend` tells the gateway *where* to send traffic — in this case, OpenAI's API — and which credentials to use (the `openai-secret` was pre-created during setup). The `HTTPRoute` tells the gateway *when* to send traffic there — any request whose path starts with `/openai` gets routed to this backend. This is the standard Kubernetes Gateway API pattern: routes match traffic, backends define destinations.

The OpenAI API key secret was already created during track setup. Now create the backend and route:

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: openai
  namespace: agentgateway-system
spec:
  parentRefs:
    - name: agentgateway
      namespace: agentgateway-system
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /openai
      backendRefs:
        - name: openai-all-models
          group: agentgateway.dev
          kind: AgentgatewayBackend
      timeouts:
        request: "120s"
---
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: openai-all-models
  namespace: agentgateway-system
spec:
  ai:
    provider:
      openai: {}
  policies:
    auth:
      secretRef:
        name: openai-secret
EOF
```

## Step 2: Send a Request Through the Gateway

> **What's happening:** This is the moment of truth — you're sending a real OpenAI chat completion request, but instead of hitting `api.openai.com` directly, it goes through your gateway at `localhost:8080/openai`. The gateway intercepts the request, injects the real OpenAI API key from the Kubernetes secret, forwards it to OpenAI, and returns the response. From the client's perspective, it looks like a normal OpenAI call. Behind the scenes, the gateway captured metrics (token counts, latency) and emitted a distributed trace.

```bash
source /root/.bashrc

curl -s "$GATEWAY_IP:8080/openai" \
  -H "content-type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "What is an AI gateway and why do enterprises need one? Answer in 2 sentences."
      }
    ]
  }' | jq .
```

You should get a response from OpenAI — but now it went through your gateway, which means metrics and traces were captured.

## Step 3: View Access Logs

> **What's happening:** The gateway automatically writes a structured JSON log for every request. Unlike generic HTTP access logs, these include AI-specific fields: which model was requested, which model actually responded (they can differ), how many input and output tokens were used, and the full prompt and response bodies. The trace ID in each log entry lets you correlate it with the distributed trace in Grafana or the Solo UI. This is the foundation for cost tracking, audit, and debugging.

AgentGateway logs every request with LLM-specific metadata:

```bash
kubectl logs deploy/agentgateway -n agentgateway-system --tail 5
```

Notice the log includes:
- `gen_ai.request.model` and `gen_ai.response.model`
- `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens`
- `request.body` and `response.body` for full prompt visibility
- Trace ID for correlation with distributed traces

## Step 4: View Metrics Endpoint

> **What's happening:** The gateway exposes a Prometheus-compatible `/metrics` endpoint on a sidecar port (15020). Prometheus scrapes this endpoint automatically. The metrics include counters for total requests, token usage breakdowns (input vs. output), latency histograms, and per-model/per-backend breakdowns. These are the same metrics that power Grafana dashboards and alerts in production.

AgentGateway exposes Prometheus-compatible metrics:

```bash
kubectl port-forward -n agentgateway-system deployment/agentgateway 15020:15020 &
sleep 2
curl -s http://localhost:15020/metrics | grep -E "agentgateway_(request|token)" | head -20
kill %1 2>/dev/null
```

## Step 5: Explore Grafana

> **What's happening:** Grafana is pre-configured with Tempo as a data source. When you explore Tempo, you'll see distributed traces — each trace represents one LLM request flowing through the gateway. The trace spans show: client → gateway ingress → backend routing → OpenAI API call → response processing. Inside these spans you'll find LLM-specific attributes like prompt content, token counts, model names, and timing breakdowns. This is how you debug slow responses or unexpected behavior in production.

Switch to the **Grafana** tab and log in:
- Username: `admin`
- Password: `admin`

Navigate to **Home > Explore** and select **Tempo** to see distributed traces for your request. You'll see LLM-specific spans with prompt content, token counts, and response data.

## Step 6: Explore the Solo UI

> **What's happening:** The Solo UI provides an alternative, AI-gateway-focused view of the same trace data. While Grafana is a general-purpose observability tool, the Solo UI is purpose-built for AgentGateway — it understands the relationship between routes, backends, and policies, and presents traces in that context. This is especially useful for platform teams who want a dedicated AI gateway operations dashboard separate from their general infrastructure monitoring.

Switch to the **Solo UI** tab. The Solo Enterprise Management UI also collects traces from the gateway. Navigate to the **Tracing** section to see your LLM requests with full execution details. This gives you a second view of the same traffic — useful for teams that prefer a dedicated AI gateway dashboard over Grafana.

## Step 7: Send a Few More Requests

> **What's happening:** Sending multiple requests in parallel (using `&` to background each `curl`) generates enough data to see meaningful patterns in Grafana's metrics and traces. Each request uses a different prompt (`Count to 1`, `Count to 2`, `Count to 3`), so you'll see varying token counts in the metrics — demonstrating the gateway's ability to track per-request token usage. The `wait` command ensures all background requests complete before moving on.

Generate some traffic for better metrics:

```bash
for i in {1..3}; do
  curl -s "$GATEWAY_IP:8080/openai" \
    -H "content-type: application/json" \
    -d "{
      \"model\": \"gpt-4o-mini\",
      \"messages\": [{\"role\": \"user\", \"content\": \"Count to $i\"}]
    }" | jq -r '.choices[0].message.content' &
done
wait
```

## ✅ What You've Learned

- `AgentgatewayBackend` defines where to route LLM traffic (provider + credentials)
- `HTTPRoute` maps URL paths to backends (standard Gateway API)
- Every request gets: structured logs, Prometheus metrics, OTLP traces
- Token usage is tracked automatically per model
- Traces are available in both Grafana (Tempo) and the Solo UI

**Next up:** API Key Management — control who can access the gateway.
