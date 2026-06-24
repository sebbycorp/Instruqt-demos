# Agentgateway Tokenomic (Docker) Track — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a 4-lab standalone Instruqt track that teaches customers to run Agentgateway as a Docker container, make it an OpenAI-compatible proxy, attach a *separate* everything-MCP container over HTTP, and analyze a week of pre-seeded LLM traffic for cost + usage.

**Architecture:** All work lives under `00-agentgateway-tokenomic/`. The gateway runs as a Docker container (`cr.agentgateway.dev/agentgateway:v1.3.1`) with a bind-mounted `/config` dir (config.yaml + cost catalog + SQLite request DB). A second container (`node:20-alpine` running `server-everything streamableHttp`) is the MCP backend, reachable over a shared `agw` Docker network. The mock-log generator pre-seeds the gateway's own `request_logs` DB at track setup. Each lab = `assignment.md` + `setup-server` + `check-server` + `solve-server`.

**Tech Stack:** Instruqt (track.yml/config.yml + bash lifecycle scripts), Docker, Agentgateway v1.3.x standalone config YAML, SQLite, `uv` (Python ≥3.11) for the generator, `rsvg-convert` for SVG→PNG diagrams.

**Reference spec:** `docs/superpowers/specs/2026-06-24-agentgateway-docker-tokenomic-track-design.md` (includes a local smoke-test validation section with the exact working commands — all reproduced below).

**Slug:** `agentgateway-tokenomic` (play link `https://play.instruqt.com/soloio/tracks/agentgateway-tokenomic`).

---

## File Structure

```
00-agentgateway-tokenomic/
  track.yml                         # Task 1 — metadata, slug, tags
  config.yml                        # Task 1 — VM sandbox + OPENAI_API_KEY secret
  README.md                         # Task 8 — authoring/test notes
  assets/
    base-costs.json                 # Task 1 — copied verbatim from 01-ai-cost-webinar-workshop
    gen-mock-logs.py                # Task 1 — copied verbatim from 01-ai-cost-webinar-workshop
    diagram-journey.svg/.png        # Task 3
    diagram-01-gateway.svg/.png     # Task 3
    diagram-02-llm.svg/.png         # Task 3
    diagram-03-mcp.svg/.png         # Task 3
    diagram-04-analyze.svg/.png     # Task 3
  track_scripts/
    setup-server                    # Task 2 — docker check, network, /root/agentgateway provisioning, DB pre-seed, bashrc helper
    cleanup-server                  # Task 2 — remove containers + network
  01-run-gateway-docker/            # Task 4
    assignment.md  setup-server  check-server  solve-server
  02-add-an-llm/                    # Task 5
    assignment.md  setup-server  check-server  solve-server
  03-add-everything-mcp/            # Task 6
    assignment.md  setup-server  check-server  solve-server
  04-analyze-the-traffic/           # Task 7
    assignment.md  setup-server  check-server  solve-server
```

**Reference dir** (read-only source for patterns + reused assets): `01-ai-cost-webinar-workshop/`.

**Working dir for all commands below:** repo root `/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos`. Quote the path or `cd` into the track dir as noted.

---

## Validated facts (from smoke test — do not re-derive)

- Image `cr.agentgateway.dev/agentgateway:v1.3.1` runs; UI at `:15000/ui` needs `-e ADMIN_ADDR=0.0.0.0:15000`.
- Ports: `:4000` LLM, `:3000` MCP, `:15000` admin/UI.
- Config bind-mounted at `/config`; DB url `sqlite:////config/data/data.db` (4 slashes); catalog `/config/costs/base-costs.json`.
- LLM model uses `apiKey: "$OPENAI_API_KEY"` (env substitution; container gets `-e OPENAI_API_KEY`).
- MCP target host `http://mcp-everything:8080/mcp/` (trailing slash); MCP container = `node:20-alpine` + `npx -y @modelcontextprotocol/server-everything streamableHttp` with `-e PORT=8080`.
- Gateway's own DB tables are `request_logs` + `request_log_payloads` — generator schema matches; pre-seed BEFORE gateway start.
- Generator MUST run via `uv run` (PEP 723 pins Python ≥3.11).
- `agw-restart` = `docker restart agentgateway`.

---

### Task 1: Scaffold track dir, sandbox config, metadata, reused assets

**Files:**
- Create: `00-agentgateway-tokenomic/config.yml`
- Create: `00-agentgateway-tokenomic/track.yml`
- Create: `00-agentgateway-tokenomic/assets/base-costs.json` (copy)
- Create: `00-agentgateway-tokenomic/assets/gen-mock-logs.py` (copy)

- [ ] **Step 1: Create directory skeleton**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
mkdir -p 00-agentgateway-tokenomic/{assets,track_scripts,01-run-gateway-docker,02-add-an-llm,03-add-everything-mcp,04-analyze-the-traffic}
```
Expected: directories created, no output.

- [ ] **Step 2: Copy the two reused assets verbatim**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
cp 01-ai-cost-webinar-workshop/assets/base-costs.json 00-agentgateway-tokenomic/assets/base-costs.json
cp 01-ai-cost-webinar-workshop/assets/gen-mock-logs.py 00-agentgateway-tokenomic/assets/gen-mock-logs.py
wc -l 00-agentgateway-tokenomic/assets/gen-mock-logs.py 00-agentgateway-tokenomic/assets/base-costs.json
```
Expected: `gen-mock-logs.py` ~698 lines, `base-costs.json` present.

- [ ] **Step 3: Create `config.yml`** (sandbox VM + secret)

Create `00-agentgateway-tokenomic/config.yml`:
```yaml
version: "3"
virtualmachines:
- name: server
  image: solo-test-236622/workshop-instruqt-kgateway-20251023
  shell: /usr/bin/bash
  machine_type: n1-standard-4
secrets:
- name: OPENAI_API_KEY
```

- [ ] **Step 4: Create `track.yml`** (metadata; `id`/`checksum` are assigned by `instruqt track push` — omit them)

Create `00-agentgateway-tokenomic/track.yml`:
```yaml
slug: agentgateway-tokenomic
title: 'Agentgateway in Docker: Govern Your AI Token Spend'
teaser: Run Agentgateway as a Docker container, proxy LLM and MCP traffic through one
  control point, then analyze a week of token spend.
description: |-
  Your AI agents call LLMs and MCP tools directly — no chokepoint, no cost
  visibility, no governance. **Agentgateway** is a single Rust binary you can run
  as a Docker container to put one control point in front of all of it.

  In this hands-on lab you'll:

  - 🐳 Run Agentgateway as a Docker container and connect to its built-in UI
  - 🌐 Add an OpenAI model and turn the gateway into an OpenAI-compatible proxy
  - 🔌 Deploy a separate MCP server and bring tool traffic under the same gateway
  - 📊 Analyze a week of captured traffic — cost by model/provider/user and usage

  **No Kubernetes. No prior Agentgateway experience.** Just Docker basics and a terminal.
icon: ""
tags:
- ai
- llm
- mcp
- docker
- agentgateway
- solo-io
owner: soloio
developers:
- sebastian.maniak@solo.io
idle_timeout: 1800
timelimit: 3600
lab_config:
  sidebar_enabled: true
  feedback_recap_enabled: true
  feedback_tab_enabled: false
  loadingMessages: true
  hideStopButton: false
enhanced_loading: false
```

