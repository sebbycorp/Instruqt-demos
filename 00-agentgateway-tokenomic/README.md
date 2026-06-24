# Agentgateway in Docker: Govern Your AI Token Spend (Instruqt track)

A 4-lab standalone track. The learner runs Agentgateway as a Docker container,
turns it into an OpenAI-compatible proxy, attaches a separate everything-MCP
container over HTTP, and analyzes a week of pre-seeded token spend.

## Labs
1. **run-gateway-docker** — `docker run` the gateway with a minimal config; connect to the UI.
2. **add-an-llm** — add an OpenAI model; first call through `:4000`.
3. **add-everything-mcp** — separate MCP container on the `agw` network; tools on `:3000`.
4. **analyze-the-traffic** — query `request_logs` for cost + usage.

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
