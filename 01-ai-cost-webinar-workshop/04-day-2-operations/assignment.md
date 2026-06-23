---
slug: day-2-operations
id: a3xmnpwlqjxt
type: challenge
title: Explore the Gateway UI
teaser: Take the Agentgateway web UI for a spin — chat playground, live config, and
  client setup.
notes:
- type: text
  contents: "# \U0001F9ED Explore the Gateway UI\n\nEverything you've done in YAML,
    Agentgateway also gives you in a **web UI** —\nyour mission control for AI traffic.
    Let's click around: chat with your models,\nsee the live config, and grab a ready-made
    snippet to point any app at the\ngateway.\n"
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
and **Tools**. No commands this time — just explore.

## 1. Chat Playground — talk to your gateway

Left nav → **Chat Playground**. This sends a **real** chat completion *through the
gateway* (great for setup debugging).

- **Model:** pick `openai/*` (your catch-all model)
- **Specific model:** type `gpt-4o-mini`
- Type a question in **User message** and hit **Send**

You'll get a live reply — and because it went through the gateway, it was just
priced and logged (you'll see it on the Costs/Logs pages). Notice the **Include
MCP tools** toggle too — that lets the model call MCP tools you expose later.

## 2. Raw Configuration — the whole gateway, in one place

Left nav → **Tools → Raw Configuration**. This is the **full gateway YAML**, live
and editable — your `config.database`, `modelCatalog`, `llm.models`, everything.
You can **Copy**, **Download**, or edit and **Save** (it hot-reloads). This is the
same config you'd otherwise hand-edit in `/root/config.yaml`.

## 3. Client Setup — point any app at the gateway

Left nav → **Client Setup**. This **generates connection snippets** for any
OpenAI-compatible client:

- It shows your **Gateway Base URL** (`…:4000/v1`), the **model**, and the **auth** header
- Switch the **Integration** dropdown between **curl**, the **OpenAI Python/Node SDKs**, and more
- Hit **Copy** — that's literally all a developer needs to route their app through your gateway

This is the "make it yours" button: any team drops in that base URL and their
traffic is instantly metered, priced, and governed.

## 4. Look around: Costs, Logs, Analytics

Click **Costs**, **Logs**, and **Analytics** in the LLM section to see the spend
and requests you've already generated — the same data, visualized.

> You've seen the gateway from every angle. Next: put **guardrails on the spend**
> so it can't run away. ➡️
