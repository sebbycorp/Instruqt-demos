# Agentgateway in Docker — Instruqt Track Design

**Date:** 2026-06-24
**Author:** Sebastian Maniak
**Track dir:** `00-agentgateway-tokenomic/`
**Status:** Approved design — ready for implementation plan

---

## 1. Goal & Premise

Teach a customer to run **Agentgateway as a standalone Docker container**, turn it
into a control point for both **LLM** and **MCP tool** traffic, and then **analyze
the traffic** the gateway has already captured.

Narrative arc:

1. **Run the gateway in Docker** with a minimal config — connect to it, see the UI.
2. **Add an LLM** — it becomes an OpenAI-compatible proxy; cost tracking is automatic.
3. **Add a separate everything-MCP server** over the network — LLM + tools, one chokepoint.
4. **Analyze the mock traffic already sent** — cost/tokenomics + usage, via SQL and the UI.

This differs from the existing `01-ai-cost-webinar-workshop` (which runs the raw
binary). Here **everything is Docker**: the gateway is a container, the MCP server is
a *separate* container, and they talk over a shared Docker network.

Design priorities (per user): **diagrams throughout**, and **maximally easy to
execute and understand** — every lab opens with a diagram of what's being built,
a one-line "what you'll see," and pure copy-paste command blocks.

## 2. Audience & Success

- **Audience:** customers/SEs new to Agentgateway; comfortable with a terminal and Docker basics. No Kubernetes.
- **Success:** by the end, the learner has a running gateway proxying OpenAI + an MCP server, and can answer cost/usage questions ("which model costs the most?", "who are the top users?") from real-looking traffic.
- **Time budget:** ~30–45 min. `timelimit: 3600`, `idle_timeout: 1800`.

## 3. Sandbox (`config.yml`)

```yaml
version: "3"
virtualmachines:
- name: server
  image: solo-test-236622/workshop-instruqt-kgateway-20251023   # already has Docker (runs k3d)
  shell: /usr/bin/bash
  machine_type: n1-standard-4
secrets:
- name: OPENAI_API_KEY
```

No binary install. Docker is present on this image (it backs k3d); `track_scripts/setup-server`
verifies `docker info` and starts the daemon if needed.

## 4. Docker Topology

```
docker network create agw          # shared user-defined network (name resolution)

# Gateway container
docker run -d --name agentgateway --network agw \
  -v /root/agentgateway:/config \
  -p 4000:4000 -p 3000:3000 -p 15000:15000 \
  -e ADMIN_ADDR=0.0.0.0:15000 \
  -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:<pinned-tag> -f /config/config.yaml

# MCP container (added in Lab 03)
docker run -d --name mcp-everything --network agw \
  -e PORT=8080 -p 8080:8080 \
  node:20-alpine \
  npx -y @modelcontextprotocol/server-everything streamableHttp
```

Key facts (verified against agentgateway docs, 2026-06-24):

- **Image:** `cr.agentgateway.dev/agentgateway` (official registry). UI ships in-container on `:15000/ui`.
- `-e ADMIN_ADDR=0.0.0.0:15000` is **required** so the UI/admin binds on all interfaces and the Instruqt service tab can reach it (default binds localhost).
- **Ports:** `:4000` LLM API, `:3000` MCP, `:15000` admin + UI.
- **Config + assets are mounted under `/config`**, so in-container paths are:
  - cost catalog: `/config/costs/base-costs.json`
  - request DB: `sqlite:////config/data/data.db` (4 slashes = absolute path)
- **Remote MCP over HTTP:** `mcp.targets[].mcp.host: http://mcp-everything:8080/mcp/` (trailing slash required; gateway resolves `mcp-everything` via the `agw` network).
- **Live calls go through the Terminal** (`localhost:4000` / `:3000`, published ports). The **UI service tab** reaches `:15000`. Same pattern as existing tracks — the browser can't reach `:4000/:3000` through the hosted lab.
- **Image tag:** pin to a known-good tag (target `:v1.3.1` to match the binary version used elsewhere). Build step verifies the tag resolves; falls back to ghcr.io mirror or `latest` if it doesn't.

### Helper in `~/.bashrc`

```bash
agw-restart() { docker restart agentgateway >/dev/null 2>&1; sleep 3; }
```

(Config is bind-mounted, so a restart re-reads `config.yaml` after edits.)

## 5. Labs

Each lab dir contains `assignment.md`, `setup-server`, `check-server`, `solve-server`.
Every `assignment.md` starts with: its **architecture diagram** (PNG), a **"What we're
building"** line, a **"What you'll see"** line, then numbered copy-paste steps.

### Lab 01 — `run-gateway-docker`
- **Build:** pull + `docker run` the gateway with a **minimal** mounted config (admin UI + cost catalog + request DB, **no models yet**). Create the `agw` network first.
- **Connect:** open the **Agentgateway UI** tab (`:15000/ui`); `curl :15000/config_dump`.
- **Teach:** anatomy of the run command — `-v` mount, `-p` publishes, `-e ADMIN_ADDR`.
- **Opens with the full-journey overview diagram** (the destination) + the Lab-01 diagram (just the gateway box).
- **Check:** container `agentgateway` is running AND `:15000/config_dump` answers.

### Lab 02 — `add-an-llm`
- **Build:** add an OpenAI model — via the **UI** (Models → Add model, API key = env var `OPENAI_API_KEY`) **or** an `llm.models` YAML block in the Editor; `agw-restart`.
- **Connect:** `POST localhost:4000/v1/chat/completions` with `model: openai/gpt-4.1-nano`.
- **Teach:** OpenAI-compatible proxy; the key never leaves the gateway; the bundled cost catalog already priced the call.
- **Diagram:** gateway → OpenAI added.
- **Check:** a test call to `:4000` returns HTTP 200.

