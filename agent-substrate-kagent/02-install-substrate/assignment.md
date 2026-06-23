---
slug: install-substrate
id: p4qd8mtzk1rn
type: challenge
title: Install Agent Substrate
teaser: Install the Agent Substrate control plane into the ate-system namespace and map each running pod to its role.
notes:
- type: text
  contents: |
    # 🏗️ The substrate control plane

    Agent Substrate installs into the `ate-system` namespace. The key components:

    | Pod | Role |
    |-----|------|
    | `ate-api-server` | Control plane: actor lifecycle, scheduling, suspend/resume. Bypasses kube-scheduler. |
    | `ate-controller` | Reconciles `WorkerPool` + `ActorTemplate` CRDs into Deployments. |
    | `atelet` (DaemonSet) | Node supervisor: pulls images, manages sandbox lifecycle, talks to object storage. |
    | `atenet-router` | Envoy L7 router; resolves which worker serves each request. |
    | `valkey-cluster-{0..5}` | State store: actor/worker records and locks. |
    | `rustfs` | In-cluster S3-compatible object storage holding snapshot images. |

    The substrate v0.0.6 chart defaults to **JWT auth** (ServiceAccount tokens), so a
    stock kind cluster works — no feature gates, no custom kind config.
tabs:
- id: trm2bb02inst
  title: Terminal
  type: terminal
  hostname: server
- id: cod2bb02inst
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Install Agent Substrate

You'll install Agent Substrate (v0.0.6) from its published Helm charts into the `ate-system`
namespace.

## Step 1: Install the substrate CRDs

```bash
helm upgrade --install substrate-crds \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate-crds \
  --version 0.0.6 \
  --namespace ate-system --create-namespace --wait
```

## Step 2: Install the substrate control plane

```bash
helm upgrade --install substrate \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate \
  --version 0.0.6 \
  --namespace ate-system --wait --timeout 10m
```

This pulls several images and starts the valkey cluster, so it can take a few minutes.

## Step 3: Watch it come up

```bash
kubectl get pods -n ate-system
```

Wait until you see `ate-api-server`, `ate-controller`, `atelet-*`, `atenet-router`,
`valkey-cluster-0` through `-5`, and `rustfs` all `Running` (a few init Jobs will show
`Completed`).

## Step 4: Inspect the CRDs you just installed

```bash
kubectl get crd | grep ate.dev
kubectl get workerpools.ate.dev -A
```

There are no WorkerPools yet — kagent will create one in the next challenge.

## ✅ What you've learned

- Agent Substrate runs its control plane in `ate-system`.
- `ate-api-server` schedules actors onto workers; `atenet-router` routes requests; `atelet`
  drives the sandbox lifecycle; `valkey` stores state; `rustfs` stores snapshots.
- The `WorkerPool` and `ActorTemplate` CRDs are now available.
