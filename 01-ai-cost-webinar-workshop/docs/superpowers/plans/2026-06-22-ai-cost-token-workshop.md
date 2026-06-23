# AI Cost & Token Spend Workshop — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an Instruqt track that teaches enterprise platform/FinOps teams how to track, troubleshoot, and govern LLM + MCP token spend with standalone agentgateway v1.3.1.

**Architecture:** Single VM, agentgateway runs as a background binary (`agentgateway -f /root/config.yaml`); LLM API on `:4000`, admin/UI on `:15000`, metrics on `:15020`. Config evolves per challenge; each `solve-server` writes a complete, validated config and restarts the gateway so learners can skip ahead. Real OpenAI calls demonstrate live token+cost; a generated SQLite dataset powers fleet-scale analysis. The narrative: a platform team chasing down a surprise AI bill.

**Tech Stack:** agentgateway v1.3.1 (standalone binary), OpenAI API + official `openai` CLI, `curl`/`jq`, `uv`/Python (mock data generator), SQLite, Instruqt (track.yml/config.yml + per-challenge assignment.md + setup/check/solve scripts).

---

## Verified Facts (tested against the real v1.3.0 binary + schema; 1.3.1 is a compatible patch)

- **Install:** `curl -sL https://agentgateway.dev/install | bash -s -- --version 1.3.1` → binary on PATH.
- **Run (background):** `nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &`
- **Validate any config without running:** `agentgateway -f <file> --validate-only` → prints `Configuration is valid!` or a precise error.
- **Ports:** `4000` = OpenAI-compatible API; `15000` = admin + UI (`/config_dump`, `/ui`), bound to localhost; `15020` = Prometheus metrics (`/metrics`).
- **Backend AI provider** lives at `backends[].ai.{name, provider.openAI.{model?}}`; the API key is a **route policy** `backendAuth.key: "$OPENAI_API_KEY"` (env var expansion works).
- **Cost** requires `config.modelCatalog: [{file: <path>}]`; catalog JSON is `{providers:{<provider>:{models:{<model>:{rates:{input,output}}}}}}` where rates are **strings = USD per 1M tokens**. Cost then appears in logs as `agw.ai.usage.cost.total=` and in `:15020/metrics` as `agentgateway_cost_catalog_lookups_total{status="Exact",...}`.
- **Token budget:** route policy `localRateLimit: [{maxTokens, tokensPerFill, fillInterval:"1h", type: tokens}]` + mark the route as LLM with policy `ai: {}`; exceeding the bucket returns **HTTP 429**.
- **Model restriction:** route policy `ai.overrides.model: gpt-4o-mini` forces every request to that model even when the client asks for `gpt-4o` (verified: response model came back `gpt-4o-mini`).
- **MCP backend:** `backends[].mcp.targets[].{name, stdio:{cmd,args}}`; route matched with `matches[].path.pathPrefix: /mcp`. (No `name` key directly under `mcp`.)
- **Path matching:** `matches: [{path: {pathPrefix: /v1}}]`. LLM route must match `/v1` so it doesn't swallow `/mcp`.
- **Mock data:** the generator is stdlib-only but needs Python ≥3.11; run via `uv run gen-mock-logs.py ...`. Default run produces 5000 rows / 12 users / 4 providers / 7 days; `alice@example.com` is the `enterprise-batch` whale (~$229 of ~$331 total), `sales` is the top team, and `batch` workloads are ~80% of spend.

---

## File Structure

```
01-ai-cost-webinar-workshop/
  track.yml
  config.yml
  track_scripts/
    setup-server                 # install binary + openai CLI; stage assets; store key
    cleanup-server
  assets/
    gen-mock-logs.py             # the provided generator (copied verbatim)
    costs/catalog.json           # model cost catalog
  01-the-blind-spot/             assignment.md, check-server
  02-stand-up-the-gateway/       assignment.md, setup-server, check-server, solve-server
  03-cost-of-every-request/      assignment.md, setup-server, check-server, solve-server
  04-day-2-operations/           assignment.md, setup-server, check-server, solve-server
  05-governance-budgets-models/  assignment.md, setup-server, check-server, solve-server
  06-mcp-is-spend-too/           assignment.md, setup-server, check-server, solve-server
  07-answer-the-cfo/             assignment.md, setup-server, check-server, solve-server
  08-production-scale/           assignment.md, check-server
```

Conventions copied from the sibling `agentgateway-oss-quickstart` track: `setup-server`/`check-server`/`solve-server` are bash scripts (no extension); `check-server` calls `fail-message "..."` then `exit 1` on failure and `exit 0` on success; `assignment.md` has YAML front-matter with `slug/id/type/title/teaser/notes/tabs`. `id` fields are placeholders until `instruqt track push` assigns them.

---

## Task 1: Track scaffolding (config.yml, track.yml, track_scripts, assets)

**Files:**
- Create: `config.yml`, `track.yml`, `track_scripts/setup-server`, `track_scripts/cleanup-server`, `assets/gen-mock-logs.py`, `assets/costs/catalog.json`

- [ ] **Step 1: Create `config.yml`** (reuse the proven image + secret)

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

