# Instruqt Lab — Enterprise AgentGateway Budgets (v2026.6.3)

**Date:** 2026-07-02
**Status:** Approved by Sebastian (4-challenge story arc)
**Directory:** `02-agent-gateway-budget/`

## Goal

A clean, simple Instruqt track showcasing the new `EnterpriseAgentgatewayBudget`
CRD ("Budgets & Dimensions", Enterprise AgentGateway v2026.6.3) with OpenAI.
Learner ends by tripping a token Block budget (HTTP 429) while proving an
unbudgeted route keeps working.

Source material: sebbycorp/k8s-goose commit `e4d9f825` (working v2026.6.3
budget manifests, validated server-side) and the existing
`agentgateway-enterprise-workshop` track conventions.

## Track identity

- Slug `agentgateway-enterprise-budget`, title "Enterprise AgentGateway
  Budgets: Govern Your AI Spend".
- Same VM image as sibling labs (`solo-test-236622/workshop-instruqt-kgateway-20251023`,
  k3d pre-installed), `n1-standard-4`.
- Secrets: `OPENAI_API_KEY`, `AGENTGATEWAY_LICENSE_KEY`. Timelimit 3600s,
  idle 1800s.

## Track setup (`track_scripts/setup-server`)

Mirrors the enterprise workshop script, pinned to chart `v2026.6.3`, minus
monitoring and Solo UI (keep boot fast):

1. k3d cluster start, Gateway API CRDs v1.4.0 experimental.
2. Helm: `enterprise-agentgateway-crds` + `enterprise-agentgateway`
   (license key, `gatewayClassParametersRefs` → `agentgateway-params`).
3. Class-level `EnterpriseAgentgatewayParameters agentgateway-params`
   (shared extensions incl. ext-cache for budget counters, JSON access logs).
4. Gateway `agentgateway` (no `infrastructure.parametersRef` — challenge 02
   attaches one), `openai` AgentgatewayBackend (auth via `openai-secret`),
   `/openai` HTTPRoute.
5. systemd port-forward `agentgateway` svc → localhost:8080.

## Challenges

Tabs per challenge: Terminal + Code Editor (`/root`). Learner applies YAML via
heredocs, matching the enterprise workshop style. Live requests use
`gpt-4o-mini` (cheap; 2000-token budget trips in a few requests).

1. **01-gateway-is-running** — verify pods/CRDs (spotlight the new
   `enterpriseagentgatewaybudgets` CRD), Gateway programmed, first chat
   completion through `/openai`. Check: gateway programmed + live completion.
2. **02-price-every-request** — create `model-cost-catalog` ConfigMap
   (gpt-4o 2.50/10.00, gpt-4o-mini 0.15/0.60 USD per 1M tokens), create
   `EnterpriseAgentgatewayParameters budget-params` with
   `modelCatalog.sources[].configMap`, patch Gateway `infrastructure.parametersRef`
   → `budget-params`, create isolated `/budget-demo` HTTPRoute to the same
   backend. Learner computes USD cost of a request from usage + catalog rates.
   Check: all four resources present and route serving.
3. **03-audit-the-spend** — `EnterpriseAgentgatewayBudget budget-demo` with
   `demo-usd-audit` (USD 5 / Day / Audit) + `EnterpriseAgentgatewayPolicy
   budget-demo-enforcement` (`entBudgetEnforcement.discovery.namespaces.from:
   Same`, targetRef the `budget-demo` route). Traffic keeps flowing — Audit
   observes, never blocks. Check: budget + policy exist, route returns 200.
4. **04-enforce-the-budget** — add `demo-token-block` (Tokens 2000 / Day /
   Block) to the budget, loop completions until HTTP 429, then prove `/openai`
   still returns 200 (blast-radius isolation). Check: block entry exists +
   429 on `/budget-demo` + 200 on `/openai`.

Each challenge's `solve-server` applies the exact YAML the assignment teaches
so skip-to-end works; `check-server` uses `fail-message` per convention.

## Out of scope

- Budget **dimensions** (per-user/team scoping) — docs haven't landed
  (per k8s-goose design doc); mention in closing notes only.
- Monitoring stack, Solo UI, kagent.

## Verification

- `instruqt track validate` clean, then `instruqt track push`.
- Full runtime verification happens on first Instruqt play-through
  (`instruqt track test` optionally).