- [ ] **Step 5: Commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/config.yml 00-agentgateway-tokenomic/track.yml 00-agentgateway-tokenomic/assets/base-costs.json 00-agentgateway-tokenomic/assets/gen-mock-logs.py
git commit -m "Scaffold agentgateway-tokenomic track: config.yml, track.yml, reused assets"
```

---

### Task 2: Track lifecycle scripts (setup-server, cleanup-server)

**Files:**
- Create: `00-agentgateway-tokenomic/track_scripts/setup-server`
- Create: `00-agentgateway-tokenomic/track_scripts/cleanup-server`

The track setup-server runs ONCE before challenges. It: waits for bootstrap, stores the OpenAI key, ensures Docker is up, pre-pulls images, creates the `agw` network, provisions `/root/agentgateway/{config.yaml,costs,data}`, **pre-seeds the request DB**, and installs the `agw-restart` bashrc helper. It does NOT start the gateway (the learner does that in Lab 01).

- [ ] **Step 1: Create `track_scripts/setup-server`**

Create `00-agentgateway-tokenomic/track_scripts/setup-server`:
```bash
#!/bin/bash

# Wait for instruqt bootstrap
until [ -f /opt/instruqt/bootstrap/host-bootstrap-completed ]; do
  sleep 1
done

set -euo pipefail
set-workdir /root

AGW_IMAGE="cr.agentgateway.dev/agentgateway:v1.3.1"
MCP_IMAGE="node:20-alpine"

# --- OpenAI key (Instruqt secret) available to every challenge + the container ---
if [ -n "${OPENAI_API_KEY:-}" ]; then
  echo "export OPENAI_API_KEY=${OPENAI_API_KEY}" >> ~/.bashrc
  echo "${OPENAI_API_KEY}" > /root/.openai-api-key
  echo "=== OpenAI API key stored ==="
else
  echo "=== WARNING: OPENAI_API_KEY not set ==="
fi

# --- Docker must be running (the base image ships Docker for k3d) ---
if ! docker info >/dev/null 2>&1; then
  echo "=== starting dockerd ==="
  systemctl start docker >/dev/null 2>&1 || service docker start >/dev/null 2>&1 || (dockerd >/tmp/dockerd.log 2>&1 &)
  for _ in $(seq 1 30); do docker info >/dev/null 2>&1 && break; sleep 1; done
fi
docker info >/dev/null 2>&1 && echo "docker: ok" || echo "WARNING: docker not available"

# --- Pre-pull images so labs aren't waiting on cold pulls ---
docker pull "$AGW_IMAGE" >/dev/null 2>&1 || echo "WARNING: could not pre-pull $AGW_IMAGE"
docker pull "$MCP_IMAGE" >/dev/null 2>&1 || echo "WARNING: could not pre-pull $MCP_IMAGE"

# --- Shared Docker network for gateway <-> MCP name resolution ---
docker network inspect agw >/dev/null 2>&1 || docker network create agw >/dev/null
echo "network agw: ok"

# --- uv drives the mock-log generator (PEP 723 header pins Python >=3.11) ---
if ! command -v uv >/dev/null 2>&1; then
  curl -LsSf https://astral.sh/uv/install.sh | sh
fi
ln -sf "$HOME/.local/bin/uv" /usr/local/bin/uv 2>/dev/null || true
export PATH="$HOME/.local/bin:$PATH"

# sqlite3 for the analysis lab (defensive; usually present).
command -v sqlite3 >/dev/null 2>&1 || { apt-get update -y >/dev/null 2>&1 && apt-get install -y sqlite3 >/dev/null 2>&1; } || true

# --- Provision the bind-mounted config dir ---
mkdir -p /root/agentgateway/costs /root/agentgateway/data
chmod -R 777 /root/agentgateway

# Cost catalog (Instruqt assets are markdown URLs, not VM files — provision deterministically)
cat > /root/agentgateway/costs/base-costs.json <<'JSON'
__BASE_COSTS_JSON__
JSON

# Mock-log generator
cat > /root/agentgateway/gen-mock-logs.py <<'PYEOF'
__GEN_MOCK_LOGS_PY__
PYEOF
chmod +x /root/agentgateway/gen-mock-logs.py

# Minimal config (no models) — the learner starts the gateway with this in Lab 01.
cat > /root/agentgateway/config.yaml <<'YAML'
# Minimal Agentgateway config — boots the gateway + UI with no models yet.
# Add your OpenAI model in the UI (Models -> Add model) or paste the YAML block from the assignment.
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:////config/data/data.db"
  modelCatalog:
    - file: /config/costs/base-costs.json
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models: []
frontendPolicies:
  http:
    maxBufferSize: 33554432
YAML

# --- Pre-seed a week of traffic into the gateway's own request DB (BEFORE gateway start) ---
uv run --python 3.12 /root/agentgateway/gen-mock-logs.py \
  --output /root/agentgateway/data/data.db --replace --requests 5000 --days 7 --seed 7 \
  || echo "WARNING: mock-log pre-seed failed"
chmod -R 777 /root/agentgateway/data
echo "pre-seeded rows: $(sqlite3 /root/agentgateway/data/data.db 'SELECT COUNT(*) FROM request_logs;' 2>/dev/null || echo MISSING)"

# --- Convenience helper: restart the gateway after config edits ---
cat >> ~/.bashrc <<'RC'
export PATH="$HOME/.local/bin:/usr/local/bin:$PATH"
agw-restart() { docker restart agentgateway >/dev/null 2>&1; sleep 3; }
RC

echo "=== Track setup complete ==="
```

> NOTE: replace `__BASE_COSTS_JSON__` with the full verbatim contents of `00-agentgateway-tokenomic/assets/base-costs.json`, and `__GEN_MOCK_LOGS_PY__` with the full verbatim contents of `00-agentgateway-tokenomic/assets/gen-mock-logs.py`. Do the substitution with this command rather than by hand:
>
> ```bash
> cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
> python3 - <<'PY'
> from pathlib import Path
> s = Path("track_scripts/setup-server").read_text()
> s = s.replace("__BASE_COSTS_JSON__", Path("assets/base-costs.json").read_text().rstrip("\n"))
> s = s.replace("__GEN_MOCK_LOGS_PY__", Path("assets/gen-mock-logs.py").read_text().rstrip("\n"))
> Path("track_scripts/setup-server").write_text(s)
> print("substituted; setup-server is now", len(s), "bytes")
> PY
> ```
> Verify neither placeholder remains: `grep -c '__BASE_COSTS_JSON__\|__GEN_MOCK_LOGS_PY__' track_scripts/setup-server` must print `0`.

- [ ] **Step 2: Create `track_scripts/cleanup-server`**

Create `00-agentgateway-tokenomic/track_scripts/cleanup-server`:
```bash
#!/bin/bash
set -uo pipefail
docker rm -f agentgateway mcp-everything >/dev/null 2>&1 || true
docker network rm agw >/dev/null 2>&1 || true
echo "cleanup complete"
```

- [ ] **Step 3: Make scripts executable**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
chmod +x track_scripts/setup-server track_scripts/cleanup-server
```

