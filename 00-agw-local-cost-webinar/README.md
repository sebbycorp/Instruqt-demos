# AgentGateway Local — AI Cost Control Plane in Docker (Instruqt track)

A hands-on webinar that stands up **Agentgateway OSS entirely in Docker** on one VM —
no Kubernetes — and makes every dollar of LLM **and** MCP spend visible and governable.
It's the Docker-local sibling of `01-ai-cost-webinar-workshop` (which runs the binary).

**~45–60 min · 5 labs · real OpenAI traffic + a live MCP tool server.**

![Architecture](assets/diagram-architecture.png)

## Labs

| # | Lab | What you do |
|---|-----|-------------|
| 1 | **Stand Up the Gateway in Docker** | `docker run` the gateway; OpenAI on `:4000`, MCP on `:3000`, UI on `:15000`, cost tracking on. |
| 2 | **The Cost of Every Request** | Generate traffic; read per-request token + USD cost from the SQLite `request_logs` and the UI. |
| 3 | **Govern the Spend** | Add a `localRateLimit` token budget (→ `429`) and keep the model pinned. |
| 4 | **Cost-Aware Routing** | Header routing: `x-priority: high` → `gpt-4.1`, default → `gpt-4.1-nano` (~20× cheaper). |
| 5 | **MCP Is Spend Too → Answer the CFO** | List MCP tools (schemas = input tokens), call one, then total the whole bill by model. |

Each lab is **self-contained and resumable**: its `setup-server` seeds `/root/config.yaml`
to the previous lab's solved end-state, and `solve-server` writes this lab's end-state.

## Architecture

Everything runs on a single Instruqt VM with Docker, on a user-defined bridge network
`agw-net` so containers resolve each other by name.

| Container | Image | Role |
|-----------|-------|------|
| `agentgateway` | `cr.agentgateway.dev/agentgateway:v1.3.1` | the gateway — LLM `:4000`, MCP `:3000`, admin UI `:15000` |
| `mcp-everything` | `node:22-alpine` running `npx -y @modelcontextprotocol/server-everything streamableHttp` | MCP tool server (internal `:3001`, path `/mcp`) |

The gateway reaches the MCP server over `agw-net` at `http://mcp-everything:3001/mcp`
(an HTTP target — the gateway image has no Node, so the MCP server is its own container
rather than a stdio `npx` inside the gateway).

**Mounts into the gateway container:** `/root/config.yaml` → `/config.yaml`,
`/root/costs` → `/costs` (cost catalog), `/root/data` → `/data` (SQLite request log).

## Ports

- `4000` — OpenAI-compatible LLM API (`llm.port`; clients point here).
- `3000` — MCP endpoint (`mcp.port`; `llm` and `mcp` must use unique ports).
- `15000` — admin API + UI (`/ui`), bound to `0.0.0.0` via `config.adminAddr` so the
  lab's **Agentgateway UI** tab can reach it.

## Helper commands (loaded in the terminal + used by lab scripts)

Defined in `/root/agw-helpers.sh` (sourced by `~/.bashrc` and by the check/solve scripts):

- `agw-validate` — `--validate-only` against `/root/config.yaml`.
- `agw-up` / `agw-restart` — (re)create the gateway container from the config.
- `agw-logs` — tail the gateway container logs.
- `agw-down` — stop the gateway container.

## Layout

- `config.yml` — single VM, `OPENAI_API_KEY` secret.
- `track.yml` — track metadata (`id`/`checksum` assigned on push).
- `track_scripts/setup-server` — pulls images, creates `agw-net`, starts `mcp-everything`,
  provisions `/root/costs/base-costs.json`, writes the `agw-*` helpers.
- `track_scripts/cleanup-server` — removes both containers and the network.
- `assets/costs/base-costs.json` — model cost catalog (USD per 1M tokens).
- `assets/diagram-architecture.{svg,png}` — the diagram above.
- `NN-*/` — per-lab `assignment.md`, `setup-server`, `check-server`, `solve-server`.
- `docs/superpowers/specs/` — the design spec.

## Cost story demonstrated

- **Per-request cost** from a cost catalog + SQLite log — no month-end surprise.
- **Token budgets** (`localRateLimit`, `type: tokens`) that fail fast with `429`.
- **Cost-aware routing** — cheap by default, premium only on opt-in header.
- **MCP traffic** under the same log and the same governance as LLM traffic.
