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

## Step 2 — Write the config

Open the **Editor** tab and create `/root/config.yaml`:

```yaml
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
```

What this says: listen on **:4000**, accept OpenAI-style `/v1/...` calls, attach
your OpenAI key with the `backendAuth` policy, and forward to OpenAI. The
`$OPENAI_API_KEY` is read from the environment — clients never see the real key.

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

> Next: make that control point put a **dollar figure on every call**. ➡️