### Lab 03 — `add-everything-mcp`
- **Build:** `docker run` the **separate** `mcp-everything` container on the `agw` network (streamable HTTP, port 8080); add a top-level `mcp:` block (`port: 3000`, target `host: http://mcp-everything:8080/mcp/`); `agw-restart`.
- **Connect:** MCP handshake on `:3000` (`initialize` → `notifications/initialized` → `tools/list`).
- **Teach:** tool schemas + results become **input tokens** on every turn — MCP is spend too; now it shares one control point, one log, one config.
- **Diagram:** two containers on a network, gateway fronting both LLM and MCP.
- **Check:** `mcp-everything` container running; `:3000` returns 200; `tools/list` returns ≥1 tool.

### Lab 04 — `analyze-the-traffic`
- **Setup pre-seeds** ~5,000 rows of a week of realistic LLM traffic into the gateway's request DB (`/root/agentgateway/data/data.db`) using `gen-mock-logs.py` — so "the gateway has already seen a week of traffic." The learner's own Lab-02/03 calls are in there too.
- **Analyze (guided, copy-paste SQL with expected-shape answers shown inline):**
  - **Cost / tokenomics:** total spend; spend by provider; top-spend models; cost per request; total tokens.
  - **Usage & traffic:** request volume; top models by call count; busiest users and groups.
  - Plus a pointer to the **UI data pages** for the same view visually.
- **Teach:** the gateway turns invisible AI spend into queryable, attributable data.
- **Diagram:** the request DB / logs flowing out of the gateway into analysis.
- **Check:** **lenient** — verify the DB exists and has rows (`request_logs` count > 0); completion marker. Queries are for learning, not gated on exact numbers.

## 6. Diagrams

Authored as clean **SVG**, rendered to **PNG** with `rsvg-convert` (confirmed available).
`assignment.md` embeds the PNG (matches existing tracks). Stored in `00-agentgateway-tokenomic/assets/`.

- `diagram-journey.{svg,png}` — full 4-step overview (shown in Lab 01).
- `diagram-01-gateway.{svg,png}` — single gateway container + UI/admin + ports.
- `diagram-02-llm.{svg,png}` — gateway → OpenAI; client points at :4000.
- `diagram-03-mcp.{svg,png}` — gateway + separate MCP container on `agw` network; :4000 LLM, :3000 MCP.
- `diagram-04-analyze.{svg,png}` — traffic → request DB → SQL/UI analysis (cost + usage).

Visual style: simple labeled boxes/arrows, consistent palette, port labels on every
edge, container names visible — so the picture *is* the mental model of the commands.

## 7. Reused Assets (from `01-ai-cost-webinar-workshop`)

- **`assets/base-costs.json`** — the model cost catalog (prices every call).
- **`gen-mock-logs.py`** — SQLite request-log generator; writes the `request_logs`
  schema the gateway uses. Pointed at the gateway's mounted DB path.
- Patterns for `track.yml`, `track_scripts/cleanup-server`, `README.md`.

Adaptation: the DB path moves to `/root/agentgateway/data/data.db` (the mounted
`/config/data/data.db` inside the container). The catalog moves to
`/root/agentgateway/costs/base-costs.json`.

## 8. File Layout

```
00-agentgateway-tokenomic/
  track.yml
  config.yml
  README.md
  assets/
    base-costs.json
    gen-mock-logs.py
    diagram-journey.svg / .png
    diagram-01-gateway.svg / .png
    diagram-02-llm.svg / .png
    diagram-03-mcp.svg / .png
    diagram-04-analyze.svg / .png
  track_scripts/
    setup-server        # verify docker; create agw network; provision /root/agentgateway/{config.yaml,costs,data}; stage generator; bashrc helper
    cleanup-server
  01-run-gateway-docker/   { assignment.md, setup-server, check-server, solve-server }
  02-add-an-llm/           { assignment.md, setup-server, check-server, solve-server }
  03-add-everything-mcp/   { assignment.md, setup-server, check-server, solve-server }
  04-analyze-the-traffic/  { assignment.md, setup-server, check-server, solve-server }
```

## 9. Risks & Mitigations

- **Docker image tag drift** — verify the pinned tag resolves at build; fall back to ghcr.io mirror / `latest`. Pull in `track_scripts/setup-server` so labs aren't waiting on a cold pull.
- **everything-MCP HTTP mode** — confirm `npx … server-everything streamableHttp` honors `PORT` and serves `/mcp`. If the npx-in-`node:20-alpine` path is flaky, fall back to the official `mcp/everything` image with a command override, or pre-pull/warm in setup.
- **DB write permissions** — container runs as a user; ensure `/root/agentgateway/data` is writable and the generator-created DB is readable by that user (chmod/`--user` alignment). Verify the gateway can append to the pre-seeded DB.
- **`sqlite:////` path** — 4 slashes needed for absolute paths; validate config with `--validate-only` equivalent (or a dry `docker run`).
- **UI reachability** — `ADMIN_ADDR=0.0.0.0:15000` is mandatory; without it the service tab 502s.

## 10. Out of Scope

Auth/virtual keys, rate limiting/budgets, cost-aware routing, multiple providers,
failover, Kubernetes. (Available as a follow-on track; "what's next" pointer in the README.)
