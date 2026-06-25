# Agentgateway Tokenomics: See and Govern Your AI Token Spend (Instruqt track)

A 4-lab standalone Agentgateway lab. The learner stands up the gateway as their AI
control point, turns it into an LLM-native proxy (OpenAI as the example; Anthropic,
Gemini, Bedrock, etc. work the same way), attaches a separate
everything-MCP server over HTTP, and analyzes a week of pre-seeded token spend.
(Implementation detail: the gateway and MCP server run as Docker containers — this
is plumbing, not the lesson.)

## Labs
1. **run-gateway-docker** — `docker run` the gateway with a minimal config; connect to the UI.
2. **add-an-llm** — add an OpenAI model (LLM-native proxy); first call through `:4000`; UI Logs + optional user/group identity.
3. **cost-of-every-request** — per-call USD cost, premium-vs-cheap model, catalog overrides, UI Costs.
4. **govern-the-spend** — `localRateLimit` (`type: tokens`) token budget; over-budget → 429.
5. **cost-aware-routing** — cheap by default, premium only on `x-priority: high`.
6. **add-everything-mcp** — separate MCP container on the `agw` network; tools on `:3000`.
7. **analyze-the-traffic** — query `request_logs` for cost + usage (week of seeded traffic).
8. **virtual-keys** — per-team `apiKey` gate (`mode: strict`); no key → 401, team key → 200.

## How it runs
- Gateway: `cr.agentgateway.dev/agentgateway:v1.3.1`, config bind-mounted at `/config`, ports 4000/3000/15000.
- MCP: `node:20-alpine` running `server-everything streamableHttp` on the shared `agw` network.
- Mock traffic: `assets/gen-mock-logs.py` seeds `request_logs` at track setup (run via `uv`, Python ≥3.11).
- `OPENAI_API_KEY` is an Instruqt secret, passed to the container via `-e`; never exposed to the learner.

## Develop / test
```bash
instruqt track validate
instruqt track push                                              # assigns id/checksum
instruqt track test agentgateway-tokenomic --skip-fail-check     # setup -> solve -> check on a real VM
```

**Live track:** https://play.instruqt.com/manage/soloio/tracks/agentgateway-tokenomic
