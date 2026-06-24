---
slug: answer-the-cfo
id: xnjzpvrc6jc7
type: challenge
title: Answer the CFO
teaser: Use fleet-scale gateway logs to finally answer "who spent the money?"
notes:
- type: text
  contents: "# \U0001F4CA Answer the CFO\n\nBack to the original question. A week
    of gateway request logs across 12\nusers and 4 providers is waiting in SQLite
    — the same fields Agentgateway\nrecords in production. Time to find out where
    the money went.\n"
tabs:
- id: upptowlm0w23
  title: Terminal
  type: terminal
  hostname: server
- id: vdysbceelims
  title: Editor
  type: code
  hostname: server
  path: /root
- id: 1hk6nljxy8am
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Answer the CFO

## Step 0 — Your gateway already kept the receipts

Since Challenge 3, `config.database` has been writing **every request you made**
to a SQLite table at `/root/data/data.db`. Look at your own real traffic:

```bash
sqlite3 -box /root/data/data.db \
  "SELECT gen_ai_request_model model, COUNT(*) calls, ROUND(SUM(cost),6) usd \
   FROM request_logs GROUP BY model ORDER BY usd DESC;"
```

That's real — but it's only the handful of calls *you* made in this lab. Open the
**Agentgateway UI** tab → **Costs**: those same calls are already charted, no SQL
required. To answer the CFO, though, you need **fleet scale** — many users, teams,
and providers over a week.

## Step 1 — Add a fleet-scale week to the same database

The generator writes the **exact same `request_logs` schema** Agentgateway uses,
so we append a representative week straight into your gateway's *own* database:

```bash
uv run /root/gen-mock-logs.py --append --requests 5000 --days 7 -o /root/data/data.db
agw-restart
```

Now go back to the **Costs** and **Analytics** pages and refresh. Same UI — but
now it's **fleet scale**, and notice it's **multi-provider**: OpenAI, Anthropic
(Claude), Google, and Bedrock traffic all under one gateway. Everything below you
can also read off those charts; we'll use SQL so you see the exact numbers.

> In production you'd point these queries at your real Agentgateway database or
> log store — the schema is identical.

## Step 2 — The total damage

```bash
sqlite3 -box /root/data/data.db \
  "SELECT COUNT(*) requests, ROUND(SUM(cost),2) usd, SUM(total_tokens) tokens FROM request_logs;"
```

~5,000 requests, **~$331**, ~110M tokens in a week.

## Step 3 — Chargeback by team

```bash
sqlite3 -box /root/data/data.db \
  "SELECT agentgateway_group team, ROUND(SUM(cost),2) usd, COUNT(*) reqs \
   FROM request_logs GROUP BY team ORDER BY usd DESC;"
```

`sales` (~**$233**) dwarfs every other team. Now you can bill it back.

## Step 4 — Name the top spenders

```bash
sqlite3 -box /root/data/data.db \
  "SELECT agentgateway_user user, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY user ORDER BY usd DESC LIMIT 5;"
```

`alice@example.com` alone is ~**$229** of the ~$331. There's your prime suspect.

## Step 5 — Which providers & models — including Claude

```bash
sqlite3 -box /root/data/data.db \
  "SELECT gen_ai_provider_name provider, gen_ai_request_model model, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY provider, model ORDER BY usd DESC LIMIT 6;"
```

Spend is spread across vendors — `openai/gpt-4.1`, `anthropic/claude-sonnet-4-5`,
`google/gemini-2.5-pro`, `bedrock/...`. **This cross-provider view is something no
single-vendor dashboard can give you.** Want just the Claude bill?

```bash
sqlite3 -box /root/data/data.db \
  "SELECT agentgateway_user user, ROUND(SUM(cost),2) usd, COUNT(*) reqs \
   FROM request_logs WHERE gen_ai_provider_name='anthropic' \
   GROUP BY user ORDER BY usd DESC LIMIT 5;"
```

## Step 6 — Which clients and agents are calling

Every request carries the calling client's user-agent. Slice spend by it:

```bash
sqlite3 -box /root/data/data.db \
  "SELECT json_extract(attributes_json,'\$.\"user_agent.name\"') client,
          COUNT(*) reqs, ROUND(SUM(cost),2) usd
   FROM request_logs GROUP BY client ORDER BY usd DESC LIMIT 8;"
```

You can see **Claude Code, Claude Desktop, Codex, Cursor, Gemini CLI** and the
SDKs — every agent and copilot in the org, attributed by name. This is how you
tell "a person in the IDE" from "a batch script."

## Step 7 — The insight that closes the case

```bash
sqlite3 -box /root/data/data.db \
  "SELECT json_extract(attributes_json,'\$.workload') workload, COUNT(*) n, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY workload ORDER BY usd DESC;"
```

**`batch` workloads are ~10% of requests but ~80% (~$263) of the spend.** A handful
of batch jobs — from one user, on one team — drove the bill. Mystery solved.

## Step 8 — Were limits already biting?

```bash
sqlite3 -box /root/data/data.db \
  "SELECT http_status, COUNT(*) n FROM request_logs GROUP BY http_status ORDER BY n DESC;"
```

The `429`s are where budgets kicked in. **The token budget from Challenge 5 is
exactly what would have caught alice's batch job before it ran up $229.**

> You went from a blind five-figure bill to per-team, per-user, per-provider,
> per-client, per-workload attribution — and the policies to prevent the next one.

## You did it 🎉

You started blind — a five-figure bill and no idea who spent it. You finished
with a gateway that turns every AI call into an answer. Along the way you built:

- 💵 **Live cost + token tracking** on every request, priced in real USD
- 🔧 **Day-2 troubleshooting** through the admin API and UI
- 🛡️ **Token budgets** that throttle runaway spend before it bills
- 🎯 **Cost-aware routing** — cheap by default, premium only on demand
- 🏷️ **CEL cost tagging** to slice spend by tier
- 🔌 **MCP tool traffic** under the same roof
- 🔑 **Per-team virtual keys** so the real provider key never leaks
- 📊 **Fleet-scale attribution** — spend by team, user, provider, client, and workload

**Now make it yours:** point any OpenAI client (or agent, or copilot) at
`http://localhost:4000/v1`, and it's instantly metered, priced, and governed —
no app changes. That's the whole pitch: **one gateway, total control of AI spend.**
