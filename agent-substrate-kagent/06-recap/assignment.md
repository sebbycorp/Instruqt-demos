---
slug: recap
id: u8wh2rxd5zmq
type: challenge
title: Recap and what's next
teaser: Review what you built and where Agent Substrate goes from here.
notes:
- type: text
  contents: |
    # 🎓 Nicely done

    You deployed Agent Substrate and kagent on kind, ran a SandboxAgent inside a
    gVisor actor, and watched it suspend and resume.
tabs:
- id: trm6ff06recp
  title: Terminal
  type: terminal
  hostname: server
difficulty: ""
enhanced_loading: null
---

# Recap and what's next

## What you built

- **Agent Substrate** (`ate-system`) — control plane (`ate-api-server`), router
  (`atenet-router`), node supervisor (`atelet`), state (`valkey`), snapshots (`rustfs`).
- **kagent** (`kagent`) — wired to substrate with a default `WorkerPool`.
- A **`SandboxAgent`** running as a per-session gVisor actor, suspended to object storage
  between requests and resumed sub-second on demand.

## Scaling the WorkerPool

One worker serves many declarative sessions sequentially because each session releases its
slot the moment it snapshots back. To run overlapping sessions or long-lived agents, scale
the pool:

```bash
kubectl scale workerpool kagent-default -n kagent --replicas=3
```

## What's next (beyond this lab)

- **`AgentHarness` (`runtime: substrate`)** — long-lived runtimes like OpenClaw run as
  substrate actors reached through a kagent gateway. These need an object-storage bucket
  (e.g. `gs://...`) and don't auto-suspend, so each pins a worker slot.
- **Identity** — substrate can mint per-actor JWTs and certs for mTLS.
- **Observability** — substrate exposes metrics for activation latency and worker pool use.

## A note on this environment

Agent Substrate is **very early / pre-1.0** — APIs will change and it's not production-ready.
Substrate's in-cluster TLS certs also expire after ~24h on idle clusters, so this lab is best
run in a single sitting.

## Cleanup (if running outside Instruqt)

```bash
kind delete cluster --name kagent-substrate
```

## ✅ You've learned

- What an Agent Substrate is and the problem it solves.
- How to deploy substrate + kagent on kind from published charts.
- How a SandboxAgent runs as a gVisor actor and how suspend/resume works.
