---
slug: add-an-llm
id: ss7rwb9xy9yo
type: challenge
title: Add an LLM
teaser: Turn the gateway into an OpenAI-compatible proxy.
notes:
- type: text
  contents: "# \U0001F310 Add an LLM\n\nPoint the gateway at OpenAI and it becomes
    an OpenAI-compatible proxy your apps\ncall instead of api.openai.com — with cost
    tracking on by default.\n"
tabs:
- id: 2x4vmpolxfao
  title: Terminal
  type: terminal
  hostname: server
- id: bev7veegqx0l
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: curkyxyh00us
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Add an LLM

![Lab 2 — OpenAI-compatible proxy](../assets/diagram-02-llm.png)

**What we're building:** an `llm.models` entry so clients call `openai/...` through
`:4000` and the gateway forwards to OpenAI using a key they never see.

## Step 1 — Add an OpenAI model

**Option A — the UI:** open the **Agentgateway UI** → **Models** → **Add model**.
Incoming match `openai/*`, provider **OpenAI**, API key **Env var** `OPENAI_API_KEY`,
**Save**.

**Option B — YAML (Editor):** open `/root/agentgateway/config.yaml` and replace the
empty `models: []` with:

```yaml
  models:
  - name: "openai/*"
    provider: openAI
    params:
      model: gpt-4.1-nano
      apiKey: "$OPENAI_API_KEY"
```

Validate the config, then restart the gateway so it re-reads the file. (The
`--validate-only` run parses your config and exits; pass the key so the
`$OPENAI_API_KEY` reference resolves.)

```bash
docker run --rm -v /root/agentgateway:/config -e OPENAI_API_KEY \
  cr.agentgateway.dev/agentgateway:v1.3.1 -f /config/config.yaml --validate-only

docker restart agentgateway
```

`$OPENAI_API_KEY` is read from the container's environment — clients never see the
real key.

## Step 2 — Send your first call *through* the gateway

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/gpt-4.1-nano","messages":[{"role":"user","content":"Say hello in one sentence."}],"max_tokens":20}' \
  | jq -r '.choices[0].message.content'
```

**What you'll see:** a normal OpenAI response — but it went through **your** gateway.
The bundled cost catalog already priced this call and logged it to the request
database (you'll mine that in Lab 4).

> Next: bring **tool traffic** under the same gateway with a separate MCP server. ➡️
