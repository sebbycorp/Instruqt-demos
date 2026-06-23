---
slug: the-blind-spot
id: ""
type: challenge
title: The Blind Spot
teaser: Why your AI bill is a mystery — and how tokens actually get spent.
notes:
- type: text
  contents: |
    # 🧾 The Blind Spot

    Finance just forwarded a five-figure OpenAI bill and asked one question:
    **"Who spent this?"**

    You're on the platform team. You can't answer. Let's understand why — and
    what we're going to do about it.
tabs:
- id: ""
  title: Terminal
  type: terminal
  hostname: server
difficulty: ""
enhanced_loading: null
---

# The Blind Spot

It's Monday. Finance forwards a **$40,000 OpenAI invoice** and asks the only
question that matters: *who spent this?* Across the company, agents, copilots
(Cursor, Claude Code), SDKs, and one-off scripts are all calling LLMs and MCP
tools **directly**. There is no shared control point. This is **shadow AI** —
the same loss of control enterprises hit with microservices in 2016, before API
gateways and service meshes brought order.

Before we fix it, you need to understand what you're actually paying for.

## How tokens are spent

LLMs don't bill per request — they bill per **token**. A token is roughly ¾ of a
word. Every request has two metered parts:

- **Input tokens** — everything you send: the system prompt, the conversation
  history, retrieved documents, and tool definitions.
- **Output tokens** — everything the model generates back.

Price is quoted **per 1 million tokens**, and it varies *enormously* by model.
For the same answer, a frontier model can cost **~17× more** than a small one —
you'll measure that exact gap yourself in a few minutes.

## What quietly inflates the bill

- **Long context** — every turn resends the whole conversation. Input tokens
  grow with the chat.
- **Reasoning tokens** — "thinking" models bill for internal reasoning you never
  see.
- **Streaming** — same token cost, but harder to meter after the fact.
- **Embeddings** — cheap per call, brutal in bulk (indexing millions of docs).
- **Tool / function calls (incl. MCP)** — every tool's JSON schema rides along in
  the input, and every tool **result** comes back as more input tokens on the
  next call. More tools = more spend, invisibly. (We'll see this in the MCP
  challenge.)

## Why the provider dashboard doesn't save you

OpenAI's dashboard shows spend **per API key / per account**, **delayed by
hours**, and with **no idea** which team, agent, or human made the call. You
can't do chargeback. You can't set a per-team budget. You can't catch a runaway
batch job before it runs for six hours.

## The fix

Put **one gateway in front of all AI traffic**. Every call flows through a single
place you operate — so you get token usage, **real USD cost**, attribution, and
governance, across every model and provider, in real time.

That gateway is **Agentgateway**. Let's stand it up. ➡️

> ✅ Nothing to run here — click **Check** to continue.