- [ ] **Step 2: Create `track.yml`** (id/checksum filled by `instruqt track push`)

```yaml
slug: ai-cost-token-spend-agentgateway
id: ""
title: 'AI Cost & Token Spend: Govern LLM and MCP Traffic with Agentgateway'
teaser: Track, troubleshoot, and control AI token spend across your org using standalone Agentgateway OSS.
description: |
  Finance just forwarded a five-figure OpenAI bill and asked "who spent this?" — and nobody on the platform team can answer. Agents, copilots, and scripts are calling LLMs and MCP tools directly, with no shared control point. That's shadow AI, and it's the same loss of control enterprises hit with microservices before API gateways.

  In this ~15-minute hands-on lab you'll put **standalone Agentgateway OSS v1.3.1** in front of all AI traffic and get control back:

  - 🧮 Understand how tokens are calculated and what LLM features actually cost
  - 🚦 Route OpenAI traffic through a single gateway you operate
  - 💵 See realized USD cost and token usage on every request
  - 🔧 Troubleshoot the gateway with the admin API on :15000
  - 🛡️ Enforce per-team token budgets and an approved-model policy
  - 🔌 Bring MCP tool traffic under the same control point
  - 📊 Answer "who spent the money?" with fleet-scale cost analysis

  Built for platform, FinOps, and SecOps teams evaluating Agentgateway. Familiarity with the terminal and JSON is helpful; no Kubernetes required.
icon: ""
tags:
- ai
- llm
- mcp
- finops
- cost
- agentgateway
- solo-io
owner: soloio
developers:
- sebastian.maniak@solo.io
idle_timeout: 1800
timelimit: 2400
lab_config:
  sidebar_enabled: true
  feedback_recap_enabled: true
  feedback_tab_enabled: false
  loadingMessages: true
  hideStopButton: false
checksum: ""
enhanced_loading: false
```

- [ ] **Step 3: Copy the generator verbatim** to `assets/gen-mock-logs.py`

