# AgentGateway Local Cost Webinar — Design

**Date:** 2026-06-24
**Track slug:** `00-agw-local-cost-webinar`
**Format:** Instruqt webinar, 5 labs, ~45–60 min
**Theme:** Stand up agentgateway OSS **in Docker** on a single VM, front OpenAI + an MCP
server through it, and make every dollar of LLM **and** MCP spend visible and governable.

This is the Docker-local sibling of `01-ai-cost-webinar-workshop` (which runs the
agentgateway *binary*). Same cost story, containerized.

## Goals

- Lab 1 stands up the entire local stack in Docker: gateway container + MCP server
  container + OpenAI backend + cost catalog + sqlite request log + UI.
- Labs 2–5 use that running stack to tell the cost story: see per-request cost,
  govern spend, route by cost, and prove MCP traffic is spend too — ending on a
  "answer the CFO" summary.
- Every lab is **self-contained and resumable**: each lab's `setup-server` seeds the
  config to the previous lab's solved end-state; `solve-server` writes this lab's
  end-state. A learner can jump to any lab.

## Architecture

Single Instruqt VM (`solo-test-.../workshop-instruqt-kgateway-*`, `n1-standard-4`),
Docker preinstalled. One user-defined bridge network `agw-net` so containers resolve
each other by name.

**Containers (created in Lab 1, reused by all labs):**

| Container | Image | Ports (host) | Role |
|-----------|-------|--------------|------|
| `agentgateway` | `cr.agentgateway.dev/agentgateway:v1.3.1` | 4000 (LLM), 3000 (MCP), 15000 (UI/admin) | the gateway |
| `mcp-everything` | `node:22-alpine` running `npx -y @modelcontextprotocol/server-everything streamableHttp` | internal `:3001` only | MCP tool server |

`mcp-everything` listens on `PORT=3001` at path `/mcp` (verified from package source).
The gateway reaches it over `agw-net` via `http://mcp-everything:3001/mcp`.

**Volumes / mounts into the gateway container:**
- `/root/config.yaml` → `/config.yaml` (the config the learner edits)
- `/root/costs` → `/costs` (cost catalog JSON, layered)
- `/root/data` → `/data` (sqlite request log `data.db`)

**Secrets:** `OPENAI_API_KEY` (Instruqt secret), passed into the gateway container
with `-e OPENAI_API_KEY`. The config references it as `apiKey: "$OPENAI_API_KEY"`
(agentgateway expands `$VAR` from its process env).

**LLM backend:** OpenAI only. `gpt-4.1-nano` (cheap) is the default route; `gpt-4.1`
(premium) is introduced in Lab 4 for the routing/cost-comparison story.

## Helper functions (in `~/.bashrc`, written by track setup)

- `agw-up` — `docker rm -f agentgateway`, then `docker run -d` the gateway with the
  mounts/ports/network above. Used to (re)start after a config edit.
- `agw-restart` — alias for `agw-up`.
- `agw-validate` — `docker run --rm -e OPENAI_API_KEY=... ... -f /config.yaml --validate-only`.
- `agw-logs` — `docker logs -f agentgateway`.

## Per-lab plan

**Lab 1 — Stand Up the Gateway.** Learner writes `/root/config.yaml` (config block
with adminAddr/database/modelCatalog; `llm` on :4000 with `openai/*`→gpt-4.1-nano;
`mcp` on :3000 targeting `http://mcp-everything:3001/mcp`), then `agw-up`. Verifies a
chat completion on :4000 (200) and an MCP `initialize` on :3000 (200), and that the
UI answers on :15000. *Check:* gateway container running; `:4000` chat 200; `:3000`
initialize 200.

**Lab 2 — The Cost of Every Request.** Send a handful of completions, then query
`/root/data/data.db` `request_logs` for `gen_ai_request_model`, tokens, and `cost`;
open the UI **Costs** tab. *Check:* db has ≥1 row with `cost > 0`.

**Lab 3 — Govern Spend.** Add `llm.policies.localRateLimit` (a small per-window token
budget) and pin the model. Re-up, then prove a burst returns **429** once the budget
is spent. *Check:* config has a token `localRateLimit`; a loop of calls eventually
returns 429.

**Lab 4 — Cost-Aware Routing.** Add an exact-match premium model `openai/gpt-4.1`
ahead of the `openai/*`→nano fallback. Call both; compare logged cost. *Check:* a
request for `openai/gpt-4.1` logs a gpt-4.1 row; a request for another name logs
nano; premium cost > cheap cost.

**Lab 5 — MCP Is Spend Too + Answer the CFO.** Use the MCP endpoint to list tools and
see that tool schemas are input tokens; make a tool-using call; then run a single
"CFO summary" SQL query: total spend, spend by model, request count. *Check:* MCP
tools list returns tools; the summary query runs and returns totals.

## File layout

```
00-agw-local-cost-webinar/
  track.yml
  config.yml
  track_scripts/setup-server      # network + mcp-everything + catalog + bashrc helpers
  track_scripts/cleanup-server     # docker rm -f containers, network rm
  assets/costs/base-costs.json     # reused cost catalog (USD per 1M tokens)
  01-stand-up-the-gateway/{assignment.md,setup-server,check-server,solve-server}
  02-cost-of-every-request/{...}
  03-govern-spend/{...}
  04-cost-aware-routing/{...}
  05-mcp-is-spend-too/{...}
  docs/superpowers/specs/2026-06-24-agw-local-cost-webinar-design.md
```

## Conventions (matched from existing tracks)

- `assignment.md` = YAML front matter (slug/id/type/title/teaser/notes/tabs) + markdown.
- Tabs per lab: Terminal, Editor (`/root`), Agentgateway UI (service, port 15000, path `/ui`).
- `check-server` uses `fail-message "..."` for learner-facing failures, exit 1 on fail.
- `setup-server` seeds the starting config; `solve-server` writes the solved end-state.
- IDs in front matter are placeholder strings; Instruqt regenerates/assigns on push.

## Non-goals (YAGNI)

- No Kubernetes, no Helm, no enterprise UI/auth (covered by other tracks).
- No multi-provider (Anthropic/Bedrock) — OpenAI's two tiers carry the cost story.
- No custom gateway image — MCP runs as its own container over HTTP instead.
