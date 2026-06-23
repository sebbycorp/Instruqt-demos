---
slug: production-scale
id: ""
type: challenge
title: Production Scale
teaser: Where enterprise adoption goes next — guardrails, per-team keys, tracing, Enterprise.
notes:
- type: text
  contents: |
    # 🏁 Production Scale

    You took a blind, ungoverned AI bill and brought it under one control point.
    Here's what the production-grade version of this looks like.
tabs:
- id: ""
  title: Terminal
  type: terminal
  hostname: server
difficulty: ""
enhanced_loading: null
---

# Production Scale

In ~15 minutes you went from *"who spent this?"* to a single gateway that gives
you **visibility, attribution, troubleshooting, and governance** over LLM and
MCP traffic. Recap of the journey:

1. **The blind spot** — tokens, cost, and why dashboards fail.
2. **Stand up the gateway** — one chokepoint for all AI traffic.
3. **Cost of every request** — token + USD attribution, live.
4. **Day-2 ops** — operate and troubleshoot via `:15000`.
5. **Governance** — token budgets + approved-model pin.
6. **MCP is spend too** — tools under the same control point.
7. **Answer the CFO** — per-team, per-user, per-workload attribution.

## What production adds

- **Prompt guards / guardrails** — regex rules to block secrets and PII in
  requests and mask them in responses; OpenAI Moderation, AWS Bedrock
  Guardrails, and Google Model Armor integrations.
  → `agentgateway.dev/docs/standalone/latest/llm/prompt-guards/overview`
- **Per-team budgets at scale** — `remoteRateLimit` descriptors keyed by API key,
  user, or group, backed by a shared rate-limit service across replicas.
  → `agentgateway.dev/docs/standalone/latest/configuration/resiliency/rate-limits`
- **Virtual keys** — issue per-team gateway keys so the upstream provider key is
  never exposed and every call is attributable.
  → `agentgateway.dev/docs/standalone/latest/llm/virtual-keys`
- **Tracing & observability** — OpenTelemetry traces with full token/cost
  attributes; Langfuse for prompt/response tracing and per-session cost; Grafana
  dashboards off the `:15020` metrics.
  → `agentgateway.dev/docs/standalone/latest/llm/observability`
- **Agentgateway Enterprise (Solo.io)** — managed control plane, UI, SSO,
  multi-cluster, and support for running this across the whole org.

## Try it on your own traffic

Point any OpenAI client at `http://localhost:4000/v1`, set a throwaway API key,
and your real apps flow through the gateway — instantly metered and governed.

> Thanks for taking the workshop. Click **Check** to finish. 🎉
