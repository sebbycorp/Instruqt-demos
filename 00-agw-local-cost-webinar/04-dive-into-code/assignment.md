---
slug: dive-into-code
id: 0apupodciltp
type: challenge
title: Dive Into Code — A Client App
teaser: One small app, both planes of the gateway — OpenAI on :4000 and your tool
  on :3000.
notes:
- type: text
  contents: |
    Now use it like a developer would. You'll write a small Python app that points the
    standard OpenAI SDK at the gateway (so no app code knows the real key) and, in the
    same script, calls the MCP tool you built — both through the one gateway. This is the
    developer experience your platform gives every team.
tabs:
- id: bnjdman6ryyy
  title: Terminal
  type: terminal
  hostname: server
- id: ltrrrxns4cnq
  title: Editor
  type: code
  hostname: server
  path: /root
- id: cxtopealqved
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Dive Into Code — A Client App

The gateway (`:4000` LLM, `:3000` MCP) and your `mcp-custom` server are running. Let's
write an app that uses both.

## Step 1 — Write the app

In the **Editor**, open `/root/app/app.py` and make it talk to both planes. Notice the
OpenAI client just points `base_url` at the gateway — the app never sees the real key:

```python
import asyncio
from openai import OpenAI
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

GATEWAY_LLM = "http://localhost:4000/v1"
GATEWAY_MCP = "http://localhost:3000"

def ask_llm():
    client = OpenAI(base_url=GATEWAY_LLM, api_key="sk-not-needed")
    resp = client.chat.completions.create(
        model="openai/gpt-4.1-nano",
        messages=[{"role": "user", "content": "In one sentence, what is an AI gateway?"}],
        max_tokens=40,
    )
    return resp.choices[0].message.content.strip()

async def use_tools():
    async with streamablehttp_client(GATEWAY_MCP) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            names = [t.name for t in tools.tools]
            result = await session.call_tool(
                "estimate_tokens", {"text": "How many tokens is this sentence?"}
            )
            return names, result.content[0].text

def main():
    print("LLM via gateway :4000 ->", ask_llm())
    names, tokens = asyncio.run(use_tools())
    print("MCP tools via gateway :3000 ->", names)
    print("estimate_tokens result ->", tokens)

if __name__ == "__main__":
    main()
```

## Step 2 — Run it

`uv` fetches the dependencies into a throwaway environment — no virtualenv to manage:

```bash
uv run --with openai --with 'mcp==1.12.4' /root/app/app.py
```

You should see a one-sentence answer from OpenAI, your tool names, and the
`estimate_tokens` result — all served through the single gateway.

## Step 3 — See it in the UI

Refresh the **Agentgateway UI**. The chat call your app just made shows up in
**Requests** with its cost, right next to every other call. Application traffic is now
first-class, observable spend.

Click **Check** when your app runs and prints the tool result.

> The whole point: app developers use the **standard** OpenAI SDK and a **standard** MCP
> client. They point at the gateway and get keys, routing, cost, and tools for free.
