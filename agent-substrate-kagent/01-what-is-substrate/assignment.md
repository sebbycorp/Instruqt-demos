---
slug: what-is-substrate
id: m3vx9twha2qk
type: challenge
title: What is an Agent Substrate?
teaser: Understand the substrate model — actors, workers, golden snapshots, and suspend/resume — then confirm your kind cluster is ready.
notes:
- type: text
  contents: |
    # 🧠 Agent Substrate

    **Agent Substrate** is a Kubernetes-native runtime for running huge numbers of
    *stateful* agent sessions on a *small* pool of pre-warmed pods.

    ## The problem

    Agent workloads are bursty and idle most of the time. Plain Kubernetes handles
    this badly:
    - idle pods still consume resources
    - the API server can't track millions of objects
    - pod cold-start takes seconds — too slow for millisecond tasks

    ## The idea

    Decouple **actor** lifecycle from **pod** lifecycle:
    - An **Actor** is one logical agent session.
    - A **Worker** is a pre-warmed pod that can host one actor at a time.
    - Idle actors are **suspended**: gVisor checkpoints their full state (RAM +
      filesystem) to object storage, and the worker is returned to the pool.
    - On the next request the actor is **resumed** sub-second into a free worker —
      exactly where it left off.

    Think serverless scale-to-zero, but for *stateful* sessions.
tabs:
- id: trm1aa01what
  title: Terminal
  type: terminal
  hostname: server
- id: cod1aa01what
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# What is an Agent Substrate?

Agent Substrate runs many stateful agent **actors** on a few pre-warmed **worker** pods,
suspending and resuming them via gVisor checkpoint/restore. In this lab you'll deploy it on
a local `kind` cluster and run a real agent on top of it.

## Key terms

| Term | Meaning |
|------|---------|
| **Actor** | One logical agent session. State: `RUNNING` or `SUSPENDED`. |
| **Worker** | A pre-warmed pod hosting at most one actor. State: `IDLE` or `BUSY`. |
| **WorkerPool** | A set of warm standby worker pods (a CRD). |
| **ActorTemplate** | The immutable "class" an actor is created from (a CRD). Creating one builds a **golden snapshot** (version 0). |
| **Golden snapshot** | The initial frozen image every new actor is restored from. |
| **Suspend / Resume** | Checkpoint actor state to object storage / restore it sub-second. |

## Step 1: Confirm your cluster is ready

Your environment already created a `kind` cluster named `kagent-substrate`. Verify it:

```bash
kubectl get nodes
```

You should see one node in `Ready` state.

```bash
kubectl cluster-info
```

## Step 2: Confirm your tools

```bash
kind version
helm version --short
grpcurl --version
```

## ✅ What you've learned

- An Agent Substrate maps many **actors** onto few **worker** pods.
- Idle actors are **suspended** to object storage and **resumed** on demand via gVisor.
- The building blocks are `WorkerPool` and `ActorTemplate` CRDs, plus dynamic `Actor`/`Worker` records.

Next, you'll install Agent Substrate itself.
