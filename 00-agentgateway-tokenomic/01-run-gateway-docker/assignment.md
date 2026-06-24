---
slug: run-gateway-docker
id: jb3bysq5ryex
type: challenge
title: Run the Gateway in Docker
teaser: Put one Docker container in front of all your AI traffic.
notes:
- type: text
  contents: "# \U0001F433 Run the Gateway in Docker\n\nAgentgateway is a single Rust
    binary — and a single Docker container. Start it\nwith a minimal config and connect
    to its built-in web UI. No Kubernetes, no\nsidecars.\n"
tabs:
- id: eemwgichzjbp
  title: Terminal
  type: terminal
  hostname: server
- id: pjaqr0b8tbkg
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: kuebxvrqykcl
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
