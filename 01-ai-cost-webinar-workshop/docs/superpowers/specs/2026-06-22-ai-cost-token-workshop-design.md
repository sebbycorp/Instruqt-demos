# AI Cost & Token Spend Workshop — Design Spec

**Date:** 2026-06-22
**Author:** Sebastian Maniak
**Status:** Approved (structure) — pending spec review

## Summary

A hands-on Instruqt workshop teaching how LLM/MCP token spend works and how to
track, troubleshoot, and govern it using **agentgateway OSS standalone v1.3.1**
(the single binary from github.com/agentgateway/agentgateway, *not* the
Kubernetes/Helm control plane used by sibling tracks).

The workshop targets a **~15 minute** self-paced run (8 challenges, two of which
are read-only). It uses **real OpenAI calls** through the gateway for live token
+ cost visibility, and a **mock SQLite dataset** for fleet-scale cost analysis.

## Narrative — The Enterprise Story

The whole track is told from the seat of a **platform / FinOps team at an
enterprise that is adopting AI fast and unevenly**. Dozens of dev teams
(eng, sales, support, security, product, marketing) are calling OpenAI and other
providers directly — through Cursor, Claude Code, SDKs, scripts — with no shared
control point. This is the audience: **enterprise practitioners evaluating
agentgateway** as the answer.

The inciting incident: **Finance forwards a surprise five-figure AI bill and
asks "who spent this?"** — and the platform team can't answer. Provider
dashboards are per-account, delayed, and don't map to teams, agents, or humans.
That's *shadow AI*, and it's the same loss-of-control enterprises hit with
microservices before API gateways and service meshes.

Each challenge is a beat in getting that control back:

1. **The blind spot** — why the bill is a mystery and dashboards don't cut it.
2. **Stand up the chokepoint** — put agentgateway in front of all AI traffic.
3. **Attribution** — every call now carries tokens + realized USD cost.
4. **Day-2 ops** — operate and troubleshoot the gateway via the admin API.
5. **Governance** — enforce per-team token budgets and an approved-model policy
   to stop the runaway spender (FinOps guardrails).
6. **MCP is spend too** — bring agent tool traffic under the same control point.
7. **Answer the CFO** — "show me spend by team/model/user" from the fleet data.
8. **Production scale** — what enterprise adoption looks like next (Langfuse,
   per-team virtual keys, guardrails, SSO, Enterprise).