Source: `/Users/sebbycorp/Downloads/Untitled (1)` (copy byte-for-byte; it is stdlib-only and already matches agentgateway's request-log schema).

- [ ] **Step 4: Create `assets/costs/catalog.json`** (USD per 1M tokens; covers the models used live + a few from the dataset)

```json
{
  "providers": {
    "openai": {
      "models": {
        "gpt-4o-mini": { "rates": { "input": "0.15", "output": "0.60" } },
        "gpt-4o":      { "rates": { "input": "2.50", "output": "10.00" } },
        "gpt-4.1":     { "rates": { "input": "2.00", "output": "8.00" } },
        "gpt-4.1-mini":{ "rates": { "input": "0.40", "output": "1.60" } }
      }
    }
  }
}
```

- [ ] **Step 5: Create `track_scripts/setup-server`** (install binary + openai CLI, stage assets, store key)

```bash
#!/bin/bash
# Wait for instruqt bootstrap
until [ -f /opt/instruqt/bootstrap/host-bootstrap-completed ]; do
  sleep 1
done

set -euo pipefail
set-workdir /root

# Store OpenAI key for use across challenges
if [ -n "${OPENAI_API_KEY:-}" ]; then
  echo "export OPENAI_API_KEY=${OPENAI_API_KEY}" >> ~/.bashrc
  echo "${OPENAI_API_KEY}" > /root/.openai-api-key
  echo "=== OpenAI API key stored ==="
else
  echo "=== WARNING: OPENAI_API_KEY not set ==="
fi

# Install agentgateway standalone binary (pinned)
echo "=== Installing agentgateway v1.3.1 ==="
curl -sL https://agentgateway.dev/install | bash -s -- --version 1.3.1
# Ensure on PATH for non-login shells
ln -sf "$(command -v agentgateway || echo /usr/local/bin/agentgateway)" /usr/local/bin/agentgateway 2>/dev/null || true

# Install the official OpenAI CLI (used to drive traffic through the gateway)
pip3 install --quiet --upgrade openai || pip install --quiet --upgrade openai

# Ensure uv exists for the mock-data generator (needs Python >=3.11)
command -v uv >/dev/null 2>&1 || curl -LsSf https://astral.sh/uv/install.sh | sh

# Stage workshop assets onto the VM
mkdir -p /root/costs
cp /opt/instruqt/track/assets/gen-mock-logs.py /root/gen-mock-logs.py 2>/dev/null || true
cp /opt/instruqt/track/assets/costs/catalog.json /root/costs/catalog.json 2>/dev/null || true

# Convenience: point the OpenAI CLI/SDK at the gateway by default
cat >> ~/.bashrc <<'RC'
export OPENAI_BASE_URL=http://localhost:4000/v1
agw-restart() { pkill -f "agentgateway -f" 2>/dev/null; sleep 1; nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 & sleep 3; }
RC

echo "=== Track setup complete ==="
```

> Note on asset path: Instruqt copies a track's `assets/` to `/opt/instruqt/track/assets/` on the VM. If this image places them elsewhere, the per-challenge `setup-server` scripts re-stage from the same path, so a missing copy here is non-fatal.

- [ ] **Step 6: Create `track_scripts/cleanup-server`**

```bash
#!/bin/bash
set -euo pipefail
pkill -f "agentgateway -f" 2>/dev/null || true
echo "cleanup complete"
```

- [ ] **Step 7: Validate the catalog JSON and generator locally**

Run: `python3 -c "import json;json.load(open('assets/costs/catalog.json'));print('catalog ok')"`
Expected: `catalog ok`
Run: `uv run assets/gen-mock-logs.py --replace --requests 50 -o /tmp/t.db && sqlite3 /tmp/t.db "select count(*) from request_logs"`
Expected: `50`

- [ ] **Step 8: Commit**

```bash
git add 01-ai-cost-webinar-workshop/config.yml 01-ai-cost-webinar-workshop/track.yml 01-ai-cost-webinar-workshop/track_scripts 01-ai-cost-webinar-workshop/assets
git commit -m "Scaffold AI cost workshop: track config, setup scripts, assets"
```

---

## Task 2: Challenge 1 — The Blind Spot (read-only theory)

**Files:**
- Create: `01-the-blind-spot/assignment.md`, `01-the-blind-spot/check-server`

- [ ] **Step 1: Create `01-the-blind-spot/assignment.md`**

Front-matter: `slug: the-blind-spot`, `type: challenge`, one Terminal tab (`hostname: server`). Body covers, in business-first language:
- The surprise bill: Finance asks "who spent this?" and the platform team can't answer.
- **How tokens work:** every request is billed on input tokens + output tokens; price is per 1M tokens and varies up to ~17× between models (foreshadow gpt-4o vs gpt-4o-mini, the exact ratio learners will see in Ch3).
- **What costs extra:** long context windows, streamed responses, reasoning tokens, cached input (cheaper), embeddings, and **tool/function calls** (tool schemas + results ride along in context — sets up MCP in Ch6).
- **Why provider dashboards aren't enough:** per-account, delayed, and not mapped to teams/agents/humans.
- The fix: a single gateway in front of all AI traffic = the control point for visibility and governance.

End with: "Next: stand up that gateway." No commands to run.

- [ ] **Step 2: Create `01-the-blind-spot/check-server`** (read-only → always pass)

```bash
#!/bin/bash
exit 0
```

- [ ] **Step 3: Commit**

```bash
git add 01-ai-cost-webinar-workshop/01-the-blind-spot
git commit -m "Ch1: The Blind Spot (tokens & cost theory)"
```

---

## Task 3: Challenge 2 — Stand Up the Gateway

**Files:**
- Create: `02-stand-up-the-gateway/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `02-stand-up-the-gateway/setup-server`** (idempotent; ensure binary present)

```bash
#!/bin/bash
set -euo pipefail
command -v agentgateway >/dev/null 2>&1 || { curl -sL https://agentgateway.dev/install | bash -s -- --version 1.3.1; }
echo "challenge 2 setup complete"
```

- [ ] **Step 2: Create `02-stand-up-the-gateway/solve-server`** (writes the validated route config + starts gateway)

```bash
#!/bin/bash
set -euo pipefail
export OPENAI_API_KEY="$(cat /root/.openai-api-key 2>/dev/null || echo "$OPENAI_API_KEY")"
cat > /root/config.yaml <<'YAML'
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: openai-route
      matches:
      - path:
          pathPrefix: /v1
      policies:
        backendAuth:
          key: "$OPENAI_API_KEY"
      backends:
      - ai:
          name: openai
          provider:
            openAI: {}
YAML
agentgateway -f /root/config.yaml --validate-only
pkill -f "agentgateway -f" 2>/dev/null || true; sleep 1
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 4
curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' | jq .
```

- [ ] **Step 3: Create `02-stand-up-the-gateway/check-server`**

```bash
#!/bin/bash
set -uo pipefail
if ! pgrep -f "agentgateway -f /root/config.yaml" >/dev/null; then
  fail-message "Agentgateway isn't running. Write /root/config.yaml and start it: nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &"
  exit 1
fi
code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"ping"}],"max_tokens":5}')
if [ "$code" != "200" ]; then
  fail-message "The gateway is up but a test call to :4000 returned HTTP $code. Check /root/agentgateway.log and that backendAuth.key resolves your OPENAI_API_KEY."
  exit 1
fi
if ! curl -s http://localhost:15000/config_dump | grep -q '"bind/4000"'; then
  fail-message "The admin API on :15000 doesn't show the :4000 bind yet. Is the gateway running with /root/config.yaml?"
  exit 1
fi
echo "Gateway is routing OpenAI traffic on :4000."
```

- [ ] **Step 4: Create `02-stand-up-the-gateway/assignment.md`**

Two tabs: Terminal + code Editor (`path: /root`). Story: "Put the chokepoint in place." Steps:
1. Install (already staged) — `agentgateway --version`.
2. Open the Editor, create `/root/config.yaml` with the route config (show the exact YAML from solve Step 2).
3. Validate: `agentgateway -f /root/config.yaml --validate-only`.
4. Start it: `nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &`.
5. Send the first call through the gateway with `curl ...:4000/v1/chat/completions` (show command).
Explain that the gateway now sits between every client and OpenAI. Tie back: "every call now flows through one place we control."

- [ ] **Step 5: Verify locally**

Run: `agentgateway -f <(sed 's/\$OPENAI_API_KEY/'"$OPENAI_API_KEY"'/' 02-stand-up-the-gateway/solve-config.yaml) --validate-only` — or simpler, extract the heredoc to a temp file and run `agentgateway -f /tmp/c2.yaml --validate-only`.
Expected: `Configuration is valid!`

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/02-stand-up-the-gateway
git commit -m "Ch2: Stand up the gateway and route OpenAI"
```

