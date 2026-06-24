---
slug: day-2-operations
id: a3xmnpwlqjxt
type: challenge
title: Explore the Gateway UI
teaser: Take the Agentgateway web UI for a spin — live costs, logs, config, and client
  setup.
notes:
- type: text
  contents: "# \U0001F9ED Explore the Gateway UI\n\nEverything you've done in YAML,
    Agentgateway also gives you in a **web UI** —\nyour mission control for AI traffic.
    You'll drive a few calls from the Terminal,\nthen watch them show up as costs,
    logs, and charts — and grab a ready-made\nsnippet to point any app at the gateway.\n"
tabs:
- id: fg2dsy86jle2
  title: Terminal
  type: terminal
  hostname: server
- id: 09qu2qpqmkri
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Explore the Gateway UI

Open the **Agentgateway UI** tab (served on `:15000/ui`). The left nav groups
everything: **LLM** (Models, Providers, Policies, Costs…), **MCP**, **Traffic**,
and **Tools**.

> **Where do live calls come from?** From the **Terminal**, not the browser. The
> Terminal runs *on* the gateway VM, so `localhost:4000` is the gateway. (The UI's
> in-browser Chat Playground talks to `:4000` from *your* laptop, which can't reach
> the VM through the hosted lab — so we drive traffic from the Terminal and use the
> UI to *observe* it.)

## 1. Generate some traffic (Terminal)

In the **Terminal**, fire a few calls through the gateway so the UI has data to show:

```bash
for q in "What is an AI gateway?" "Summarize MCP in one line." "Name three FinOps metrics."; do
  curl -s http://localhost:4000/v1/chat/completions -H 'Content-Type: application/json' \
    -d "{\"model\":\"openai/gpt-4.1-nano\",\"messages\":[{\"role\":\"user\",\"content\":\"$q\"}],\"max_tokens\":40}" \
    | jq -r '.choices[0].message.content'
done
```

Three real replies — each priced and logged by the gateway. Now go look at them.

## 2. Costs & Analytics — the spend, visualized

Left nav → **Costs** (and **Analytics**). The calls you just made appear here as
**charts** — spend by model, token volume, request counts — no SQL required. This
is the manager-facing view of the same data you'll query directly later.

## 3. Logs — every request, line by line

Left nav → **Logs**. Each call you sent is a row: model, tokens, cost, status,
latency. This is the per-request detail behind the charts.

## 4. Models & Providers — what's configured

Left nav → **Models** and **Providers**. Here's your `openai/*` model and its
OpenAI provider — the same thing you defined in YAML, shown as managed objects you
can edit in place.

## 5. Raw Configuration — the whole gateway, in one place

Left nav → **Tools → Raw Configuration**. This is the **full gateway YAML**, live
and editable — your `config.database`, `modelCatalog`, `llm.models`, everything.
**Copy**, **Download**, or edit and **Save** (it hot-reloads) — the same config you'd
otherwise hand-edit in `/root/config.yaml`.

## 6. Client Setup — point any app at the gateway

Left nav → **Client Setup**. This **generates connection snippets** for any
OpenAI-compatible client:

- It shows your **Gateway Base URL** (`…:4000/v1`), the **model**, and the **auth** header
- Switch the **Integration** dropdown between **curl**, the **OpenAI Python/Node SDKs**, and more
- Hit **Copy** — that's all a developer needs to route their app through your gateway

This is the "make it yours" button: any team drops in that base URL and their
traffic is instantly metered, priced, and governed.

> You've seen the gateway from every angle. Next: put **guardrails on the spend**
> so it can't run away. ➡️