- [ ] **Step 4: Verify the generator block is valid Python after substitution**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
bash -n track_scripts/setup-server && echo "setup-server: bash syntax OK"
```
Expected: `setup-server: bash syntax OK` and the earlier grep returned `0`.

- [ ] **Step 5: Commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/track_scripts/
git commit -m "Add track lifecycle scripts: docker setup + DB pre-seed + cleanup"
```

---

### Task 3: Diagrams (5 SVGs → PNGs)

**Files:**
- Create: `00-agentgateway-tokenomic/assets/diagram-journey.svg` (+ `.png`)
- Create: `00-agentgateway-tokenomic/assets/diagram-01-gateway.svg` (+ `.png`)
- Create: `00-agentgateway-tokenomic/assets/diagram-02-llm.svg` (+ `.png`)
- Create: `00-agentgateway-tokenomic/assets/diagram-03-mcp.svg` (+ `.png`)
- Create: `00-agentgateway-tokenomic/assets/diagram-04-analyze.svg` (+ `.png`)

Visual style: white background, 880×320 viewport, rounded boxes, one accent color for the gateway (`#4f46e5`), neutral gray for clients/backends, labeled arrows with port numbers. Keep them simple — the picture must map 1:1 to the commands in each lab.

- [ ] **Step 1: Write `diagram-journey.svg`** (the 4-step overview shown in Lab 01)

Create `00-agentgateway-tokenomic/assets/diagram-journey.svg`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 920 200" font-family="Segoe UI, Arial, sans-serif">
  <rect width="920" height="200" fill="#ffffff"/>
  <g text-anchor="middle">
    <rect x="20"  y="70" width="190" height="70" rx="10" fill="#eef2ff" stroke="#4f46e5"/>
    <text x="115" y="100" font-size="15" font-weight="bold" fill="#3730a3">1 · Run in Docker</text>
    <text x="115" y="122" font-size="12" fill="#475569">gateway + UI</text>
    <rect x="250" y="70" width="190" height="70" rx="10" fill="#eef2ff" stroke="#4f46e5"/>
    <text x="345" y="100" font-size="15" font-weight="bold" fill="#3730a3">2 · Add an LLM</text>
    <text x="345" y="122" font-size="12" fill="#475569">OpenAI proxy :4000</text>
    <rect x="480" y="70" width="190" height="70" rx="10" fill="#eef2ff" stroke="#4f46e5"/>
    <text x="575" y="100" font-size="15" font-weight="bold" fill="#3730a3">3 · Add MCP</text>
    <text x="575" y="122" font-size="12" fill="#475569">tool traffic :3000</text>
    <rect x="710" y="70" width="190" height="70" rx="10" fill="#eef2ff" stroke="#4f46e5"/>
    <text x="805" y="100" font-size="15" font-weight="bold" fill="#3730a3">4 · Analyze</text>
    <text x="805" y="122" font-size="12" fill="#475569">cost + usage</text>
    <path d="M210 105 H250" stroke="#94a3b8" stroke-width="2" marker-end="url(#a)"/>
    <path d="M440 105 H480" stroke="#94a3b8" stroke-width="2" marker-end="url(#a)"/>
    <path d="M670 105 H710" stroke="#94a3b8" stroke-width="2" marker-end="url(#a)"/>
    <text x="460" y="35" font-size="18" font-weight="bold" fill="#0f172a">Agentgateway in Docker — what you'll build</text>
  </g>
  <defs><marker id="a" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#94a3b8"/></marker></defs>
</svg>
```

- [ ] **Step 2: Write `diagram-01-gateway.svg`**

Create `00-agentgateway-tokenomic/assets/diagram-01-gateway.svg`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 280" font-family="Segoe UI, Arial, sans-serif">
  <rect width="880" height="280" fill="#ffffff"/>
  <g text-anchor="middle">
    <rect x="60" y="110" width="170" height="70" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="145" y="142" font-size="14" font-weight="bold" fill="#0f172a">Your browser</text>
    <text x="145" y="162" font-size="12" fill="#475569">Terminal + UI tab</text>
    <rect x="360" y="70" width="240" height="150" rx="12" fill="#eef2ff" stroke="#4f46e5" stroke-width="2"/>
    <text x="480" y="100" font-size="15" font-weight="bold" fill="#3730a3">agentgateway container</text>
    <text x="480" y="128" font-size="12" fill="#475569">Docker · cr.agentgateway.dev</text>
    <rect x="385" y="145" width="90" height="50" rx="8" fill="#ffffff" stroke="#4f46e5"/>
    <text x="430" y="167" font-size="12" font-weight="bold" fill="#3730a3">Admin/UI</text>
    <text x="430" y="184" font-size="12" fill="#475569">:15000</text>
    <rect x="490" y="145" width="90" height="50" rx="8" fill="#ffffff" stroke="#cbd5e1"/>
    <text x="535" y="167" font-size="12" font-weight="bold" fill="#64748b">LLM</text>
    <text x="535" y="184" font-size="12" fill="#94a3b8">:4000 (empty)</text>
    <path d="M230 145 H360" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <text x="295" y="135" font-size="12" fill="#3730a3">:15000/ui</text>
    <text x="440" y="40" font-size="17" font-weight="bold" fill="#0f172a">Lab 1 · Run the gateway, open the UI</text>
  </g>
  <defs><marker id="a" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#4f46e5"/></marker></defs>
</svg>
```

- [ ] **Step 3: Write `diagram-02-llm.svg`**