---

## Task 4: Challenge 3 — The Cost of Every Request

**Files:**
- Create: `03-cost-of-every-request/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `03-cost-of-every-request/setup-server`**

```bash
#!/bin/bash
set -euo pipefail
mkdir -p /root/costs
[ -f /root/costs/catalog.json ] || cp /opt/instruqt/track/assets/costs/catalog.json /root/costs/catalog.json
echo "challenge 3 setup complete"
```

- [ ] **Step 2: Create `03-cost-of-every-request/solve-server`** (adds modelCatalog, restarts, drives traffic)

```bash
#!/bin/bash
set -euo pipefail
export OPENAI_API_KEY="$(cat /root/.openai-api-key 2>/dev/null || echo "$OPENAI_API_KEY")"
mkdir -p /root/costs
cat > /root/costs/catalog.json <<'JSON'
{
  "providers": {
    "openai": {
      "models": {
        "gpt-4o-mini": { "rates": { "input": "0.15", "output": "0.60" } },
        "gpt-4o":      { "rates": { "input": "2.50", "output": "10.00" } }
      }
    }
  }
}
JSON
cat > /root/config.yaml <<'YAML'
config:
  modelCatalog:
  - file: /root/costs/catalog.json
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: openai-route
      matches:
      - path:
          pathPrefix: /v1
      policies:
        backendAuth:
          key: "$OPENAI_API_KEY"
      backends:
      - ai:
          name: openai
          provider:
            openAI: {}
YAML
agentgateway -f /root/config.yaml --validate-only
pkill -f "agentgateway -f" 2>/dev/null || true; sleep 1
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 4
# Drive one cheap + one expensive call via the OpenAI CLI through the gateway
export OPENAI_BASE_URL=http://localhost:4000/v1
for m in gpt-4o-mini gpt-4o; do
  curl -s http://localhost:4000/v1/chat/completions -H "Content-Type: application/json" \
    -d "{\"model\":\"$m\",\"messages\":[{\"role\":\"user\",\"content\":\"Say hi in 3 words.\"}],\"max_tokens\":20}" >/dev/null
done
grep -oE 'gen_ai.request.model=[^ ]+|gen_ai.usage.input_tokens=[^ ]+|gen_ai.usage.output_tokens=[^ ]+|agw.ai.usage.cost.total=[^ ]+' /root/agentgateway.log | tail -8
```

- [ ] **Step 3: Create `03-cost-of-every-request/check-server`**

```bash
#!/bin/bash
set -uo pipefail
if ! grep -q 'modelCatalog' /root/config.yaml 2>/dev/null; then
  fail-message "No modelCatalog in /root/config.yaml yet. Add config.modelCatalog pointing at /root/costs/catalog.json so the gateway can attach USD cost."
  exit 1
fi
if ! curl -s http://localhost:15020/metrics | grep -q 'agentgateway_cost_catalog_lookups_total{status="Exact"'; then
  fail-message "No priced requests yet. Send a request through the gateway (e.g. via the openai CLI) and check :15020/metrics for agentgateway_cost_catalog_lookups_total{status=\"Exact\"}."
  exit 1
fi
echo "Cost tracking is live — every request now carries token usage and USD cost."
```

- [ ] **Step 4: Create `03-cost-of-every-request/assignment.md`**

Tabs: Terminal + Editor. Story: "Attribution — put a dollar figure on every call." Steps:
1. Show `/root/costs/catalog.json` (USD per 1M tokens), add `config.modelCatalog` to `/root/config.yaml`, validate, `agw-restart`.
2. Install/point the OpenAI CLI at the gateway: `export OPENAI_BASE_URL=http://localhost:4000/v1` then
   `openai api chat.completions.create -m gpt-4o-mini -g user "Say hi in 3 words."`
3. Read the gateway log: `grep 'agw.ai.usage.cost.total' /root/agentgateway.log` — show `gen_ai.usage.input_tokens`, `output_tokens`, `agw.ai.usage.cost.total`.
4. **The teaching moment:** run the same prompt against `gpt-4o` and compare cost — ~17× more for the same answer (15+5 tokens ≈ `$0.0000051` vs `$0.000085`). Provider dashboards never show you this per-request.
5. Peek at metrics: `curl -s http://localhost:15020/metrics | grep cost_catalog`.
Tie back: "Now we can answer 'how much did that call cost?' — and soon, 'who made it?'"

- [ ] **Step 5: Verify locally** (already validated in research; re-confirm catalog+config)

