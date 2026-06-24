---
slug: start-the-gateway
id: k4ztvwmda40d
type: challenge
title: Start the Gateway in Docker
teaser: One docker run and you have an AI gateway with a web UI on this box.
notes:
- type: text
  contents: |
    Agentgateway ships as a single container image. Before we route any traffic, let's
    just get it running with its web UI so you can see what it gives you. You'll start it
    with a skeleton config (no models yet) — in the next lab you'll wire in OpenAI.
tabs:
- id: la3qx3lg6s0p
  title: Terminal
  type: terminal
  hostname: server
- id: nbyf7nhlxloi
  title: Editor
  type: code
  hostname: server
  path: /root
- id: nacbtwt7aehy
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Start the Gateway in Docker

![All in Docker on one VM: the gateway container fronts OpenAI and an MCP tool server](../assets/diagram-architecture.png)

Everything in this webinar runs in Docker on this one machine. A Docker network
`agw-net` and the cost catalog at `/root/costs/base-costs.json` are already in place.
The skeleton config is at `/root/config.yaml` (open in the **Editor**).

## Step 1 — Run the gateway container

This is the whole command. It mounts your config + cost catalog + a request-log volume
and publishes the three ports — LLM `:4000`, MCP `:3000`, and the UI/admin `:15000`:

```bash
docker run -d --name agentgateway --network agw-net --restart unless-stopped \
  -e OPENAI_API_KEY="$OPENAI_API_KEY" \
  -v /root/config.yaml:/config.yaml \
  -v /root/costs:/costs \
  -v /root/data:/data \
  -p 4000:4000 -p 3000:3000 -p 15000:15000 \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config.yaml
```

That exact command is also wrapped in a helper for later — `agw-up` runs it for you,
and `agw-reload` picks up config edits without retyping it.

## Step 2 — Confirm it's running

```bash
docker ps --filter name=agentgateway
curl -s -o /dev/null -w "UI on :15000 -> %{http_code}\n" http://localhost:15000/ui/
```

A running container and an HTTP code from `:15000` mean you're live.

## Step 3 — Open the UI

Click the **Agentgateway UI** tab. You'll see the dashboard — **Models**, **Requests**,
**Costs** — all empty for now, because we haven't given it any models. That's the next
lab.

Click **Check** once the container is up and the UI answers.

> Why the `-e OPENAI_API_KEY` already, with no models yet? So that when you add an OpenAI
> model next lab, a quick `agw-reload` picks it up — the key is already in the container.