Create `00-agentgateway-tokenomic/assets/diagram-02-llm.svg`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 280" font-family="Segoe UI, Arial, sans-serif">
  <rect width="880" height="280" fill="#ffffff"/>
  <g text-anchor="middle">
    <rect x="30" y="110" width="150" height="70" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="105" y="142" font-size="14" font-weight="bold" fill="#0f172a">curl / client</text>
    <text x="105" y="162" font-size="12" fill="#475569">points at :4000</text>
    <rect x="340" y="95" width="220" height="100" rx="12" fill="#eef2ff" stroke="#4f46e5" stroke-width="2"/>
    <text x="450" y="130" font-size="15" font-weight="bold" fill="#3730a3">agentgateway</text>
    <text x="450" y="152" font-size="12" fill="#475569">LLM proxy :4000</text>
    <text x="450" y="172" font-size="12" fill="#475569">cost catalog on</text>
    <rect x="700" y="110" width="150" height="70" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="775" y="142" font-size="14" font-weight="bold" fill="#0f172a">OpenAI</text>
    <text x="775" y="162" font-size="12" fill="#475569">api.openai.com</text>
    <path d="M180 145 H340" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <text x="260" y="135" font-size="12" fill="#3730a3">openai/*</text>
    <path d="M560 145 H700" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <text x="630" y="135" font-size="12" fill="#3730a3">$OPENAI_API_KEY</text>
    <text x="440" y="45" font-size="17" font-weight="bold" fill="#0f172a">Lab 2 · OpenAI-compatible proxy</text>
  </g>
  <defs><marker id="a" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#4f46e5"/></marker></defs>
</svg>
```

- [ ] **Step 4: Write `diagram-03-mcp.svg`**

Create `00-agentgateway-tokenomic/assets/diagram-03-mcp.svg`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 320" font-family="Segoe UI, Arial, sans-serif">
  <rect width="880" height="320" fill="#ffffff"/>
  <g text-anchor="middle">
    <rect x="20" y="120" width="140" height="70" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="90" y="150" font-size="13" font-weight="bold" fill="#0f172a">client</text>
    <text x="90" y="170" font-size="11" fill="#475569">:4000 + :3000</text>
    <rect x="300" y="60" width="220" height="200" rx="12" fill="#eef2ff" stroke="#4f46e5" stroke-width="2"/>
    <text x="410" y="92" font-size="15" font-weight="bold" fill="#3730a3">agentgateway</text>
    <rect x="325" y="110" width="170" height="50" rx="8" fill="#ffffff" stroke="#4f46e5"/>
    <text x="410" y="140" font-size="12" fill="#3730a3">LLM :4000</text>
    <rect x="325" y="175" width="170" height="50" rx="8" fill="#ffffff" stroke="#4f46e5"/>
    <text x="410" y="205" font-size="12" fill="#3730a3">MCP :3000</text>
    <rect x="660" y="60" width="190" height="80" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="755" y="92" font-size="13" font-weight="bold" fill="#0f172a">OpenAI</text>
    <text x="755" y="115" font-size="11" fill="#475569">api.openai.com</text>
    <rect x="660" y="180" width="190" height="80" rx="10" fill="#ecfdf5" stroke="#10b981"/>
    <text x="755" y="210" font-size="13" font-weight="bold" fill="#065f46">mcp-everything</text>
    <text x="755" y="233" font-size="11" fill="#475569">separate container</text>
    <path d="M160 155 H300" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <path d="M520 135 H660" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <path d="M520 200 H660" stroke="#10b981" stroke-width="2" marker-end="url(#b)"/>
    <text x="590" y="190" font-size="11" fill="#065f46">:8080/mcp/</text>
    <text x="430" y="285" font-size="12" fill="#475569">shared Docker network "agw"</text>
    <text x="440" y="35" font-size="17" font-weight="bold" fill="#0f172a">Lab 3 · LLM + MCP, one gateway</text>
  </g>
  <defs>
    <marker id="a" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#4f46e5"/></marker>
    <marker id="b" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#10b981"/></marker>
  </defs>
</svg>
```

- [ ] **Step 5: Write `diagram-04-analyze.svg`**

Create `00-agentgateway-tokenomic/assets/diagram-04-analyze.svg`:
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 880 280" font-family="Segoe UI, Arial, sans-serif">
  <rect width="880" height="280" fill="#ffffff"/>
  <g text-anchor="middle">
    <rect x="40" y="105" width="200" height="80" rx="12" fill="#eef2ff" stroke="#4f46e5" stroke-width="2"/>
    <text x="140" y="138" font-size="14" font-weight="bold" fill="#3730a3">agentgateway</text>
    <text x="140" y="160" font-size="12" fill="#475569">every call logged</text>
    <rect x="350" y="105" width="180" height="80" rx="12" fill="#fff7ed" stroke="#f59e0b"/>
    <text x="440" y="138" font-size="14" font-weight="bold" fill="#92400e">request_logs</text>
    <text x="440" y="160" font-size="12" fill="#475569">SQLite · ~5k rows</text>
    <rect x="640" y="70" width="200" height="60" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="740" y="98" font-size="13" font-weight="bold" fill="#0f172a">Cost / tokenomics</text>
    <text x="740" y="117" font-size="11" fill="#475569">spend by model/user</text>
    <rect x="640" y="160" width="200" height="60" rx="10" fill="#f1f5f9" stroke="#94a3b8"/>
    <text x="740" y="188" font-size="13" font-weight="bold" fill="#0f172a">Usage / traffic</text>
    <text x="740" y="207" font-size="11" fill="#475569">top models, busiest users</text>
    <path d="M240 145 H350" stroke="#4f46e5" stroke-width="2" marker-end="url(#a)"/>
    <path d="M530 135 L640 105" stroke="#94a3b8" stroke-width="2" marker-end="url(#a)"/>
    <path d="M530 155 L640 185" stroke="#94a3b8" stroke-width="2" marker-end="url(#a)"/>
    <text x="440" y="40" font-size="17" font-weight="bold" fill="#0f172a">Lab 4 · Analyze a week of token spend</text>
  </g>
  <defs><marker id="a" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0 0 L6 3 L0 6 z" fill="#94a3b8"/></marker></defs>
</svg>
```

- [ ] **Step 6: Render all SVGs to PNG**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic/assets"
for f in diagram-journey diagram-01-gateway diagram-02-llm diagram-03-mcp diagram-04-analyze; do
  rsvg-convert -w 1320 "$f.svg" -o "$f.png" && echo "rendered $f.png"
done
ls -1 *.png
```
Expected: 5 PNG files listed.

- [ ] **Step 7: Commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/assets/*.svg 00-agentgateway-tokenomic/assets/*.png
git commit -m "Add progressive architecture diagrams (journey + one per lab)"
```

---

### Task 4: Lab 01 — run-gateway-docker

**Files:**
- Create: `00-agentgateway-tokenomic/01-run-gateway-docker/assignment.md`
- Create: `00-agentgateway-tokenomic/01-run-gateway-docker/setup-server`
- Create: `00-agentgateway-tokenomic/01-run-gateway-docker/check-server`
- Create: `00-agentgateway-tokenomic/01-run-gateway-docker/solve-server`

- [ ] **Step 1: Create `assignment.md`**

Create `00-agentgateway-tokenomic/01-run-gateway-docker/assignment.md`:
````markdown
---
slug: run-gateway-docker
id: GENERATED_ON_PUSH
type: challenge
title: Run the Gateway in Docker
teaser: Put one Docker container in front of all your AI traffic.
notes:
- type: text
  contents: |
    # 🐳 Run the Gateway in Docker

    Agentgateway is a single Rust binary — and a single Docker container. Start it
    with a minimal config and connect to its built-in web UI. No Kubernetes, no
    sidecars.
tabs:
- id: terminal
  title: Terminal
  type: terminal
  hostname: server
- id: editor
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Run the Gateway in Docker

![What you'll build across this lab](../assets/diagram-journey.png)

**What we're building:** one Agentgateway container that becomes the single control
point for every LLM and MCP call your agents make. We start it empty, then add an
LLM (Lab 2), an MCP server (Lab 3), and finish by analyzing the traffic (Lab 4).

![Lab 1 — run the gateway and open the UI](../assets/diagram-01-gateway.png)

A minimal **`/root/agentgateway/config.yaml`** is already staged (admin UI + a default
cost catalog + a request database, no models yet). Three ports matter: **:4000** (LLM
API), **:3000** (MCP), and **:15000** (admin + UI).

## Step 1 — Start the gateway container

The config dir is bind-mounted into the container at `/config`. We publish all three
ports and set `ADMIN_ADDR` so the UI is reachable from your browser.

```bash
docker run -d --name agentgateway --network agw \
  -v /root/agentgateway:/config \
  -p 4000:4000 -p 3000:3000 -p 15000:15000 \
  -e ADMIN_ADDR=0.0.0.0:15000 \
  -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml

sleep 5
docker ps --filter name=agentgateway
```

**What you'll see:** the container `agentgateway` with status `Up`.

## Step 2 — Connect to the admin API

```bash
curl -s -o /dev/null -w "admin HTTP %{http_code}\n" http://localhost:15000/config_dump
```

`200` means the gateway is alive and reporting its config.

## Step 3 — Open the UI

Open the **Agentgateway UI** tab (it points at `:15000/ui`). You'll see **Welcome to
Agentgateway**. There are no models yet — you'll add one next.

> Everything in this lab runs on the gateway VM, so the **Terminal** reaches the
> gateway directly at `localhost`. The **UI tab** reaches `:15000` for visual setup
> and observability.

> Next: add an OpenAI model and send your first call *through* the gateway. ➡️
````

- [ ] **Step 2: Create `setup-server`** (idempotent safety net — track setup already did the heavy lifting)

Create `00-agentgateway-tokenomic/01-run-gateway-docker/setup-server`:
```bash
#!/bin/bash
set -uo pipefail
# Track setup already provisioned /root/agentgateway and pre-pulled images.
# Be defensive in case a challenge is started in isolation.
docker network inspect agw >/dev/null 2>&1 || docker network create agw >/dev/null
[ -f /root/agentgateway/config.yaml ] || echo "WARNING: config.yaml missing (track setup-server should have created it)"
echo "challenge 1 setup complete"
```

- [ ] **Step 3: Create `check-server`**

Create `00-agentgateway-tokenomic/01-run-gateway-docker/check-server`:
```bash
#!/bin/bash
set -uo pipefail
if ! docker ps --filter name=agentgateway --filter status=running --format '{{.Names}}' | grep -q '^agentgateway$'; then
  fail-message "The 'agentgateway' container isn't running. Start it with the docker run command in Step 1."
  exit 1
fi
code=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:15000/config_dump)
if [ "$code" != "200" ]; then
  fail-message "The admin API on :15000 returned HTTP $code. Did you pass -e ADMIN_ADDR=0.0.0.0:15000 and publish -p 15000:15000?"
  exit 1
fi
echo "Gateway container is running and the admin API on :15000 is responding."
```

- [ ] **Step 4: Create `solve-server`** (what `instruqt track test` runs to auto-complete the step)

Create `00-agentgateway-tokenomic/01-run-gateway-docker/solve-server`:
```bash
#!/bin/bash
set -uo pipefail
docker rm -f agentgateway >/dev/null 2>&1 || true
docker run -d --name agentgateway --network agw \
  -v /root/agentgateway:/config \
  -p 4000:4000 -p 3000:3000 -p 15000:15000 \
  -e ADMIN_ADDR=0.0.0.0:15000 \
  -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml
sleep 6
echo "solve: gateway started"
```

- [ ] **Step 5: Make scripts executable + verify locally**

Run (reproduces the validated Stage A against the real config dir if present locally; on the VM `instruqt track test` will run it):
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
chmod +x 01-run-gateway-docker/setup-server 01-run-gateway-docker/check-server 01-run-gateway-docker/solve-server
bash -n 01-run-gateway-docker/*server && echo "lab1 scripts: syntax OK"
```
Expected: `lab1 scripts: syntax OK`.

- [ ] **Step 6: Commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/01-run-gateway-docker/
git commit -m "Lab 1: run-gateway-docker (assignment + lifecycle scripts)"
```

---

### Task 5: Lab 02 — add-an-llm

**Files:**
- Create: `00-agentgateway-tokenomic/02-add-an-llm/assignment.md`
- Create: `00-agentgateway-tokenomic/02-add-an-llm/setup-server`
- Create: `00-agentgateway-tokenomic/02-add-an-llm/check-server`
- Create: `00-agentgateway-tokenomic/02-add-an-llm/solve-server`

- [ ] **Step 1: Create `assignment.md`**

Create `00-agentgateway-tokenomic/02-add-an-llm/assignment.md`:
````markdown
---
slug: add-an-llm
id: GENERATED_ON_PUSH
type: challenge
title: Add an LLM
teaser: Turn the gateway into an OpenAI-compatible proxy.
notes:
- type: text
  contents: |
    # 🌐 Add an LLM

    Point the gateway at OpenAI and it becomes an OpenAI-compatible proxy your apps
    call instead of api.openai.com — with cost tracking on by default.
tabs:
- id: terminal
  title: Terminal
  type: terminal
  hostname: server
- id: editor
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Add an LLM

![Lab 2 — OpenAI-compatible proxy](../assets/diagram-02-llm.png)

**What we're building:** an `llm.models` entry so clients call `openai/...` through
`:4000` and the gateway forwards to OpenAI using a key they never see.

## Step 1 — Add an OpenAI model

**Option A — the UI:** open the **Agentgateway UI** → **Models** → **Add model**.
Incoming match `openai/*`, provider **OpenAI**, API key **Env var** `OPENAI_API_KEY`,
**Save**.

**Option B — YAML (Editor):** open `/root/agentgateway/config.yaml` and replace the
empty `models: []` with:

```yaml
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
```

Then restart the gateway so it re-reads the file:

```bash
agw-restart
```

`$OPENAI_API_KEY` is read from the container's environment — clients never see the
real key.

## Step 2 — Send your first call *through* the gateway

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' \
  | jq -r '.choices[0].message.content'
```

**What you'll see:** a normal OpenAI response — but it went through **your** gateway.
The bundled cost catalog already priced this call and logged it to the request
database (you'll mine that in Lab 4).

> Next: bring **tool traffic** under the same gateway with a separate MCP server. ➡️
````

- [ ] **Step 2: Create `setup-server`** (ensure gateway is running so the lab is enterable even if started in isolation)

Create `00-agentgateway-tokenomic/02-add-an-llm/setup-server`:
```bash
#!/bin/bash
set -uo pipefail
docker network inspect agw >/dev/null 2>&1 || docker network create agw >/dev/null
if ! docker ps --filter name=agentgateway --filter status=running --format '{{.Names}}' | grep -q '^agentgateway$'; then
  docker rm -f agentgateway >/dev/null 2>&1 || true
  docker run -d --name agentgateway --network agw -v /root/agentgateway:/config \
    -p 4000:4000 -p 3000:3000 -p 15000:15000 -e ADMIN_ADDR=0.0.0.0:15000 -e OPENAI_API_KEY \
    cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml >/dev/null
  sleep 5
fi
echo "challenge 2 setup complete"
```

- [ ] **Step 3: Create `check-server`** (validated Stage B logic)

Create `00-agentgateway-tokenomic/02-add-an-llm/check-server`:
```bash
#!/bin/bash
set -uo pipefail
if ! docker ps --filter name=agentgateway --filter status=running --format '{{.Names}}' | grep -q '^agentgateway$'; then
  fail-message "The 'agentgateway' container isn't running. Start it (Lab 1) then add the model."
  exit 1
fi
if ! grep -q 'openai/\*' /root/agentgateway/config.yaml 2>/dev/null && ! grep -q 'provider: openAI' /root/agentgateway/config.yaml 2>/dev/null; then
  fail-message "No OpenAI model found in /root/agentgateway/config.yaml. Add the llm.models entry (or use the UI) and run agw-restart."
  exit 1
fi
code=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:4000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"ping"}],"max_tokens":5}')
if [ "$code" != "200" ]; then
  fail-message "A test call to :4000 returned HTTP $code. Ensure the model has params.apiKey set to \$OPENAI_API_KEY and run agw-restart."
  exit 1
fi
echo "Gateway is proxying OpenAI on :4000 (test call returned 200)."
```

- [ ] **Step 4: Create `solve-server`**

Create `00-agentgateway-tokenomic/02-add-an-llm/solve-server`:
```bash
#!/bin/bash
set -uo pipefail
cat > /root/agentgateway/config.yaml <<'YAML'
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:////config/data/data.db"
  modelCatalog:
    - file: /config/costs/base-costs.json
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
frontendPolicies:
  http:
    maxBufferSize: 33554432
YAML
docker restart agentgateway >/dev/null 2>&1; sleep 4
echo "solve: OpenAI model added"
```

- [ ] **Step 5: chmod + syntax check + commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
chmod +x 02-add-an-llm/setup-server 02-add-an-llm/check-server 02-add-an-llm/solve-server
bash -n 02-add-an-llm/*server && echo "lab2 scripts: syntax OK"
cd ..
git add 00-agentgateway-tokenomic/02-add-an-llm/
git commit -m "Lab 2: add-an-llm (OpenAI proxy + validated check)"
```

---

### Task 6: Lab 03 — add-everything-mcp

**Files:**
- Create: `00-agentgateway-tokenomic/03-add-everything-mcp/assignment.md`
- Create: `00-agentgateway-tokenomic/03-add-everything-mcp/setup-server`
- Create: `00-agentgateway-tokenomic/03-add-everything-mcp/check-server`
- Create: `00-agentgateway-tokenomic/03-add-everything-mcp/solve-server`

- [ ] **Step 1: Create `assignment.md`**

Create `00-agentgateway-tokenomic/03-add-everything-mcp/assignment.md`:
````markdown
---
slug: add-everything-mcp
id: GENERATED_ON_PUSH
type: challenge
title: Add an MCP Server
teaser: Bring agent tool traffic under the same gateway.
notes:
- type: text
  contents: |
    # 🔌 Add an MCP Server

    Agents call tools over MCP. Run a separate MCP container and put it behind the
    same gateway so LLM and tool traffic share one control point.
tabs:
- id: terminal
  title: Terminal
  type: terminal
  hostname: server
- id: editor
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Add an MCP Server

![Lab 3 — LLM + MCP behind one gateway](../assets/diagram-03-mcp.png)

**What we're building:** a **separate** `mcp-everything` container on the shared
`agw` Docker network, with the gateway proxying it on `:3000`. The gateway reaches
it by container name over HTTP.

## Step 1 — Run the MCP server as its own container

```bash
docker run -d --name mcp-everything --network agw \
  -e PORT=8080 -p 8080:8080 \
  node:20-alpine \
  npx -y @modelcontextprotocol/server-everything streamableHttp

sleep 5
docker logs mcp-everything 2>&1 | tail -3
```

**What you'll see:** `MCP Streamable HTTP Server listening on port 8080`.

## Step 2 — Point the gateway at it

In the **Editor**, add a top-level `mcp:` block to `/root/agentgateway/config.yaml`
(a sibling of `llm:`). The host uses the container name on the `agw` network:

```yaml
mcp:
  port: 3000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      exposeHeaders: ["Mcp-Session-Id"]
  targets:
  - name: everything
    mcp:
      host: http://mcp-everything:8080/mcp/
```

```bash
agw-restart
```

## Step 3 — List the tools (and see the token cost)

```bash
SID=$(curl -s -D - -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')

curl -s -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -s -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | grep '^data:' | sed 's/^data: //' \
  | jq '{tool_count: (.result.tools|length), tool_names: [.result.tools[].name]}'
```

**What you'll see:** a dozen-plus tools. Every tool name, description, and JSON schema
is sent to the model as **input tokens** on each turn — that's the hidden cost of
tool-heavy agents, now visible and governable at the gateway.

> Next: a week of traffic has already flowed through this gateway. Let's analyze the
> spend. ➡️
````

- [ ] **Step 2: Create `setup-server`** (ensure gateway running + pre-warm MCP image)

Create `00-agentgateway-tokenomic/03-add-everything-mcp/setup-server`:
```bash
#!/bin/bash
set -uo pipefail
docker network inspect agw >/dev/null 2>&1 || docker network create agw >/dev/null
docker pull node:20-alpine >/dev/null 2>&1 || true
if ! docker ps --filter name=agentgateway --filter status=running --format '{{.Names}}' | grep -q '^agentgateway$'; then
  docker rm -f agentgateway >/dev/null 2>&1 || true
  docker run -d --name agentgateway --network agw -v /root/agentgateway:/config \
    -p 4000:4000 -p 3000:3000 -p 15000:15000 -e ADMIN_ADDR=0.0.0.0:15000 -e OPENAI_API_KEY \
    cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml >/dev/null
  sleep 5
fi
echo "challenge 3 setup complete"
```

- [ ] **Step 3: Create `check-server`** (validated Stage C logic)

Create `00-agentgateway-tokenomic/03-add-everything-mcp/check-server`:
```bash
#!/bin/bash
set -uo pipefail
if ! docker ps --filter name=mcp-everything --filter status=running --format '{{.Names}}' | grep -q '^mcp-everything$'; then
  fail-message "The 'mcp-everything' container isn't running. Start it with the docker run command in Step 1."
  exit 1
fi
if ! grep -q 'mcp-everything:8080/mcp' /root/agentgateway/config.yaml 2>/dev/null; then
  fail-message "No MCP target in /root/agentgateway/config.yaml. Add the top-level mcp: block (port 3000, host http://mcp-everything:8080/mcp/), then agw-restart."
  exit 1
fi
code=$(curl -s -o /dev/null -w '%{http_code}' -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"c","version":"1"}}}')
if [ "$code" != "200" ]; then
  fail-message "MCP endpoint on :3000 returned HTTP $code (expected 200). Check the mcp: block port and run agw-restart."
  exit 1
fi
SID=$(curl -s -D - -o /dev/null -X POST http://localhost:3000 \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')
curl -s -o /dev/null -X POST http://localhost:3000 -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'
tools=$(curl -s -X POST http://localhost:3000 -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -H "mcp-session-id: $SID" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' | grep '^data:' | sed 's/^data: //' | jq -r '.result.tools | length' 2>/dev/null)
if [ -z "${tools:-}" ] || ! [ "$tools" -ge 1 ] 2>/dev/null; then
  fail-message "MCP tools/list returned no tools. Re-run the Step 3 handshake against :3000."
  exit 1
fi
echo "MCP is live on :3000 ($tools tools), LLM on :4000 — both under one gateway."
```

- [ ] **Step 4: Create `solve-server`**

Create `00-agentgateway-tokenomic/03-add-everything-mcp/solve-server`:
```bash
#!/bin/bash
set -uo pipefail
docker rm -f mcp-everything >/dev/null 2>&1 || true
docker run -d --name mcp-everything --network agw -e PORT=8080 -p 8080:8080 \
  node:20-alpine npx -y @modelcontextprotocol/server-everything streamableHttp >/dev/null
sleep 25
cat > /root/agentgateway/config.yaml <<'YAML'
config:
  adminAddr: "0.0.0.0:15000"
  database:
    url: "sqlite:////config/data/data.db"
  modelCatalog:
    - file: /config/costs/base-costs.json
llm:
  port: 4000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      allowMethods: ["GET","POST","OPTIONS"]
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
mcp:
  port: 3000
  policies:
    cors:
      allowOrigins: ["*"]
      allowHeaders: ["*"]
      exposeHeaders: ["Mcp-Session-Id"]
  targets:
  - name: everything
    mcp:
      host: http://mcp-everything:8080/mcp/
frontendPolicies:
  http:
    maxBufferSize: 33554432
YAML
docker restart agentgateway >/dev/null 2>&1; sleep 4
echo "solve: MCP server added"
```

- [ ] **Step 5: chmod + syntax check + commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
chmod +x 03-add-everything-mcp/setup-server 03-add-everything-mcp/check-server 03-add-everything-mcp/solve-server
bash -n 03-add-everything-mcp/*server && echo "lab3 scripts: syntax OK"
cd ..
git add 00-agentgateway-tokenomic/03-add-everything-mcp/
git commit -m "Lab 3: add-everything-mcp (separate container over HTTP + validated check)"
```

---

### Task 7: Lab 04 — analyze-the-traffic

**Files:**
- Create: `00-agentgateway-tokenomic/04-analyze-the-traffic/assignment.md`
- Create: `00-agentgateway-tokenomic/04-analyze-the-traffic/setup-server`
- Create: `00-agentgateway-tokenomic/04-analyze-the-traffic/check-server`
- Create: `00-agentgateway-tokenomic/04-analyze-the-traffic/solve-server`

- [ ] **Step 1: Create `assignment.md`** (queries are the validated Stage D queries)

Create `00-agentgateway-tokenomic/04-analyze-the-traffic/assignment.md`:
````markdown
---
slug: analyze-the-traffic
id: GENERATED_ON_PUSH
type: challenge
title: Analyze the Traffic
teaser: Turn captured traffic into cost and usage answers.
notes:
- type: text
  contents: |
    # 📊 Analyze the Traffic

    A week of traffic has already flowed through your gateway. Every call was logged
    with tokens and cost. Time to answer the questions a CFO would ask.
tabs:
- id: terminal
  title: Terminal
  type: terminal
  hostname: server
- id: ui
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Analyze the Traffic

![Lab 4 — analyze a week of token spend](../assets/diagram-04-analyze.png)

**What we're analyzing:** the gateway logged every call to a SQLite request database
(`/root/agentgateway/data/data.db`) — about a week of traffic across multiple models,
providers, and users. The query the gateway answers from is the same table you'll
query directly.

The database (`request_logs`) has one row per request with `cost`, `total_tokens`,
`gen_ai_provider_name`, `gen_ai_request_model`, `agentgateway_user`, and
`agentgateway_group`.

## Cost / tokenomics

```bash
DB=/root/agentgateway/data/data.db

# Total spend, tokens, requests
sqlite3 -header -column "$DB" \
 "SELECT printf('\$%.2f', SUM(cost)) total_spend, SUM(total_tokens) tokens, COUNT(*) requests FROM request_logs;"

# Spend by provider
sqlite3 -header -column "$DB" \
 "SELECT gen_ai_provider_name provider, printf('\$%.2f', SUM(cost)) spend, COUNT(*) calls
  FROM request_logs GROUP BY 1 ORDER BY SUM(cost) DESC;"

# Top-spend models
sqlite3 -header -column "$DB" \
 "SELECT gen_ai_request_model model, printf('\$%.2f', SUM(cost)) spend
  FROM request_logs GROUP BY 1 ORDER BY SUM(cost) DESC LIMIT 5;"
```

**What you'll see** (your numbers will be similar): a few hundred dollars of total
spend, OpenAI as the largest provider, and `gpt-4.1` / `claude-sonnet-4-5` near the
top of per-model spend.

## Usage / traffic

```bash
DB=/root/agentgateway/data/data.db

# Busiest users (and what they cost)
sqlite3 -header -column "$DB" \
 "SELECT agentgateway_user user, COUNT(*) calls, printf('\$%.2f', SUM(cost)) spend
  FROM request_logs GROUP BY 1 ORDER BY calls DESC LIMIT 5;"

# Spend by team/group
sqlite3 -header -column "$DB" \
 "SELECT agentgateway_group team, printf('\$%.2f', SUM(cost)) spend
  FROM request_logs GROUP BY 1 ORDER BY SUM(cost) DESC;"
```

**What you'll see:** a single user (e.g. `alice@example.com`) driving a large share of
spend, and one team dominating the bill — exactly the attribution a direct-to-OpenAI
setup can't give you.

## See it in the UI

Open the **Agentgateway UI** tab and browse the request/usage pages for the same data
visually — the gateway turns invisible AI spend into queryable, attributable data.

> 🎉 You ran Agentgateway in Docker, proxied LLM + MCP traffic through one control
> point, and turned a week of token spend into answers. That's the foundation for
> governance: budgets, rate limits, and cost-aware routing.
````

- [ ] **Step 2: Create `setup-server`** (re-seed if the DB is missing/empty so the lab always has data)

Create `00-agentgateway-tokenomic/04-analyze-the-traffic/setup-server`:
```bash
#!/bin/bash
set -uo pipefail
export PATH="$HOME/.local/bin:/usr/local/bin:$PATH"
DB=/root/agentgateway/data/data.db
rows=$(sqlite3 "$DB" 'SELECT COUNT(*) FROM request_logs;' 2>/dev/null || echo 0)
if [ "${rows:-0}" -lt 100 ]; then
  echo "re-seeding mock traffic (rows=$rows)"
  # Gateway holds the DB open; seeding into a fresh file then swapping avoids a write lock.
  uv run --python 3.12 /root/agentgateway/gen-mock-logs.py --output /tmp/seed.db --replace --requests 5000 --days 7 --seed 7 || true
  docker stop agentgateway >/dev/null 2>&1 || true
  cp /tmp/seed.db "$DB" && chmod 777 "$DB"
  docker start agentgateway >/dev/null 2>&1 || true
fi
echo "challenge 4 setup complete (rows=$(sqlite3 "$DB" 'SELECT COUNT(*) FROM request_logs;' 2>/dev/null || echo 0))"
```

- [ ] **Step 3: Create `check-server`** (lenient — DB exists and has rows)

Create `00-agentgateway-tokenomic/04-analyze-the-traffic/check-server`:
```bash
#!/bin/bash
set -uo pipefail
DB=/root/agentgateway/data/data.db
if [ ! -f "$DB" ]; then
  fail-message "Request database not found at $DB. The track setup seeds it; try restarting the challenge."
  exit 1
fi
rows=$(sqlite3 "$DB" 'SELECT COUNT(*) FROM request_logs;' 2>/dev/null || echo 0)
if [ -z "${rows:-}" ] || ! [ "$rows" -ge 100 ] 2>/dev/null; then
  fail-message "The request log has $rows rows (expected >=100). The setup should seed ~5000; restart the challenge."
  exit 1
fi
echo "Request log has $rows rows — run the cost and usage queries to analyze the spend."
```

- [ ] **Step 4: Create `solve-server`** (runs the analysis queries so `instruqt track test` exercises them)

Create `00-agentgateway-tokenomic/04-analyze-the-traffic/solve-server`:
```bash
#!/bin/bash
set -uo pipefail
DB=/root/agentgateway/data/data.db
sqlite3 -header -column "$DB" "SELECT printf('\$%.2f', SUM(cost)) total_spend, SUM(total_tokens) tokens, COUNT(*) requests FROM request_logs;"
sqlite3 -header -column "$DB" "SELECT gen_ai_provider_name provider, printf('\$%.2f', SUM(cost)) spend FROM request_logs GROUP BY 1 ORDER BY SUM(cost) DESC;"
sqlite3 -header -column "$DB" "SELECT agentgateway_user user, COUNT(*) calls, printf('\$%.2f', SUM(cost)) spend FROM request_logs GROUP BY 1 ORDER BY calls DESC LIMIT 5;"
echo "solve: analysis queries ran"
```

- [ ] **Step 5: chmod + syntax check + commit**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
chmod +x 04-analyze-the-traffic/setup-server 04-analyze-the-traffic/check-server 04-analyze-the-traffic/solve-server
bash -n 04-analyze-the-traffic/*server && echo "lab4 scripts: syntax OK"
cd ..
git add 00-agentgateway-tokenomic/04-analyze-the-traffic/
git commit -m "Lab 4: analyze-the-traffic (cost + usage queries, lenient check)"
```

---

### Task 8: README + full Instruqt validation/test

**Files:**
- Create: `00-agentgateway-tokenomic/README.md`

- [ ] **Step 1: Create `README.md`**

Create `00-agentgateway-tokenomic/README.md`:
```markdown
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
```

- [ ] **Step 2: Validate track structure**

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
instruqt track validate
```
Expected: validation passes (no errors). Fix any reported issues (missing fields, bad slugs) before continuing.

- [ ] **Step 3: Commit README**

```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/README.md
git commit -m "Add track README + author/test notes"
```

- [ ] **Step 4: Push and run the full automated test (user-driven, requires Instruqt cloud)**

> This step provisions a real VM and runs every lab's setup → solve → check. It costs cloud minutes — run when ready to validate end-to-end.

Run:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos/00-agentgateway-tokenomic"
instruqt track push
instruqt track test agentgateway-tokenomic --skip-fail-check
```
Expected: each challenge reports solve + check passing. The `id`/`checksum` fields get written into `track.yml` and `assignment.md` files by the push — commit those afterward:
```bash
cd "/Users/sebbycorp/Library/CloudStorage/GoogleDrive-sebastian.maniak@solo.io/My Drive/Projects/Instruqt-demos"
git add 00-agentgateway-tokenomic/
git commit -m "Sync Instruqt id/checksum after push"
```

---

## Self-Review

**1. Spec coverage:**
- Sandbox/config.yml + OPENAI secret → Task 1 ✅
- Docker topology (network, gateway run, MCP container, HTTP target) → Tasks 2, 4, 5, 6 ✅
- 4 labs with assignment/setup/check/solve → Tasks 4–7 ✅
- 5 diagrams (journey + per-lab) → Task 3 ✅
- Reused assets (base-costs.json, gen-mock-logs.py) → Task 1 ✅
- DB pre-seed before gateway start + uv requirement → Task 2 ✅
- Lab 04 lenient grading + cost & usage queries → Task 7 ✅
- README + validation/test workflow → Task 8 ✅
- Out-of-scope items (auth/budgets/routing) → intentionally excluded; flagged in README "foundation for governance" ✅

**2. Placeholder scan:** `GENERATED_ON_PUSH` in each `assignment.md` `id:` is intentional — Instruqt assigns real ids on push (Task 8 Step 4 commits them). The `__BASE_COSTS_JSON__` / `__GEN_MOCK_LOGS_PY__` tokens have an explicit substitution command + a grep gate (Task 2). No other placeholders.

**3. Consistency:** container names (`agentgateway`, `mcp-everything`), network (`agw`), ports (4000/3000/15000/8080), DB path (`/root/agentgateway/data/data.db` host = `/config/data/data.db` container), image tag (`v1.3.1`), and `agw-restart` are identical across every task and match the validated smoke test.
```
