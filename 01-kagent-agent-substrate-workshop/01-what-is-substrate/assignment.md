---
slug: what-is-substrate
id: migxm5gsxgie
type: challenge
title: What is an Agent Substrate?
teaser: Understand the substrate model — actors, workers, golden snapshots, and suspend/resume
  — then confirm your kind cluster is ready.
notes:
- type: text
  contents: "# \U0001F9E0 Agent Substrate\n\n**Agent Substrate** is a Kubernetes-native
    runtime for running huge numbers of\n*stateful* agent sessions on a *small* pool
    of pre-warmed pods.\n\n## The problem\n\nAgent workloads are bursty and idle most
    of the time. Plain Kubernetes handles\nthis badly:\n- idle pods still consume
    resources\n- the API server can't track millions of objects\n- pod cold-start
    takes seconds — too slow for millisecond tasks\n\n## The idea\n\nDecouple **actor**
    lifecycle from **pod** lifecycle:\n- An **Actor** is one logical agent session.\n-
    A **Worker** is a pre-warmed pod that can host one actor at a time.\n- Idle actors
    are **suspended**: gVisor checkpoints their full state (RAM +\n  filesystem) to
    object storage, and the worker is returned to the pool.\n- On the next request
    the actor is **resumed** sub-second into a free worker —\n  exactly where it left
    off.\n\nThink serverless scale-to-zero, but for *stateful* sessions.\n"
tabs:
- id: oipuhogxgcgd
  title: Terminal
  type: terminal
  hostname: server
- id: tm6bqcaklaum
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

## The model at a glance

Many logical **actors** are mapped onto a *small* pool of warm **workers**. An idle
actor isn't a pod — it's a snapshot sitting in object storage:

```text
        many actors                   few warm workers
     (one per session)             (host one actor at a time)

   Actor A  RUNNING   ───────────────▶  ┌──────────┐
   Actor B  RUNNING   ───────────────▶  │ Worker 1 │
                                        │ Worker 2 │
   Actor C  SUSPENDED  ┐                └──────────┘
   Actor D  SUSPENDED  ├─▶ ┌────────────────────────┐
   Actor E  SUSPENDED  ┘   │ object storage (rustfs) │
     ...                   │  one snapshot per actor │
                           └────────────────────────┘

   On the next request a SUSPENDED actor is restored into a
   free worker, sub-second, exactly where it left off.
```

## The actor lifecycle

```text
   ActorTemplate ──build──▶ golden snapshot (v0)
                                  │
                                  │ first request
                                  ▼
              resume          ┌─────────┐
        ┌──────────────────▶  │ RUNNING │  runs the LLM call
        │                     └────┬────┘
   ┌─────────┐                     │ idle / done
   │SUSPENDED│ ◀───────────────────┘
   └─────────┘   suspend: checkpoint full state (RAM + FS)
                 to object storage, worker → back to pool
```

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
