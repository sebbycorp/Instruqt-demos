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
difficulty: ""
enhanced_loading: null
---

# Answer the CFO

In production, Agentgateway streams its access logs (with `agentgateway.user`,
`agentgateway.group`, `gen_ai.*`, and `agw.ai.usage.cost.*`) to your log store.
We've pre-loaded a representative **week of that data** — 5,000 requests, 12
users, 4 providers — into `/root/gw-logs.db`. The schema matches the real thing,
so these queries are exactly what you'd run against production.

## Step 1 — (Re)generate the dataset

```bash
uv run /root/gen-mock-logs.py --replace --requests 5000 --days 7 -o /root/gw-logs.db
```

## Step 2 — The total damage

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT COUNT(*) requests, ROUND(SUM(cost),2) usd, SUM(total_tokens) tokens FROM request_logs;"
```

~5,000 requests, **~$331**, ~110M tokens in a week.

## Step 3 — Chargeback by team

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT agentgateway_group team, ROUND(SUM(cost),2) usd, COUNT(*) reqs \
   FROM request_logs GROUP BY team ORDER BY usd DESC;"
```

One team (`sales`) dwarfs the rest. Now you can bill it back.

## Step 4 — Name the top spenders

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT agentgateway_user user, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY user ORDER BY usd DESC LIMIT 5;"
```

`alice@example.com` alone is ~$229 of the ~$331. There's your prime suspect.

## Step 5 — Where it went by provider/model

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT gen_ai_provider_name provider, gen_ai_request_model model, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY provider, model ORDER BY usd DESC LIMIT 6;"
```

## Step 6 — The insight that closes the case

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT json_extract(attributes_json,'\$.workload') workload, COUNT(*) n, ROUND(SUM(cost),2) usd \
   FROM request_logs GROUP BY workload ORDER BY usd DESC;"
```

**`batch` workloads are ~10% of requests but ~80% of the spend.** A handful of
batch jobs — from one user, on one team — drove the bill. Mystery solved.

## Step 7 — Were limits already biting?

```bash
sqlite3 -box /root/gw-logs.db \
  "SELECT http_status, COUNT(*) n FROM request_logs GROUP BY http_status ORDER BY n DESC;"
```

The `429`s are where budgets kicked in. **The token budget from Challenge 5 is
exactly what would have caught alice's batch job before it ran up $229.**

> You went from a blind five-figure bill to per-team, per-user, per-workload
> attribution — and the policies to prevent the next one. ➡️