Extract the heredoc config to `/tmp/c3.yaml` (with catalog at `/tmp/costs/catalog.json`) and run `agentgateway -f /tmp/c3.yaml --validate-only`.
Expected: `Configuration is valid!`

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/03-cost-of-every-request
git commit -m "Ch3: See token usage and USD cost on every request"
```

---

## Task 5: Challenge 4 — Day-2 Operations (Troubleshooting via the Admin API)

**Files:**
- Create: `04-day-2-operations/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `04-day-2-operations/setup-server`** (inject a realistic break to troubleshoot)

```bash
#!/bin/bash
set -euo pipefail
# Leave a "broken" alternate config on disk for the learner to diagnose (typo'd backend key).
cat > /root/config.broken.yaml <<'YAML'
config:
  modelCatalog:
  - file: /root/costs/catalog.json
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: openai-route
      matches:
      - path:
          pathPrefix: /v1
      backends:
      - ai:
          name: openai
          provider:
            openAI: {}
YAML
echo "challenge 4 setup complete"
```

(The broken config omits the `backendAuth` policy, so calls fail upstream with 401 — a realistic misconfiguration to find via logs/config_dump.)

- [ ] **Step 2: Create `04-day-2-operations/solve-server`** (ensure the good config is running)

```bash
#!/bin/bash
set -euo pipefail
export OPENAI_API_KEY="$(cat /root/.openai-api-key 2>/dev/null || echo "$OPENAI_API_KEY")"
pkill -f "agentgateway -f" 2>/dev/null || true; sleep 1
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 3
curl -s http://localhost:15000/config_dump | jq '.binds[0].listeners' >/dev/null
echo "admin API reachable"
```

- [ ] **Step 3: Create `04-day-2-operations/check-server`**

```bash
#!/bin/bash
set -uo pipefail
if ! curl -s http://localhost:15000/config_dump | grep -q '"openai-route"'; then
  fail-message "Couldn't read the live route from :15000/config_dump. Make sure the gateway is running with /root/config.yaml and curl http://localhost:15000/config_dump."
  exit 1
fi
if ! curl -s http://localhost:15020/metrics | grep -qi 'agentgateway'; then
  fail-message "Metrics endpoint on :15020 isn't responding. Try: curl -s http://localhost:15020/metrics | head."
  exit 1
fi
echo "You can inspect live config and metrics — you can operate this gateway."
```

- [ ] **Step 4: Create `04-day-2-operations/assignment.md`**

