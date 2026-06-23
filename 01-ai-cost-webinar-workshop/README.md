# AI Cost & Token Spend — Agentgateway OSS (Instruqt track)

A hands-on cost-optimization workshop for platform / FinOps / SecOps teams. It
frames an enterprise story — *Finance forwarded a surprise AI bill; who spent
it?* — and walks through tracking, troubleshooting, and **governing** LLM + MCP
token spend with **standalone Agentgateway OSS v1.3.1** (the single binary, no
Kubernetes).

**~25–30 min · 10 challenges · real OpenAI traffic + a generated fleet dataset.**

## Challenges

| # | Challenge | What you do |
|---|-----------|-------------|
| 1 | **The Blind Spot** | Visual intro: how tokens are spent, what inflates the bill, why dashboards fail (read-only). |
| 2 | **Stand Up the Gateway** | Install the binary, route OpenAI through `:4000`. |
| 3 | **The Cost of Every Request** | Model cost catalog → live token + USD cost in logs; gpt-4o vs gpt-4o-mini (~17×). |
| 4 | **Day-2 Operations** | Troubleshoot via the admin API on `:15000` (`config_dump`, UI) + metrics on `:15020`. |
| 5 | **Governance: Budgets & Model Pin** | `localRateLimit` token budget (→429) + approved-model `overrides`. *(Mike's C-10)* |
| 6 | **Cost-Aware Routing** | Header-based routing: `x-priority: high` → gpt-4o, default → gpt-4o-mini. *(C-8)* |
| 7 | **Tag & Slice Spend** | CEL on `llm.cost.total` → custom log fields + `cost_tier` metric label. *(C-9)* |
| 8 | **MCP Is Spend Too** | Proxy an MCP server on the same gateway; why tool calls cost tokens. |
| 9 | **Per-Team Virtual Keys** | Listener `apiKey` gate (strict) + per-key metadata; real provider key stays hidden. |
| 10 | **Answer the CFO** | Fleet-scale cost analysis (spend by team/user/model/workload) over a generated SQLite dataset. |

The config in `/root/config.yaml` **evolves across challenges** — each `solve`
writes a complete, validated config and restarts the gateway, so a learner can
skip ahead from any point.

From Challenge 3 on, `config.database` (`sqlite:///root/data/data.db`) makes the
gateway **persist every request** to a `request_logs` table — the same schema the
mock generator uses, so Challenge 10 first queries the learner's *real* captured
calls, then scales up with the generated fleet dataset.

## Layout

- `config.yml` — single VM, `OPENAI_API_KEY` secret.
- `track.yml` — track metadata (`id`/`checksum` assigned on push).
- `track_scripts/setup-server` — installs the agentgateway binary + the `openai`
  CLI + `uv` + `sqlite3`, and **embeds** the cost catalog and request-log
  generator onto the VM (Instruqt assets are markdown URLs, not VM files).
- `assets/gen-mock-logs.py` — request-log generator (run via `uv run`; needs Python ≥3.11).
- `assets/costs/catalog.json` — model cost catalog (USD per 1M tokens).
- `assets/blind-spot-{before,after}.{svg,png}` — Challenge 1 intro diagrams.
- `NN-*/` — per-challenge `assignment.md`, `setup-server`, `check-server`, `solve-server`.

## Ports

- `4000` — OpenAI-compatible LLM API (clients point here).
- `3000` — MCP endpoint (separate bind; `binds`, `llm`, and `mcp` must use unique ports).
- `15000` — admin API + UI (`/config_dump`, `/ui`). Bound to `0.0.0.0` via
  `config.adminAddr` so the lab's **Agentgateway UI** tab can reach it.
- `15020` — Prometheus metrics (`/metrics`).

## Cost controls demonstrated (vs. Mike's k8s cost-optimization demos)

| Demo | k8s (AdminTurnedDevOps) | This track (standalone) |
|------|--------------------------|--------------------------|
| **C-8** Cost-aware routing | HTTPRoute header match → premium backend | Route `matches.headers` `x-priority: high` → `ai.overrides.model: gpt-4o` |
| **C-9** CEL cost policy | `rawConfig` logging/metrics fields | `config.logging.fields.add` / `config.metrics.fields.add` on `llm.cost.total` |
| **C-10** Budget enforcement | `traffic.rateLimit.local.tokens` | `policies.localRateLimit` `type: tokens` |

## Verification

All configs are validated with `agentgateway --validate-only`, and the full
track passes `instruqt track test` (setup → solve → check) **10/10 on a real
VM**, including live OpenAI calls (cost tracking, model-pin, 429 budgets,
header routing, CEL cost tags, virtual-key 401/200, and the SQL analysis).

## Publish

```bash
cd 01-ai-cost-webinar-workshop
instruqt track push                        # publish (assigns id/checksum)
instruqt track test ai-cost-token-spend-agentgateway --skip-fail-check
```

> `--skip-fail-check` is needed because Challenge 1 is read-only (its check
> always passes, so the "check should fail before solve" assertion is skipped).

**Live track:** https://play.instruqt.com/manage/soloio/tracks/ai-cost-token-spend-agentgateway

Design + plan history: `docs/superpowers/specs/` and `docs/superpowers/plans/`.
