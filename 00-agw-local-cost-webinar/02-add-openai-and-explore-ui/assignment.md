---
slug: add-openai-and-explore-ui
id: rmaaawyit1rn
type: challenge
title: Add OpenAI & Explore the UI
teaser: Wire in OpenAI with the Instruqt key, send a call, and watch it priced live.
notes:
- type: text
  contents: |
    Your gateway is running but has no models. Now you'll add OpenAI to the config —
    using the Instruqt-provided `OPENAI_API_KEY` environment variable so the real key
    never appears in your YAML — reload, and send a request. Then explore the UI: the
    model, the request, and its exact cost all show up the moment the call completes.
tabs:
- id: 3vk1dkbtnkum
  title: Terminal
  type: terminal
  hostname: server
- id: pxjzuvrkavek
  title: Editor
  type: code
  hostname: server
  path: /root
- id: tkgtc4oxrjqb
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Add OpenAI & Explore the UI

## Step 1 — Add the OpenAI model

In the **Editor**, open `/root/config.yaml` and replace the empty `llm.models` block
(and drop the now-unneeded `providers`/`virtualModels` empties) with a real OpenAI
model. The `apiKey` is a **reference to the environment variable** the gateway already
received — not the secret itself:

```yaml
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
```

`$OPENAI_API_KEY` resolves inside the container from the `-e OPENAI_API_KEY` you passed
when you started it. That's the Instruqt secret — it stays out of your config file.

## Step 2 — Validate and reload

```bash
agw-validate          # parses the config (passes the key in for you)
agw-reload            # restarts the container so it re-reads the mounted config
```

## Step 3 — Send a request

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' \
  | jq -r '.choices[0].message.content'
```

A sentence back means your traffic is flowing through the gateway to OpenAI.

## Step 4 — Explore the UI

Open the **Agentgateway UI** tab and click through:

- **Models** — your `openai/*` route is now listed.
- **Requests / Logs** — the call you just made, with input/output tokens.
- **Costs** — the dollar cost of that request, priced from the catalog.

Send a few more calls and refresh — every one shows up, priced, in real time.

Click **Check** when OpenAI is wired in and answering on `:4000`.
