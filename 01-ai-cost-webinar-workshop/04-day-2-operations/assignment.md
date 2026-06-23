---
slug: day-2-operations
id: ""
type: challenge
title: Day-2 Operations
teaser: Inspect and troubleshoot the gateway with the admin API on :15000.
notes:
- type: text
  contents: |
    # 🔧 Day-2 Operations

    You own this gateway now. When spend spikes or calls start failing, you need
    to see **what's actually loaded and running** — not what you think you
    wrote. That's the admin API on **:15000**.
tabs:
- id: ""
  title: Terminal
  type: terminal
  hostname: server
- id: ""
  title: Agentgateway UI
  type: service
  hostname: server
  port: 15000
  path: /ui
difficulty: ""
enhanced_loading: null
---

# Day-2 Operations

Agentgateway exposes a local admin interface on **:15000** (and Prometheus
metrics on **:15020**). This is your first stop for any incident.

## Step 1 — Dump the live config (ground truth)

```bash
curl -s http://localhost:15000/config_dump | jq '.binds[0].listeners | keys'
curl -s http://localhost:15000/config_dump | jq '.binds[].listeners[].routes[]?.name? // empty'
```

`config_dump` shows the **running** version, build, binds, listeners, routes,
backends, and policies — exactly what the proxy loaded. If reality and your
intent disagree, this is where you find out.

## Step 2 — Use the visual UI

Open the **Agentgateway UI** tab (served at `:15000/ui`). Same information,
visual — routes, backends, and live state.

## Step 3 — Check metrics

```bash
curl -s http://localhost:15020/metrics | grep -E 'cost_catalog|requests' | head
```

Cost-catalog lookups, request counts, latencies — all Prometheus-ready on
`:15020`.

## Step 4 — Troubleshooting drill

A teammate swears their config "looks fine" but every call returns **401**.
Their file is at `/root/config.broken.yaml`. First, does it even validate?

```bash
agentgateway -f /root/config.broken.yaml --validate-only
```

It says **`Configuration is valid!`** — valid YAML, *wrong behavior*. Now diff it
against the working config:

```bash
diff <(grep -v '^\s*#' /root/config.yaml) /root/config.broken.yaml
```

The broken one is **missing the `backendAuth` policy** — so the gateway forwards
to OpenAI with no key and gets 401s. The fix is obvious once you can see the live
vs. intended config side by side.

## Step 5 — Make sure the good config is running

```bash
agw-restart
curl -s http://localhost:15000/config_dump | grep -q backendAuth && echo "auth policy loaded ✅"
```

> Inspecting config and metrics is how you operate this in production. Next:
> stop the spend problem from happening at all — **governance**. ➡️