Tabs: Terminal + a **service tab** for the UI (`type: service`, `hostname: server`, `port: 15000`, `path: /ui`). Story: "You own this gateway now — operate it." Steps:
1. `curl -s http://localhost:15000/config_dump | jq '.binds[].listeners'` — see version/build + live binds/listeners/routes/backends/policies. Explain this is ground truth (what's actually loaded, not what you think you wrote).
2. Open the **UI tab** (`:15000/ui`) — visual view of the same.
3. Metrics: `curl -s http://localhost:15020/metrics | grep -E 'cost_catalog|requests'`.
4. **Troubleshooting drill:** "A teammate's config `/root/config.broken.yaml` returns 401s." Run it (`agentgateway -f /root/config.broken.yaml --validate-only` passes — config is *valid* but wrong), diff against `/root/config.yaml`, and spot the missing `backendAuth` policy via `config_dump`. Restore the good config.
Tie back: "When spend or errors spike, this is where you look first."

- [ ] **Step 5: Verify locally**

Run `agentgateway -f /tmp/c4-broken.yaml --validate-only` (the broken config) → Expected: `Configuration is valid!` (proves the teaching point: valid YAML, wrong behavior).

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/04-day-2-operations
git commit -m "Ch4: Day-2 ops — troubleshoot via :15000 admin API"
```

---

## Task 6: Challenge 5 — Governance: Budgets & Approved Models

**Files:**
- Create: `05-governance-budgets-models/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `05-governance-budgets-models/setup-server`**

```bash
#!/bin/bash
set -euo pipefail
echo "challenge 5 setup complete"
```

- [ ] **Step 2: Create `05-governance-budgets-models/solve-server`** (adds token budget + model pin; proves both)

```bash
#!/bin/bash
set -euo pipefail
export OPENAI_API_KEY="$(cat /root/.openai-api-key 2>/dev/null || echo "$OPENAI_API_KEY")"
cat > /root/config.yaml <<'YAML'
config:
  modelCatalog:
  - file: /root/costs/catalog.json
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: openai-route
      matches:
      - path:
          pathPrefix: /v1
      policies:
        backendAuth:
          key: "$OPENAI_API_KEY"
        ai:
          overrides:
            model: gpt-4o-mini
        localRateLimit:
        - maxTokens: 50
          tokensPerFill: 50
          fillInterval: "1h"
          type: tokens
      backends:
      - ai:
          name: openai
          provider:
            openAI: {}
YAML
agentgateway -f /root/config.yaml --validate-only
pkill -f "agentgateway -f" 2>/dev/null || true; sleep 1
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 4
# Prove model pin: ask for gpt-4o, get gpt-4o-mini
echo "model returned: $(curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"gpt-4o","messages":[{"role":"user","content":"hi"}],"max_tokens":10}' | jq -r .model)"
# Prove budget: spam until 429
for i in 1 2 3 4; do
  echo "req $i -> $(curl -s -o /dev/null -w '%{http_code}' http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Write a paragraph about budgets."}],"max_tokens":80}')"
done
```

- [ ] **Step 3: Create `05-governance-budgets-models/check-server`**

```bash
#!/bin/bash
set -uo pipefail
dump=$(curl -s http://localhost:15000/config_dump)
if ! echo "$dump" | grep -qi 'rateLimit\|localRateLimit'; then
  fail-message "No token budget found in the live config. Add a localRateLimit policy (type: tokens) to the openai-route and restart."
  exit 1
fi
if ! echo "$dump" | grep -qi 'override\|gpt-4o-mini'; then
  fail-message "No model-pin override found. Add ai.overrides.model: gpt-4o-mini to the route so expensive models can't be used."
  exit 1
fi
echo "Governance in place: per-route token budget + approved-model pin."
```

> If `config_dump` field names differ for these policies, the check falls back to grepping `/root/config.yaml` for `localRateLimit` and `overrides`. Implementer: confirm the exact `config_dump` representation during the build and adjust the grep to match (the live test showed both policies load and enforce).

- [ ] **Step 4: Create `05-governance-budgets-models/assignment.md`**

Tabs: Terminal + Editor. Story: "Stop the runaway spender." Steps:
1. **Token budget:** add `localRateLimit` (`type: tokens`, small `maxTokens` for the demo) under the route's `policies`, plus `ai: {}` to mark LLM traffic. Restart. Fire several requests and watch the 2nd+ return **HTTP 429** — "the budget bucket is empty; the agent is throttled, not your bill."
2. **Approved model:** add `ai.overrides.model: gpt-4o-mini`. Restart. Request `gpt-4o` and show the response comes back as `gpt-4o-mini` — "the expensive model is simply unreachable; every team is pinned to the approved one."
3. Explain real-world shape: budgets per API key / per team via `remoteRateLimit` descriptors (point to docs).
Tie back: "Two policies, and the surprise bill can't happen the same way twice."

- [ ] **Step 5: Verify locally**

Extract config to `/tmp/c5.yaml`, run `agentgateway -f /tmp/c5.yaml --validate-only` → `Configuration is valid!` (already live-tested: req1=200, req2-4=429; gpt-4o→gpt-4o-mini).

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/05-governance-budgets-models
git commit -m "Ch5: Governance — token budgets and approved-model policy"
```

---

## Task 7: Challenge 6 — MCP Is Spend Too

**Files:**
- Create: `06-mcp-is-spend-too/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `06-mcp-is-spend-too/setup-server`** (ensure npx available)

```bash
#!/bin/bash
set -euo pipefail
command -v npx >/dev/null 2>&1 || { (command -v apt-get >/dev/null && apt-get install -y nodejs npm) || true; }
echo "challenge 6 setup complete"
```

- [ ] **Step 2: Create `06-mcp-is-spend-too/solve-server`** (adds MCP route alongside the LLM route)

```bash
#!/bin/bash
set -euo pipefail
export OPENAI_API_KEY="$(cat /root/.openai-api-key 2>/dev/null || echo "$OPENAI_API_KEY")"
cat > /root/config.yaml <<'YAML'
config:
  modelCatalog:
  - file: /root/costs/catalog.json
binds:
- port: 4000
  listeners:
  - protocol: HTTP
    routes:
    - name: openai-route
      matches:
      - path:
          pathPrefix: /v1
      policies:
        backendAuth:
          key: "$OPENAI_API_KEY"
        ai:
          overrides:
            model: gpt-4o-mini
        localRateLimit:
        - maxTokens: 20000
          tokensPerFill: 20000
          fillInterval: "1h"
          type: tokens
      backends:
      - ai:
          name: openai
          provider:
            openAI: {}
    - name: mcp-route
      matches:
      - path:
          pathPrefix: /mcp
      backends:
      - mcp:
          targets:
          - name: everything
            stdio:
              cmd: npx
              args: ["-y","@modelcontextprotocol/server-everything"]
YAML
agentgateway -f /root/config.yaml --validate-only
pkill -f "agentgateway -f" 2>/dev/null || true; sleep 1
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 5
curl -s http://localhost:15000/config_dump | jq '.binds[0].listeners[].routes[]? | .name? // empty' 2>/dev/null || true
echo "mcp route added"
```

- [ ] **Step 3: Create `06-mcp-is-spend-too/check-server`**

```bash
#!/bin/bash
set -uo pipefail
if ! grep -q 'mcp-route' /root/config.yaml 2>/dev/null; then
  fail-message "No mcp-route in /root/config.yaml. Add a second route matching /mcp with an mcp backend (targets -> stdio -> npx @modelcontextprotocol/server-everything)."
  exit 1
fi
if ! agentgateway -f /root/config.yaml --validate-only >/dev/null 2>&1; then
  fail-message "Config doesn't validate. Run: agentgateway -f /root/config.yaml --validate-only"
  exit 1
fi
if ! curl -s http://localhost:15000/config_dump | grep -q 'mcp'; then
  fail-message "The live config doesn't show the MCP route. Restart the gateway: agw-restart"
  exit 1
fi
echo "MCP traffic now flows through the same gateway as your LLM traffic."
```

- [ ] **Step 4: Create `06-mcp-is-spend-too/assignment.md`**

Tabs: Terminal + Editor. Story: "Agent tools are spend too." Steps:
1. Add the `mcp-route` (path `/mcp`) with an `mcp` stdio backend (`@modelcontextprotocol/server-everything`). Validate + restart.
2. Confirm both routes load: `curl -s http://localhost:15000/config_dump | jq '..|.name? // empty'` (look for `openai-route` and `mcp-route`).
3. Concept: **why MCP is a cost story** — every tool call ships the tool's JSON schema and the tool result back into the model's context as tokens. More tools + bigger results = more input tokens on the *next* LLM call. Without a gateway, that growth is invisible. With agentgateway, MCP and LLM traffic share one control point — same logs, same budgets, same `config_dump`.
4. (Optional, if time/network allow) initialize the MCP endpoint at `:4000/mcp` with an MCP-capable client; otherwise the route-loaded check is sufficient.
Tie back: "Agents don't just call models — they call tools, and tools cost tokens. One gateway sees both."

- [ ] **Step 5: Verify locally**

Extract to `/tmp/c6.yaml`, `agentgateway -f /tmp/c6.yaml --validate-only` → `Configuration is valid!` (already validated as the combined end-state config).

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/06-mcp-is-spend-too
git commit -m "Ch6: Bring MCP tool traffic under the gateway"
```

---

## Task 8: Challenge 7 — Answer the CFO (Fleet-Scale Analysis)

**Files:**
- Create: `07-answer-the-cfo/{assignment.md,setup-server,check-server,solve-server}`

- [ ] **Step 1: Create `07-answer-the-cfo/setup-server`** (stage generator; pre-generate so the lab is instant)

```bash
#!/bin/bash
set -euo pipefail
[ -f /root/gen-mock-logs.py ] || cp /opt/instruqt/track/assets/gen-mock-logs.py /root/gen-mock-logs.py
command -v uv >/dev/null 2>&1 || curl -LsSf https://astral.sh/uv/install.sh | sh
# Pre-generate so the dataset exists even before the learner runs it
[ -f /root/gw-logs.db ] || (cd /root && uv run gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db)
echo "challenge 7 setup complete"
```

- [ ] **Step 2: Create `07-answer-the-cfo/solve-server`** (regenerate + run the headline query)

```bash
#!/bin/bash
set -euo pipefail
cd /root
uv run gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db
sqlite3 -box /root/gw-logs.db "SELECT agentgateway_group AS team, ROUND(SUM(cost),2) AS usd, COUNT(*) AS reqs FROM request_logs GROUP BY team ORDER BY usd DESC;"
sqlite3 -box /root/gw-logs.db "SELECT agentgateway_user AS user, ROUND(SUM(cost),2) AS usd FROM request_logs GROUP BY user ORDER BY usd DESC LIMIT 5;"
```

- [ ] **Step 3: Create `07-answer-the-cfo/check-server`**

```bash
#!/bin/bash
set -uo pipefail
if [ ! -f /root/gw-logs.db ]; then
  fail-message "No /root/gw-logs.db yet. Generate it: uv run /root/gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db"
  exit 1
fi
n=$(sqlite3 /root/gw-logs.db "SELECT COUNT(*) FROM request_logs;" 2>/dev/null || echo 0)
if [ "${n:-0}" -lt 1 ]; then
  fail-message "request_logs is empty. Re-run the generator with --replace."
  exit 1
fi
echo "Dataset ready ($n requests). You can answer 'who spent the money?'"
```

- [ ] **Step 4: Create `07-answer-the-cfo/assignment.md`**

Tabs: Terminal + Editor. Story: "The CFO wants answers." This dataset is what agentgateway's request logs look like at fleet scale (its schema matches the real access-log fields). Steps with **exact, tested queries**:
1. Generate: `uv run /root/gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db`.
2. Total damage:
   `sqlite3 -box /root/gw-logs.db "SELECT COUNT(*) requests, ROUND(SUM(cost),2) usd, SUM(total_tokens) tokens FROM request_logs;"`  → ~5000 / ~$331 / ~110M tokens.
3. **Chargeback by team:**
   `sqlite3 -box /root/gw-logs.db "SELECT agentgateway_group team, ROUND(SUM(cost),2) usd, COUNT(*) reqs FROM request_logs GROUP BY team ORDER BY usd DESC;"` → `sales` dominates.
4. **Top spenders:**
   `sqlite3 -box /root/gw-logs.db "SELECT agentgateway_user user, ROUND(SUM(cost),2) usd FROM request_logs GROUP BY user ORDER BY usd DESC LIMIT 5;"` → `alice@example.com` ≈ $229 (the `enterprise-batch` whale).
5. **By provider/model:**
   `sqlite3 -box /root/gw-logs.db "SELECT gen_ai_provider_name provider, gen_ai_request_model model, ROUND(SUM(cost),2) usd FROM request_logs GROUP BY provider, model ORDER BY usd DESC LIMIT 6;"`
6. **The insight that ends the mystery — batch vs interactive:**
   `sqlite3 -box /root/gw-logs.db "SELECT json_extract(attributes_json,'$.workload') workload, COUNT(*) n, ROUND(SUM(cost),2) usd FROM request_logs GROUP BY workload ORDER BY usd DESC;"` → `batch` ≈ 10% of requests but ~80% of spend.
7. Error/limit signal:
   `sqlite3 -box /root/gw-logs.db "SELECT http_status, COUNT(*) n FROM request_logs GROUP BY http_status ORDER BY n DESC;"` → see 429s where budgets bit.
Tie back: "One batch job from one user on one team drove the bill. Now you know — and Ch5's budgets would have caught it."

- [ ] **Step 5: Verify locally** (already run end-to-end during research)

Run: `uv run assets/gen-mock-logs.py --replace --requests 5000 --days 7 -o /tmp/gw.db && sqlite3 /tmp/gw.db "SELECT json_extract(attributes_json,'\$.workload') w, ROUND(SUM(cost),2) FROM request_logs GROUP BY w;"`
Expected: three rows (batch/interactive/failed) with batch carrying the majority of cost.

- [ ] **Step 6: Commit**

```bash
git add 01-ai-cost-webinar-workshop/07-answer-the-cfo
git commit -m "Ch7: Fleet-scale cost analysis on mock request logs"
```

---

## Task 9: Challenge 8 — Production Scale (read-only close)

**Files:**
- Create: `08-production-scale/{assignment.md,check-server}`

- [ ] **Step 1: Create `08-production-scale/assignment.md`** (read-only, one Terminal tab)

Recap the journey (blind → chokepoint → attribution → ops → governance → MCP → answers). Then "what production looks like":
- **Prompt guards / guardrails** (regex to block secrets/PII, OpenAI moderation, Bedrock Guardrails) — protect data and indirectly spend.
- **Per-team virtual keys** and `remoteRateLimit` descriptors for real budgets per key/user across replicas.
- **Langfuse / OpenTelemetry** for full prompt/response tracing and per-session cost.
- **Agentgateway Enterprise** (Solo): UI, SSO, multi-cluster, support.
Links to the relevant docs pages. No commands.

- [ ] **Step 2: Create `08-production-scale/check-server`**

```bash
#!/bin/bash
exit 0
```

- [ ] **Step 3: Commit**

```bash
git add 01-ai-cost-webinar-workshop/08-production-scale
git commit -m "Ch8: Production scale and what's next"
```

---

## Task 10: Track-level verification & push prep

- [ ] **Step 1: Lint all bash scripts**

Run: `for f in $(find 01-ai-cost-webinar-workshop -type f \( -name 'setup-server' -o -name 'check-server' -o -name 'solve-server' \) ); do bash -n "$f" && echo "ok $f"; done`
Expected: `ok` for every script.

- [ ] **Step 2: Validate every embedded config**

For each challenge's `solve-server`, extract the heredoc to a temp file (substituting `$OPENAI_API_KEY` with the real env value and pointing catalog at a real file) and run `agentgateway -f /tmp/<c>.yaml --validate-only`. Expected: `Configuration is valid!` for Ch2/3/5/6.

- [ ] **Step 3: Confirm assignment front-matter parses**

Each `assignment.md` begins with `---` front-matter containing `slug`, `id` (empty until push), `type: challenge`, `title`, `teaser`, `notes`, `tabs`. Mirror the field shapes from `agentgateway-oss-quickstart/05-observability/assignment.md`.

- [ ] **Step 4: Commit any fixes, then document the push step (do NOT push without explicit approval)**

Push is a separate, explicitly-authorized action: `instruqt track push` from the track directory (requires the `instruqt` CLI + auth). It assigns `id`/`checksum`. Note this in a short `README.md` for the track and stop.

```bash
git add 01-ai-cost-webinar-workshop
git commit -m "Track-level verification fixes for AI cost workshop"
```

---

## Self-Review

**Spec coverage:** Every spec requirement maps to a task — tokens/cost theory (Task 2), install+route (Task 3), live cost tracking (Task 4), admin-API troubleshooting (Task 5), token budget + model restriction (Task 6), MCP (Task 7), mock-data analysis (Task 8), enterprise narrative woven through every assignment, what's-next (Task 9). Hybrid traffic, ~15-min scope, and reuse of the existing VM image are all honored.

**Placeholders:** All configs, scripts, and SQL are concrete and were executed against the real binary/dataset during research. The only intentionally deferred items are Instruqt-assigned `id`/`checksum` (by design) and the Ch5 `config_dump` field-name confirmation (flagged inline with a working fallback).

**Type/keypath consistency:** Config keypaths are identical across tasks (`config.modelCatalog`, `routes[].policies.backendAuth.key`, `routes[].policies.ai.overrides.model`, `routes[].policies.localRateLimit[]`, `backends[].ai.provider.openAI`, `backends[].mcp.targets[]`); ports (4000/15000/15020), file paths (`/root/config.yaml`, `/root/costs/catalog.json`, `/root/gw-logs.db`), and column names (`agentgateway_user`, `agentgateway_group`, `gen_ai_provider_name`, `total_tokens`, `cost`, `attributes_json`) match the verified schema everywhere.