**Tone:** speak to platform/FinOps/SecOps leads evaluating a purchase, not to a
hobbyist. Each `assignment.md` opens with the business stake ("why this matters
to the org") before the commands, and closes by tying the result back to the
surprise-bill problem. The mock dataset's users/groups/spend-tiers are the cast
of this story (e.g. the `enterprise-batch` top spender, per-`group` chargeback).

## Goals

- Explain how tokens are calculated and what LLM features cost (streaming,
  reasoning tokens, cached input, embeddings, tool/function calls).
- Show agentgateway capturing per-request token usage and **realized USD cost**
  (`gen_ai.usage.*_tokens`, `agw.ai.usage.cost.total`).
- Teach troubleshooting via the admin API on `:15000` (`/config_dump`, `/ui`,
  `/metrics`).
- Govern spend with **token-budget rate limits** and **model restriction**.
- Bring **MCP** under the same gateway and explain its token-cost implications.
- Analyze a realistic multi-user / multi-provider / multi-day cost dataset with SQL.

## Non-Goals (YAGNI)

- No Kubernetes / k3d / Helm (binary only).
- Regex/PII prompt guards — mentioned in the closing "what's next" challenge only.
- No Jaeger/Grafana stack, multi-provider failover, or virtual-key auth flows in
  the core path.

## Deployment Model

- **Single VM**, reusing the existing image
  `solo-test-236622/workshop-instruqt-kgateway-20251023`, `n1-standard-4`,
  `secrets: [OPENAI_API_KEY]` (same `config.yml` pattern as sibling tracks).
- agentgateway runs as a **background binary**: `agentgateway -f /root/config.yaml`.
  - LLM (OpenAI-compatible) API: **`:4000`**
  - Admin + UI + config_dump + metrics: **`:15000`**
- Install: `curl -sL https://agentgateway.dev/install | bash -s -- --version 1.3.1`
- The config file **evolves across challenges**. Every `solve-server` writes the
  canonical config for its challenge and restarts the gateway, so a learner can
  skip ahead from any point.
- Each challenge exposes two Instruqt tabs: a **Terminal** and a **code Editor**
  rooted at `/root` (so learners can see/edit `config.yaml`).

### Config form

agentgateway supports two config shapes:
1. `llm.models` shorthand (simplest, auto-binds).
2. Full `binds → port → listeners → routes → backends (+ policies)`.

We use the **full `binds` form** bound to `:4000` because it makes routes,
backends, and attached policies explicit — which is what `config_dump` shows and
what the policy challenge attaches to. Exact AI-backend block and policy block
YAML to be **finalized against `configuration/backends.md` and
`configuration/policies/attachment.md` during implementation** before any config
is written.

Reference skeleton (verify exact AI-backend keys at build time):

```yaml
# yaml-language-server: $schema=https://agentgateway.dev/schema/config
config:
  modelCatalog:
    - file: /root/costs/catalog.json   # added in Challenge 3
binds:
  - port: 4000
    listeners:
      - name: llm
        protocol: HTTP
        routes:
          - name: openai-route
            backends:
              - ai:                      # AI/LLM backend — exact keys TBD at build
                  provider: openAI
                  apiKey: "$OPENAI_API_KEY"
            policies: {}                 # token budget + model restriction added in Ch5
```

## Mock Data

- The provided generator (`Untitled (1)`, a stdlib-only Python script despite its
  `uv` shebang) is committed into the track as `assets/gen-mock-logs.py`.
- Run with system `python3`:
  `python3 assets/gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db`
- Its schema (`request_logs` + `attributes_json` carrying `gen_ai.*` and
  `agw.ai.usage.cost.*`) already mirrors agentgateway's real request-log format,
  so the analysis queries are authentic.

## OpenAI CLI

- Official `openai` pip CLI, installed in track/challenge setup.
- Pointed at the gateway: `OPENAI_BASE_URL=http://localhost:4000/v1`,
  `OPENAI_API_KEY=<any-placeholder>` (the gateway injects the real key).
- Used to make calls that flow through agentgateway so they appear in its logs.
- `curl` is the fallback/alternative client throughout.

## Challenge Breakdown

Each non-read-only challenge has `setup-server` (idempotent prep),
`check-server` (verification), and `solve-server` (writes canonical config +
restarts gateway). Read-only challenges have a trivially-passing check.

### 1. Why AI Cost Is Invisible (read-only)
Tokens = input + output, priced per 1M and varying by model; context window;
what costs extra (streaming, reasoning tokens, cached input, embeddings, tool
calls); why provider dashboards are fragmented/delayed and can't attribute spend
to a user or agent. Sets up the gateway as the single control point.
**Check:** trivial pass.

### 2. Install Agentgateway & Route to OpenAI
Install the v1.3.1 binary; write `config.yaml` (bind `:4000`, OpenAI route);
start the gateway in the background; send a first chat completion via `curl` to
`:4000`.
**Check:** agentgateway process running, `:4000` returns a completion,
`:15000/config_dump` reachable.

### 3. See Every Request's Token & Cost
Add `modelCatalog` so realized USD cost attaches; install the OpenAI CLI; make a
real call through the gateway; read `gen_ai.usage.input_tokens` /
`output_tokens` and `agw.ai.usage.cost.total` in the structured logs; compare a
gpt-4o vs gpt-4o-mini call to see the cost delta.
**Check:** model catalog configured and a priced request observed in logs.

### 4. Troubleshoot with the Admin API
`curl :15000/config_dump` (version/build + binds/listeners/routes/backends/
policies); browse `:15000/ui`; hit `/metrics` (e.g.
`agentgateway_cost_catalog_lookups_total`). Diagnose an intentionally broken
route by reading `config_dump`.
**Check:** `config_dump` fetched / admin reachable.

### 5. Cost Policies: Budgets & Model Limits
Attach `localRateLimit` with `type: tokens` (cap tokens per window) and trip a
`429`; add a **model restriction** policy (force `gpt-4o-mini` / block an
expensive model) and observe a denied request.
**Check:** both policies present in `config_dump` and enforcement observed.

### 6. MCP Through the Gateway
Proxy an MCP server (e.g. `server-everything` over Streamable HTTP) through
agentgateway; route a tool call; explain how MCP tool schemas + tool results
inflate the token context (and therefore spend), and that the gateway gives one
control point + visibility for both LLM and MCP traffic. Lean hands-on.
**Check:** MCP backend present in config / tool list reachable through gateway.

### 7. Analyze the Fleet — Mock Cost Data
Generate the SQLite DB with `gen-mock-logs.py`; run SQL queries: total spend,
top spenders by user and by group, cost by model/provider, error rate, batch vs
interactive workload split.
**Check:** DB generated and a representative query returns rows.

### 8. What's Next (read-only)
Regex/PII guardrails, virtual keys, Langfuse observability, agentgateway
Enterprise. Pointers to docs.
**Check:** trivial pass.

## Track Assets & Layout

```
01-ai-cost-webinar-workshop/
  track.yml                      # slug/title/teaser/description/tags/timelimit/lab_config
  config.yml                     # VM image + OPENAI_API_KEY secret
  track_scripts/
    setup-server                 # install binary + openai CLI + python deps; store key
    cleanup-server
  assets/
    gen-mock-logs.py             # the provided generator
    costs/catalog.json           # model cost catalog (Ch3)
  01-why-ai-cost-is-invisible/   assignment.md (+ check/solve as needed)
  02-install-and-route/          assignment.md, setup-server, check-server, solve-server
  03-token-and-cost/             ...
  04-troubleshoot-admin-api/     ...
  05-cost-policies/              ...
  06-mcp-through-gateway/        ...
  07-analyze-the-fleet/          ...
  08-whats-next/                 assignment.md (+ check)
```

`track.yml` `id`/`checksum` and each challenge's `id` are assigned by
`instruqt track push`; the design uses placeholders until first push.
Pushing to Instruqt is a separate, explicitly-authorized step (requires the
`instruqt` CLI + auth) and is **not** part of authoring the files.

## Verification Strategy

- `check-server` scripts verify by observable state: process running, port
  responding, `config_dump` containing expected routes/policies, SQL returning
  rows — never by trusting that a step "should" have run.
- Real OpenAI calls use small capped `max_tokens` to bound cost and latency.
- All config writes happen in `solve-server` from a canonical heredoc so the
  track is reproducible and skip-safe.

## Open Items — RESOLVED during research (tested against the real binary + schema)

1. **AI-backend / policy YAML — RESOLVED & verified.** Exact, validated keypaths
   are captured in the implementation plan:
   - AI backend: `backends[].ai.{name, provider.openAI.{model?}}`; key via route
     policy `backendAuth.key: "$OPENAI_API_KEY"`.
   - Cost: `config.modelCatalog: [{file}]`, catalog rates are strings (USD/1M).
     Verified cost in logs (`agw.ai.usage.cost.total`) and metrics on `:15020`.
   - Token budget: route policy `localRateLimit[{maxTokens,tokensPerFill,
     fillInterval,type:tokens}]` + `ai: {}` → verified HTTP 429 on overflow.
   - Model restriction: `ai.overrides.model` → verified it pins gpt-4o→gpt-4o-mini.
2. **MCP backend — RESOLVED.** `backends[].mcp.targets[].{name, stdio:{cmd,args}}`
   on a `/mcp`-prefixed route; combined LLM+MCP config validates. (stdio target;
   Streamable-HTTP target also available via `mcp:` target form if needed.)
3. **Tooling — RESOLVED.** Plan installs the binary + `openai` CLI + `uv`; the
   generator needs Python ≥3.11 so it runs via `uv run` (not bare `python3`).
4. **Install version — RESOLVED.** `--version` flag confirmed; plan pins `1.3.1`.

> Correction vs. earlier draft: metrics/Prometheus is on **`:15020/metrics`**, not
> `:15000`. `:15000` serves `/config_dump` and `/ui` only.

## Status

> **This is the original design doc (point-in-time).** The shipped track has
> since evolved — see the track `README.md` for the current, authoritative
> state. Notable changes since this design: the "Production Scale" challenge was
> removed; a visual intro was added to "The Blind Spot"; the admin UI is exposed
> via `config.adminAddr: 0.0.0.0:15000` and surfaced as a lab tab; and three
> cost-optimization challenges were added — **Cost-Aware Routing** (header-based,
> C-8), **Tag & Slice Spend** (CEL on `llm.cost.total`, C-9), and **Per-Team
> Virtual Keys**. The track is now **10 challenges** and passes `instruqt track
> test` 10/10 on a real VM.

Implementation plan written and verified:
`docs/superpowers/plans/2026-06-22-ai-cost-token-workshop.md`. Every config, the
mock-data generator, and all analysis queries were executed locally against the
real agentgateway binary and a generated dataset before the plan was finalized.
```
