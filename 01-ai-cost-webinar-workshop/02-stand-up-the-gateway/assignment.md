---
slug: stand-up-the-gateway
id: 9nswgbezzcg0
type: challenge
title: Stand Up the Gateway
teaser: Put a single chokepoint in front of all your OpenAI traffic.
notes:
- type: text
  contents: "# \U0001F6A6 Stand Up the Gateway\n\nStep one to regaining control: route
    every LLM call through **one place you\noperate**. You'll install standalone Agentgateway,
    point it at OpenAI, and\nsend your first call through it.\n"
tabs:
- id: lpcnur5ga7bv
  title: Terminal
  type: terminal
  hostname: server
- id: nlc8xtx0f7ar
  title: Editor
  type: code
  hostname: server
  path: /root
- id: fxrbngp7zx31
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Stand Up the Gateway

Agentgateway OSS is a single Rust binary — no Kubernetes, no sidecars. You give
it a config file and it becomes an **OpenAI-compatible proxy** your apps point at
instead of `api.openai.com`. That one redirect is how you get visibility and
control over everything.

## Step 1 — Confirm the binary

```bash
agentgateway --version
```

You should see version `1.3.x`.

## Step 2 — Create the config

A starter **`/root/config.yaml`** is already waiting in the **Editor** tab —
open it from the file tree on the left to read along. The simplest way to write
it is to **paste this into the Terminal** (it creates or overwrites the file):

```bash
cat > /root/config.yaml <<'EOF'
config:
  # expose the admin UI (:15000/ui) so the "Agentgateway UI" tab can reach it
  adminAddr: "0.0.0.0:15000"
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
EOF
```

What this says: listen on **:4000**, accept OpenAI-style `/v1/...` calls, attach
your OpenAI key with the `backendAuth` policy, and forward to OpenAI. The
`$OPENAI_API_KEY` is read from the environment — clients never see the real key.

> 💡 Prefer the Editor? Just edit the pre-loaded `/root/config.yaml` and save
> (`Ctrl/Cmd-S`) — no need to create a new file.

## Step 3 — Validate before you run

```bash
agentgateway -f /root/config.yaml --validate-only
```

Expected: `Configuration is valid!` (Get used to this — it's your fastest
feedback loop.)

## Step 4 — Start the gateway

```bash
nohup agentgateway -f /root/config.yaml > /root/agentgateway.log 2>&1 &
sleep 3
```

It's now listening on **:4000** (API) and **:15000** (admin/UI). More on
:15000 soon.

## Step 5 — Send your first call *through the gateway*

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' | jq .
```

You'll get a normal OpenAI response — but it went through **your** gateway. Every
client that points at `:4000` instead of OpenAI directly is now under your
control point.

## Step 6 — Explore the Agentgateway UI

Open the **Agentgateway UI** tab (served at `:15000/ui`). This standalone UI is
your window into the gateway — listeners, routes, backends, a request **Playground**,
and live traffic. You'll come back to it in later challenges to watch token
usage and cost in real time. Click around and get oriented.

> Next: make that control point put a **dollar figure on every call**. ➡️
