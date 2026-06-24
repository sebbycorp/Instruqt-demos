---
slug: chat-suspend-resume
id: ysrnblrrigyt
type: challenge
title: Chat and watch suspend/resume
teaser: Chat with your agent in the kagent UI, then drive the actor lifecycle from
  the CLI to watch it go Suspended → Running.
notes:
- type: text
  contents: "# \U0001F50D The payoff\n\nWhen you chat with `hello-substrate`, a per-session
    gVisor **actor** is restored\nfrom the golden snapshot, runs the LLM call, and
    snapshots itself back to object\nstorage — returning the worker to the pool.\n\nBetween
    requests the actor sits **SUSPENDED**. On the next request it's **RESUMED**\nsub-second.
    You'll see this both in the UI's `/substrate` page and directly from the\n`ate-api`
    via `grpcurl`.\n"
tabs:
- id: 230yqdteuzzs
  title: Terminal
  type: terminal
  hostname: server
- id: p5xyfpiaxk3n
  title: kagent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# Chat and watch suspend/resume

## Step 1: Chat with the agent (UI)

Open the **kagent UI** tab. Pick `kagent/hello-substrate` from the Agents list and send:

> *What are you, and where are you running? Answer in one sentence.*

You should get back something like:

> *I am hello-substrate, a Go ADK declarative agent running inside a gVisor actor.*

Then open the **Substrate** page (`/substrate`) in the UI to see the worker pool and actors.

## Step 2: List actors from the CLI

In the **Terminal** tab, port-forward the substrate API in the background:

```bash
kubectl port-forward -n ate-system svc/api 18443:443 >/tmp/pf-api.log 2>&1 &
sleep 3
```

Mint a short-lived token the kagent controller's identity can use:

```bash
TOKEN=$(kubectl create token kagent-controller -n kagent --audience=api.ate-system.svc --duration=15m)
```

List the actors. Note your actor's id and its status (it will be `SUSPENDED` between requests):

```bash
grpcurl -insecure -H "authorization: Bearer $TOKEN" -d '{}' \
  localhost:18443 ateapi.Control/ListActors
```

## Step 3: Resume the actor explicitly

Copy the actor id from the list above and resume it — watch it flip to `RUNNING`:

```bash
grpcurl -insecure -H "authorization: Bearer $TOKEN" \
  -d '{"actor_id":"<ACTOR_ID>"}' \
  localhost:18443 ateapi.Control/ResumeActor
```

Re-run `ListActors` to confirm the state change.

## Step 4: Save a marker

```bash
echo "export SUBSTRATE_LAB_DONE=true" >> ~/.bashrc
```

## ✅ What you've learned

- Each chat request restores an actor from a snapshot and suspends it again afterward.
- The actor's state lives in object storage between requests — the worker is freed.
- You can drive the actor lifecycle directly through the `ate-api` (`ListActors`, `ResumeActor`).
