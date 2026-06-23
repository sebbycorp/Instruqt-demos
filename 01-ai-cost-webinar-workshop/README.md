# AI Cost & Token Spend — Agentgateway OSS (Instruqt track)

A ~15-minute hands-on workshop for platform / FinOps / SecOps teams. It frames an
enterprise story — *Finance forwarded a surprise AI bill; who spent it?* — and
walks through tracking, troubleshooting, and governing LLM + MCP token spend with
**standalone Agentgateway OSS v1.3.1**.

## Challenges

1. **The Blind Spot** — how tokens are calculated and what LLM features cost (read-only).
2. **Stand Up the Gateway** — install the binary, route OpenAI through `:4000`.
3. **The Cost of Every Request** — model cost catalog → live token + USD cost in logs.
4. **Day-2 Operations** — troubleshoot via the admin API on `:15000` (+ metrics on `:15020`).
5. **Governance** — token-budget rate limit (→429) and approved-model pin.
6. **MCP Is Spend Too** — proxy an MCP server on the same gateway.
7. **Answer the CFO** — fleet-scale cost analysis over a generated SQLite dataset.
8. **Production Scale** — guardrails, virtual keys, tracing, Enterprise (read-only).

## Layout

- `config.yml` — single VM, `OPENAI_API_KEY` secret.
- `track.yml` — track metadata (`id`/`checksum` assigned on first push).
- `track_scripts/setup-server` — installs the binary + `openai` CLI + `uv`, stages assets.
- `assets/gen-mock-logs.py` — request-log generator (run via `uv run`; needs Python ≥3.11).
- `assets/costs/catalog.json` — model cost catalog (USD per 1M tokens).
- `NN-*/` — per-challenge `assignment.md`, `setup-server`, `check-server`, `solve-server`.

## Ports

- `4000` — OpenAI-compatible API (clients point here).
- `15000` — admin API + UI (`/config_dump`, `/ui`), localhost-only.
- `15020` — Prometheus metrics (`/metrics`).

## Verification done locally

All configs were validated with `agentgateway --validate-only`, and the cost
tracking, token-budget 429, model-pin override, dual LLM+MCP routing, and the
mock-data SQL analysis were all run end-to-end against the real binary + live
OpenAI calls before commit.

## Push to Instruqt

This authors the files only. To publish (assigns `id`/`checksum`):

```bash
cd 01-ai-cost-webinar-workshop
instruqt track push        # requires the instruqt CLI + auth
```

Design + plan: `docs/superpowers/specs/` and `docs/superpowers/plans/`.
