# Agent Substrate + kagent Instruqt Lab — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a 6-challenge Instruqt track that teaches Agent Substrate, then has the learner deploy Agent Substrate + OSS kagent on a `kind` cluster and run a `SandboxAgent` whose state is checkpointed/restored by gVisor.

**Architecture:** A single GCP VM (`server`, n1-standard-8) runs a `kind` cluster. The track setup script installs tooling and creates the cluster. The learner runs the substrate and kagent Helm installs themselves inside the challenges (Approach A); each challenge ships a `solve-server` (exact commands behind Instruqt's "Solve" button) and a `check-server` (validation gate). Release-mode only — published OCI charts, JWT auth, no source builds.

**Tech Stack:** Instruqt (track.yml/config.yml/setup-server/check-server/solve-server), kind, kubectl, helm, grpcurl, Agent Substrate 0.0.6, kagent OSS 0.9.7, gVisor (`ateom-gvisor:v0.0.6`).

**Spec:** `docs/superpowers/specs/2026-06-23-agent-substrate-kagent-lab-design.md`

---

## Conventions & verification model

This is an Instruqt track, not an app — there is no unit-test runner. The verification model is:

- **`shellcheck`** every bash script (`setup-server`, `check-server`, `solve-server`).
- **Frontmatter sanity**: every `assignment.md` has valid YAML frontmatter with a unique `id`, a `slug`, `type: challenge`, and the standard `tabs`.
- **`check-server` logic IS the test** for each challenge — it is the gate the learner must pass. Each task reviews its own check logic.
- **Live VM run** is the final acceptance test and must happen in the Instruqt environment (flagged in the spec as the one thing that can't be verified locally — gVisor checkpoint/restore on kind-in-a-VM).

**Instruqt helper functions** available in scripts (provided by the platform, do not define): `set-workdir`, `fail-message`, `agent-...`. Use `set -euxo pipefail` in setup/solve, and `set -uxo pipefail` (no `-e`) in check-server so we can emit `fail-message` before exiting.

**IDs**: the `id` values below are pre-generated unique 12-char lowercase-alphanumeric strings (matching the format Instruqt uses, e.g. `vxjkcawnavuq`). If the team's Instruqt CLI reassigns ids/checksum on push (as in recent commits "Sync assigned ids/checksum…"), let it — these are valid placeholders that satisfy uniqueness.

**Track directory:** `agent-substrate-kagent/` at the repo root (sibling to `kagent-enterprise-workshop/`).

---

## File Structure

```
agent-substrate-kagent/
  track.yml                                   # track metadata
  config.yml                                  # VM + OPENAI_API_KEY secret
  track_scripts/setup-server                  # tools + kind cluster + key validation
  01-what-is-substrate/
    assignment.md  setup-server  check-server  solve-server
  02-install-substrate/
    assignment.md  setup-server  check-server  solve-server
  03-install-kagent/
    assignment.md  setup-server  check-server  solve-server
  04-deploy-sandboxagent/
    assignment.md  setup-server  check-server  solve-server
  05-chat-suspend-resume/
    assignment.md  setup-server  check-server  solve-server
  06-recap/
    assignment.md
```

Each file has one responsibility: `track.yml`/`config.yml` declare the lab; `track_scripts/setup-server` provisions shared infra; each challenge dir owns its teaching content (`assignment.md`) and its three lifecycle scripts.

---

## Task 1: Scaffold track skeleton (`track.yml`, `config.yml`, directories)

**Files:**
- Create: `agent-substrate-kagent/track.yml`
- Create: `agent-substrate-kagent/config.yml`

- [ ] **Step 1: Create the directory tree**

```bash
cd "$(git rev-parse --show-toplevel)"
mkdir -p agent-substrate-kagent/track_scripts \
         agent-substrate-kagent/01-what-is-substrate \
         agent-substrate-kagent/02-install-substrate \
         agent-substrate-kagent/03-install-kagent \
         agent-substrate-kagent/04-deploy-sandboxagent \
         agent-substrate-kagent/05-chat-suspend-resume \
         agent-substrate-kagent/06-recap
```

- [ ] **Step 2: Write `agent-substrate-kagent/track.yml`**

```yaml
slug: agent-substrate-kagent
id: qz7r3knwp2vh
title: 'Agent Substrate with kagent'
teaser: Learn what an Agent Substrate is, then deploy Agent Substrate + kagent on kind and run a SandboxAgent whose state is checkpointed by gVisor.
description: |
  Agent Substrate is a Kubernetes-native runtime that runs huge numbers of stateful
  agent "actors" on a small pool of pre-warmed worker pods. Idle actors are suspended
  by checkpointing their full state (RAM + filesystem, via gVisor) to object storage,
  then resumed sub-second on demand — serverless scale-to-zero, but for *stateful*
  agent sessions.

  In this hands-on lab, you'll:

  - 🧠 Learn the substrate model — actors, workers, golden snapshots, suspend/resume
  - 🏗️ Deploy Agent Substrate (v0.0.6) on a kind cluster
  - 🤖 Install OSS kagent (v0.9.7) wired to the substrate
  - 🚀 Deploy a SandboxAgent that runs inside a gVisor actor
  - 🔍 Watch an actor go Suspended → Running across a single request

  **Prerequisites:** Kubernetes basics (kubectl, Helm, pods, services). No prior
  kagent or substrate experience required.
icon: ""
tags:
- ai
- agents
- kubernetes
- kagent
- substrate
- gvisor
- solo-io
owner: soloio
developers:
- sebastian.maniak@solo.io
idle_timeout: 1800
timelimit: 5400
lab_config:
  sidebar_enabled: true
  feedback_recap_enabled: true
  feedback_tab_enabled: false
  loadingMessages: true
  hideStopButton: false
checksum: "0"
enhanced_loading: false
```

- [ ] **Step 3: Write `agent-substrate-kagent/config.yml`**

```yaml
version: "3"
virtualmachines:
- name: server
  image: solo-test-236622/workshop-instruqt-kgateway-20251023
  shell: /usr/bin/bash
  machine_type: n1-standard-8
secrets:
- name: OPENAI_API_KEY
```

> Note: the `image` is reused from `kagent-enterprise-workshop/config.yml`. During live VM
> testing (Task 9) confirm Docker is present and usable on this image; the setup script
> installs kind/kubectl/helm/grpcurl regardless, but kind requires a working Docker daemon.

- [ ] **Step 4: Commit**

```bash
git add agent-substrate-kagent/track.yml agent-substrate-kagent/config.yml
git commit -m "scaffold: agent-substrate-kagent track metadata and config"
```

---

## Task 2: Track setup script (tools + kind cluster + key validation)

**Files:**
- Create: `agent-substrate-kagent/track_scripts/setup-server`

- [ ] **Step 1: Write `agent-substrate-kagent/track_scripts/setup-server`**

```bash
#!/bin/bash

# Wait for instruqt bootstrap
until [ -f /opt/instruqt/bootstrap/host-bootstrap-completed ]; do
  sleep 1
done

set -euxo pipefail

set-workdir /root

# Pinned versions (kept in one place; also exported into the shell for the learner)
KIND_VERSION=v0.24.0
KIND_CLUSTER=kagent-substrate
SUBSTRATE_VERSION=0.0.6
KAGENT_VERSION=0.9.7

# Shell conveniences + version env for the learner's terminal
cat >> ~/.bashrc << EOF
alias k=kubectl
complete -F __start_kubectl k 2>/dev/null || true
source /etc/bash_completion 2>/dev/null || true
export KIND_CLUSTER=${KIND_CLUSTER}
export SUBSTRATE_VERSION=${SUBSTRATE_VERSION}
export KAGENT_VERSION=${KAGENT_VERSION}
EOF

# --- Install tooling if missing ---------------------------------------------
if ! command -v kubectl >/dev/null 2>&1; then
  curl -fsSLo /usr/local/bin/kubectl \
    "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x /usr/local/bin/kubectl
fi

if ! command -v helm >/dev/null 2>&1; then
  curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

if ! command -v kind >/dev/null 2>&1; then
  curl -fsSLo /usr/local/bin/kind \
    "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"
  chmod +x /usr/local/bin/kind
fi

if ! command -v grpcurl >/dev/null 2>&1; then
  curl -fsSL "https://github.com/fullstorydev/grpcurl/releases/download/v1.9.1/grpcurl_1.9.1_linux_x86_64.tar.gz" \
    | tar -xz -C /usr/local/bin grpcurl
  chmod +x /usr/local/bin/grpcurl
fi

# --- Ensure Docker is running (kind requires it) ----------------------------
systemctl start docker 2>/dev/null || true
timeout 60 bash -c 'until docker info >/dev/null 2>&1; do sleep 2; done'

# --- Create the kind cluster ------------------------------------------------
if ! kind get clusters 2>/dev/null | grep -q "^${KIND_CLUSTER}$"; then
  kind create cluster --name "${KIND_CLUSTER}" --wait 120s
fi

kubectl wait --for=condition=Ready nodes --all --timeout=120s

# --- Validate the OpenAI key (footgun: empty key installs silently) ---------
# Disable xtrace around the secret so the key value is never echoed into logs.
set +x
if [ -n "${OPENAI_API_KEY:-}" ]; then
  echo "export OPENAI_API_KEY=${OPENAI_API_KEY}" >> ~/.bashrc
  echo "=== OPENAI_API_KEY present (len=${#OPENAI_API_KEY}) ==="
else
  echo "=== WARNING: OPENAI_API_KEY is empty — kagent install in challenge 3 will fail ==="
fi
set -x

echo "=== Track setup complete: kind cluster '${KIND_CLUSTER}' ready ==="
```

- [ ] **Step 2: Lint the script**

Run: `shellcheck agent-substrate-kagent/track_scripts/setup-server`
Expected: no errors. (Instruqt helpers `set-workdir` are external; if shellcheck flags them as undefined commands that is acceptable — they are provided at runtime. Use `# shellcheck disable=SC2154` only if needed; do not silence real issues.)

- [ ] **Step 3: Commit**

```bash
git add agent-substrate-kagent/track_scripts/setup-server
git commit -m "feat: track setup installs tooling and creates kind cluster"
```

---

## Task 3: Challenge 1 — What is an Agent Substrate?

**Files:**
- Create: `agent-substrate-kagent/01-what-is-substrate/assignment.md`
- Create: `agent-substrate-kagent/01-what-is-substrate/setup-server`
- Create: `agent-substrate-kagent/01-what-is-substrate/check-server`
- Create: `agent-substrate-kagent/01-what-is-substrate/solve-server`

- [ ] **Step 1: Write `01-what-is-substrate/assignment.md`**

````markdown
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
````

- [ ] **Step 2: Write `01-what-is-substrate/setup-server`**

```bash
#!/bin/bash
set -euxo pipefail

# Cluster was created by the track setup-server; just confirm it is reachable.
kubectl wait --for=condition=Ready nodes --all --timeout=120s

echo "Challenge 1 setup complete"
```

- [ ] **Step 3: Write `01-what-is-substrate/check-server`**

```bash
#!/bin/bash
set -uxo pipefail

if ! kubectl get nodes >/dev/null 2>&1; then
  fail-message "kind cluster not reachable. The track setup may still be running — wait a moment and re-check."
  exit 1
fi

if ! kubectl wait --for=condition=Ready nodes --all --timeout=10s >/dev/null 2>&1; then
  fail-message "Cluster node is not Ready yet. Wait a few seconds and check again."
  exit 1
fi

for tool in kind helm grpcurl; do
  if ! command -v "$tool" >/dev/null 2>&1; then
    fail-message "Required tool '$tool' is not installed. The track setup may still be running."
    exit 1
  fi
done

echo "Challenge 1 complete!"
```

- [ ] **Step 4: Write `01-what-is-substrate/solve-server`**

```bash
#!/bin/bash
set -euxo pipefail

kubectl get nodes
kubectl cluster-info

echo "Challenge 1 solved"
```

- [ ] **Step 5: Lint all three scripts**

Run: `shellcheck agent-substrate-kagent/01-what-is-substrate/{setup,check,solve}-server`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add agent-substrate-kagent/01-what-is-substrate
git commit -m "feat: challenge 1 — what is an agent substrate"
```

---

## Task 4: Challenge 2 — Install Agent Substrate

**Files:**
- Create: `agent-substrate-kagent/02-install-substrate/assignment.md`
- Create: `agent-substrate-kagent/02-install-substrate/setup-server`
- Create: `agent-substrate-kagent/02-install-substrate/check-server`
- Create: `agent-substrate-kagent/02-install-substrate/solve-server`

- [ ] **Step 1: Write `02-install-substrate/assignment.md`**

````markdown
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
````

- [ ] **Step 2: Write `02-install-substrate/setup-server`**

```bash
#!/bin/bash
set -euxo pipefail

# Nothing to pre-deploy — the learner installs substrate in this challenge.
kubectl wait --for=condition=Ready nodes --all --timeout=120s

echo "Challenge 2 setup complete"
```

- [ ] **Step 3: Write `02-install-substrate/check-server`**

```bash
#!/bin/bash
set -uxo pipefail

if ! kubectl get namespace ate-system >/dev/null 2>&1; then
  fail-message "Namespace 'ate-system' not found. Run the helm install commands from the assignment."
  exit 1
fi

# Core control-plane workloads must exist and be progressing toward Ready.
for deploy in ate-api-server ate-controller atenet-router; do
  if ! kubectl get deploy -n ate-system 2>/dev/null | grep -q "${deploy}"; then
    fail-message "Substrate component '${deploy}' not found in ate-system. Did the 'substrate' helm install finish?"
    exit 1
  fi
done

if ! kubectl get crd workerpools.ate.dev actortemplates.ate.dev >/dev/null 2>&1; then
  fail-message "Substrate CRDs (workerpools/actortemplates) missing. Run the 'substrate-crds' helm install."
  exit 1
fi

# All ate-system deployments should be Available.
if ! kubectl wait --for=condition=Available deployment --all -n ate-system --timeout=120s >/dev/null 2>&1; then
  fail-message "Substrate components are not all Available yet. Run 'kubectl get pods -n ate-system' and wait for them to be Running."
  exit 1
fi

echo "Challenge 2 complete!"
```

- [ ] **Step 4: Write `02-install-substrate/solve-server`**

```bash
#!/bin/bash
set -euxo pipefail

helm upgrade --install substrate-crds \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate-crds \
  --version 0.0.6 \
  --namespace ate-system --create-namespace --wait

helm upgrade --install substrate \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate \
  --version 0.0.6 \
  --namespace ate-system --wait --timeout 10m

kubectl wait --for=condition=Available deployment --all -n ate-system --timeout=300s

echo "Challenge 2 solved"
```

- [ ] **Step 5: Lint all three scripts**

Run: `shellcheck agent-substrate-kagent/02-install-substrate/{setup,check,solve}-server`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add agent-substrate-kagent/02-install-substrate
git commit -m "feat: challenge 2 — install agent substrate"
```

---

## Task 5: Challenge 3 — Install kagent, wired to substrate

**Files:**
- Create: `agent-substrate-kagent/03-install-kagent/assignment.md`
- Create: `agent-substrate-kagent/03-install-kagent/setup-server`
- Create: `agent-substrate-kagent/03-install-kagent/check-server`
- Create: `agent-substrate-kagent/03-install-kagent/solve-server`

- [ ] **Step 1: Write `03-install-kagent/assignment.md`**

````markdown
---
slug: install-kagent
id: r5te9nua2wjm
type: challenge
title: Install kagent, wired to substrate
teaser: Install OSS kagent v0.9.7 with the substrate integration enabled, and create the default WorkerPool.
notes:
- type: text
  contents: |
    # 🤖 kagent on substrate

    kagent is the agent runtime/control plane. Substrate is the execution layer
    *underneath* it. kagent v0.9.7 is the first release with substrate support
    (PR #1981).

    The substrate-specific install flags:

    - `controller.substrate.enabled=true` — turn on the integration
    - `controller.substrate.ateApiEndpoint` — where the substrate control plane lives
    - `controller.substrate.atenetRouterURL` — the substrate request router
    - `controller.substrate.defaultWorkerPool.*` — the pool agents land on by default
    - `substrateWorkerPool.create=true` + `.replicas=1` — create one warm worker

    ⚠️ The kagent controller **hard-fails (crash-loops)** if the substrate API isn't
    reachable at startup — which is why substrate had to be installed first.
tabs:
- id: trm3cc03kgnt
  title: Terminal
  type: terminal
  hostname: server
- id: cod3cc03kgnt
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Install kagent, wired to substrate

## Step 1: Confirm your OpenAI key is set

```bash
[[ -n "${OPENAI_API_KEY:-}" ]] && echo "key set (len=${#OPENAI_API_KEY})" || echo "OPENAI_API_KEY is empty!"
```

> ⚠️ Don't inline the key assignment on the helm line — `OPENAI_API_KEY=... helm ... --set ...="${OPENAI_API_KEY}"`
> evaluates the variable *before* the assignment runs and passes an empty string. The key is
> already exported in your shell by the lab setup.

## Step 2: Install the kagent CRDs

```bash
helm upgrade --install kagent-crds \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --version 0.9.7 \
  --namespace kagent --create-namespace --wait
```

## Step 3: Install kagent with substrate enabled

```bash
helm upgrade --install kagent \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --version 0.9.7 --namespace kagent --timeout 10m --wait \
  --set providers.default=openAI \
  --set providers.openAI.apiKey="${OPENAI_API_KEY}" \
  --set controller.substrate.enabled=true \
  --set controller.substrate.ateApiEndpoint="dns:///api.ate-system.svc:443" \
  --set controller.substrate.ateApiInsecure=true \
  --set controller.substrate.atenetRouterURL="http://atenet-router.ate-system.svc:80" \
  --set controller.substrate.ateApiTokenFile="/var/run/secrets/tokens/ate-api/token" \
  --set controller.substrate.defaultWorkerPool.namespace=kagent \
  --set controller.substrate.defaultWorkerPool.name=kagent-default \
  --set substrateWorkerPool.create=true \
  --set substrateWorkerPool.name=kagent-default \
  --set substrateWorkerPool.replicas=1 \
  --set substrateWorkerPool.ateomImage=ghcr.io/kagent-dev/substrate/ateom-gvisor:v0.0.6
```

> If helm times out while the controller waits on its database (it restarts a couple of
> times during cold start), wait for it manually and continue:
>
> ```bash
> kubectl wait deploy/kagent-controller -n kagent --for=condition=Available --timeout=10m
> ```

## Step 4: Verify the WorkerPool and the integration

```bash
kubectl get workerpools.ate.dev -A
```

You should see `kagent/kagent-default`.

```bash
kubectl run substrate-status-check -n kagent --rm -i --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  http://kagent-controller:8083/api/substrate/status
```

The response should include `"enabled": true`.

## ✅ What you've learned

- kagent v0.9.7 enables substrate via `controller.substrate.*` flags.
- `defaultWorkerPool` lets agents resolve a pool without naming it explicitly.
- The controller exposes a substrate status API confirming the integration is live.
````

- [ ] **Step 2: Write `03-install-kagent/setup-server`**

```bash
#!/bin/bash
set -euxo pipefail

# The learner installs kagent here; ensure substrate is healthy first so the
# controller doesn't crash-loop on a missing ate-api.
kubectl wait --for=condition=Available deployment --all -n ate-system --timeout=300s || true

echo "Challenge 3 setup complete"
```

- [ ] **Step 3: Write `03-install-kagent/check-server`**

```bash
#!/bin/bash
set -uxo pipefail

if ! kubectl get deploy -n kagent kagent-controller >/dev/null 2>&1; then
  fail-message "kagent-controller not found. Run the kagent helm install from the assignment."
  exit 1
fi

if ! kubectl wait deploy/kagent-controller -n kagent --for=condition=Available --timeout=300s >/dev/null 2>&1; then
  fail-message "kagent-controller is not Available yet (it can restart a couple of times during cold start). Wait and re-check."
  exit 1
fi

if ! kubectl get workerpool kagent-default -n kagent >/dev/null 2>&1; then
  fail-message "WorkerPool 'kagent/kagent-default' not found. Ensure substrateWorkerPool.create=true was set on the helm install."
  exit 1
fi

if ! kubectl get secret kagent-openai -n kagent >/dev/null 2>&1; then
  fail-message "Secret 'kagent-openai' missing — the OpenAI key was empty at install time. Re-run the kagent helm install with the key set."
  exit 1
fi

echo "Challenge 3 complete!"
```

- [ ] **Step 4: Write `03-install-kagent/solve-server`**

```bash
#!/bin/bash
set -euxo pipefail

helm upgrade --install kagent-crds \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --version 0.9.7 \
  --namespace kagent --create-namespace --wait

# Disable xtrace for the kagent install so the OpenAI key isn't echoed into logs.
set +x
helm upgrade --install kagent \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --version 0.9.7 --namespace kagent --timeout 10m --wait \
  --set providers.default=openAI \
  --set providers.openAI.apiKey="${OPENAI_API_KEY}" \
  --set controller.substrate.enabled=true \
  --set controller.substrate.ateApiEndpoint="dns:///api.ate-system.svc:443" \
  --set controller.substrate.ateApiInsecure=true \
  --set controller.substrate.atenetRouterURL="http://atenet-router.ate-system.svc:80" \
  --set controller.substrate.ateApiTokenFile="/var/run/secrets/tokens/ate-api/token" \
  --set controller.substrate.defaultWorkerPool.namespace=kagent \
  --set controller.substrate.defaultWorkerPool.name=kagent-default \
  --set substrateWorkerPool.create=true \
  --set substrateWorkerPool.name=kagent-default \
  --set substrateWorkerPool.replicas=1 \
  --set substrateWorkerPool.ateomImage=ghcr.io/kagent-dev/substrate/ateom-gvisor:v0.0.6 \
  || true
set -x

kubectl wait deploy/kagent-controller -n kagent --for=condition=Available --timeout=600s

echo "Challenge 3 solved"
```

- [ ] **Step 5: Lint all three scripts**

Run: `shellcheck agent-substrate-kagent/03-install-kagent/{setup,check,solve}-server`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add agent-substrate-kagent/03-install-kagent
git commit -m "feat: challenge 3 — install kagent wired to substrate"
```

---

## Task 6: Challenge 4 — Deploy a SandboxAgent

**Files:**
- Create: `agent-substrate-kagent/04-deploy-sandboxagent/assignment.md`
- Create: `agent-substrate-kagent/04-deploy-sandboxagent/setup-server`
- Create: `agent-substrate-kagent/04-deploy-sandboxagent/check-server`
- Create: `agent-substrate-kagent/04-deploy-sandboxagent/solve-server`

- [ ] **Step 1: Write `04-deploy-sandboxagent/assignment.md`**

````markdown
---
slug: deploy-sandboxagent
id: s6uf0pvb3xko
type: challenge
title: Deploy a SandboxAgent
teaser: Deploy a declarative SandboxAgent that runs inside a gVisor actor on the substrate, and watch its golden snapshot build.
notes:
- type: text
  contents: |
    # 🚀 SandboxAgent

    kagent's `SandboxAgent` (`platform: substrate`) is a per-session declarative agent
    that runs as a substrate **actor**.

    When you create one, kagent generates an `ActorTemplate`, which triggers the
    **golden snapshot** — the version-0 frozen image every session is restored from.
    The first snapshot takes ~60–90s.

    > The runtime must be **Go** — Python ADK isn't compatible with gVisor checkpointing.

    The manifest is pre-written to `/root/hello-substrate.yaml` (open it in the Code
    Editor tab).
tabs:
- id: trm4dd04sbox
  title: Terminal
  type: terminal
  hostname: server
- id: cod4dd04sbox
  title: Code Editor
  type: code
  hostname: server
  path: /root/hello-substrate.yaml
- id: svc4dd04ui00
  title: kagent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# Deploy a SandboxAgent

## Step 1: Review the manifest

Open `/root/hello-substrate.yaml` in the Code Editor tab:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: SandboxAgent
metadata:
  name: hello-substrate
  namespace: kagent
spec:
  type: Declarative
  description: Tiny declarative agent running inside a substrate actor
  declarative:
    runtime: go
    modelConfig: default-model-config
    systemMessage: |
      You are a friendly assistant living inside an Agent Substrate sandbox.
      When asked who you are, say "I am hello-substrate, a Go ADK declarative
      agent running inside a gVisor actor."
  platform: substrate
  substrate:
    workerPoolRef:
      name: kagent-default
```

## Step 2: Apply it

```bash
kubectl apply -f /root/hello-substrate.yaml
```

## Step 3: Wait for it to be Ready

```bash
kubectl wait sandboxagent/hello-substrate -n kagent --for=condition=Ready --timeout=5m
```

The first golden snapshot takes ~60–90s.

## Step 4: Inspect the generated substrate resources

```bash
kubectl get sandboxagent -n kagent
kubectl get actortemplates.ate.dev -A
```

kagent generated an `ActorTemplate` for your agent (owned by the SandboxAgent).

## ✅ What you've learned

- A `SandboxAgent` with `platform: substrate` runs as a substrate actor.
- Creating it generates an `ActorTemplate` and builds a golden snapshot.
- Declarative substrate agents must use the Go runtime.
````

- [ ] **Step 2: Write `04-deploy-sandboxagent/setup-server`** (writes the manifest + starts the UI port-forward)

```bash
#!/bin/bash
set -euxo pipefail

# Ensure kagent is up before the learner deploys an agent onto it.
kubectl wait deploy/kagent-controller -n kagent --for=condition=Available --timeout=300s || true

# Pre-write the manifest so it shows in the Code Editor tab.
cat > /root/hello-substrate.yaml <<'YAML'
apiVersion: kagent.dev/v1alpha2
kind: SandboxAgent
metadata:
  name: hello-substrate
  namespace: kagent
spec:
  type: Declarative
  description: Tiny declarative agent running inside a substrate actor
  declarative:
    runtime: go
    modelConfig: default-model-config
    systemMessage: |
      You are a friendly assistant living inside an Agent Substrate sandbox.
      When asked who you are, say "I am hello-substrate, a Go ADK declarative
      agent running inside a gVisor actor."
  platform: substrate
  substrate:
    workerPoolRef:
      name: kagent-default
YAML

# Expose the kagent UI on host port 8080 for the service tab.
cat > /etc/systemd/system/kagent-ui.service <<'SVCEOF'
[Unit]
Description=kagent UI Port Forward
After=cloud-final.service multi-user.target
[Service]
Restart=always
RestartSec=5s
Environment="KUBECONFIG=/root/.kube/config"
ExecStart=/usr/local/bin/kubectl -n kagent port-forward svc/kagent-ui 8080:8080 --address 0.0.0.0
[Install]
WantedBy=multi-user.target
SVCEOF

systemctl daemon-reload
systemctl enable kagent-ui
systemctl restart kagent-ui

echo "Challenge 4 setup complete"
```

- [ ] **Step 3: Write `04-deploy-sandboxagent/check-server`**

```bash
#!/bin/bash
set -uxo pipefail

if ! kubectl get sandboxagent hello-substrate -n kagent >/dev/null 2>&1; then
  fail-message "SandboxAgent 'hello-substrate' not found. Run: kubectl apply -f /root/hello-substrate.yaml"
  exit 1
fi

if ! kubectl wait sandboxagent/hello-substrate -n kagent --for=condition=Ready --timeout=300s >/dev/null 2>&1; then
  fail-message "SandboxAgent is not Ready yet. The first golden snapshot takes ~60-90s — wait and re-check. Inspect with: kubectl describe sandboxagent hello-substrate -n kagent"
  exit 1
fi

echo "Challenge 4 complete!"
```

- [ ] **Step 4: Write `04-deploy-sandboxagent/solve-server`**

```bash
#!/bin/bash
set -euxo pipefail

kubectl apply -f /root/hello-substrate.yaml
kubectl wait sandboxagent/hello-substrate -n kagent --for=condition=Ready --timeout=300s

echo "Challenge 4 solved"
```

- [ ] **Step 5: Lint all three scripts**

Run: `shellcheck agent-substrate-kagent/04-deploy-sandboxagent/{setup,check,solve}-server`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add agent-substrate-kagent/04-deploy-sandboxagent
git commit -m "feat: challenge 4 — deploy a SandboxAgent on substrate"
```

---

## Task 7: Challenge 5 — Chat & watch suspend/resume

**Files:**
- Create: `agent-substrate-kagent/05-chat-suspend-resume/assignment.md`
- Create: `agent-substrate-kagent/05-chat-suspend-resume/setup-server`
- Create: `agent-substrate-kagent/05-chat-suspend-resume/check-server`
- Create: `agent-substrate-kagent/05-chat-suspend-resume/solve-server`

- [ ] **Step 1: Write `05-chat-suspend-resume/assignment.md`**

````markdown
---
slug: chat-suspend-resume
id: t7vg1qwc4ylp
type: challenge
title: Chat and watch suspend/resume
teaser: Chat with your agent in the kagent UI, then drive the actor lifecycle from the CLI to watch it go Suspended → Running.
notes:
- type: text
  contents: |
    # 🔍 The payoff

    When you chat with `hello-substrate`, a per-session gVisor **actor** is restored
    from the golden snapshot, runs the LLM call, and snapshots itself back to object
    storage — returning the worker to the pool.

    Between requests the actor sits **SUSPENDED**. On the next request it's **RESUMED**
    sub-second. You'll see this both in the UI's `/substrate` page and directly from the
    `ate-api` via `grpcurl`.
tabs:
- id: trm5ee05chat
  title: Terminal
  type: terminal
  hostname: server
- id: svc5ee05ui00
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
````

- [ ] **Step 2: Write `05-chat-suspend-resume/setup-server`**

```bash
#!/bin/bash
set -euxo pipefail

# Make sure the agent is ready to chat with.
kubectl wait sandboxagent/hello-substrate -n kagent --for=condition=Ready --timeout=300s || true

# Ensure the UI port-forward service is running (created in challenge 4).
systemctl restart kagent-ui 2>/dev/null || true

echo "Challenge 5 setup complete"
```

- [ ] **Step 3: Write `05-chat-suspend-resume/check-server`**

```bash
#!/bin/bash
set -uxo pipefail

if ! kubectl get sandboxagent hello-substrate -n kagent >/dev/null 2>&1; then
  fail-message "SandboxAgent 'hello-substrate' not found — complete challenge 4 first."
  exit 1
fi

if ! grpcurl --version >/dev/null 2>&1; then
  fail-message "grpcurl is not installed. The track setup may not have completed."
  exit 1
fi

if ! grep -q "SUBSTRATE_LAB_DONE" /root/.bashrc; then
  fail-message "Marker not saved. Run: echo \"export SUBSTRATE_LAB_DONE=true\" >> ~/.bashrc"
  exit 1
fi

echo "Challenge 5 complete!"
```

- [ ] **Step 4: Write `05-chat-suspend-resume/solve-server`**

```bash
#!/bin/bash
set -euxo pipefail

# Demonstrate the actor lifecycle non-interactively.
kubectl port-forward -n ate-system svc/api 18443:443 >/tmp/pf-api.log 2>&1 &
PF_PID=$!
sleep 3

TOKEN=$(kubectl create token kagent-controller -n kagent --audience=api.ate-system.svc --duration=15m)
grpcurl -insecure -H "authorization: Bearer $TOKEN" -d '{}' \
  localhost:18443 ateapi.Control/ListActors || true

kill "$PF_PID" 2>/dev/null || true

echo "export SUBSTRATE_LAB_DONE=true" >> ~/.bashrc

echo "Challenge 5 solved"
```

- [ ] **Step 5: Lint all three scripts**

Run: `shellcheck agent-substrate-kagent/05-chat-suspend-resume/{setup,check,solve}-server`
Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add agent-substrate-kagent/05-chat-suspend-resume
git commit -m "feat: challenge 5 — chat and watch suspend/resume"
```

---

## Task 8: Challenge 6 — Recap & what's next

**Files:**
- Create: `agent-substrate-kagent/06-recap/assignment.md`

- [ ] **Step 1: Write `06-recap/assignment.md`**

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add agent-substrate-kagent/06-recap
git commit -m "feat: challenge 6 — recap and what's next"
```

---

## Task 9: Track README + final verification

**Files:**
- Create: `agent-substrate-kagent/README.md`
- Modify: `README.md` (repo root — add the new track to the index, matching existing entries)

- [ ] **Step 1: Inspect the repo-root README to match its track-index format**

Run: `sed -n '1,80p' README.md`
Expected: shows how existing tracks are listed. Add an `agent-substrate-kagent` entry in the same style/section as the other tracks (title, one-line description, challenge count). Keep formatting identical to neighbors.

- [ ] **Step 2: Write `agent-substrate-kagent/README.md`**

```markdown
# Agent Substrate with kagent

A 6-challenge Instruqt track teaching Agent Substrate, then deploying Agent Substrate +
OSS kagent on a `kind` cluster and running a `SandboxAgent` inside a gVisor actor.

## Challenges

1. **What is an Agent Substrate?** — the model: actors, workers, golden snapshots, suspend/resume.
2. **Install Agent Substrate** — substrate 0.0.6 into `ate-system`.
3. **Install kagent, wired to substrate** — kagent 0.9.7 with substrate integration + default WorkerPool.
4. **Deploy a SandboxAgent** — a declarative Go agent running as a substrate actor.
5. **Chat & watch suspend/resume** — UI chat + `grpcurl` actor lifecycle.
6. **Recap & what's next** — scaling, `AgentHarness`, cleanup.

## Infra

- Single GCP VM (`server`, n1-standard-8) running a `kind` cluster.
- Secret: `OPENAI_API_KEY` (Instruqt-provided).
- Versions: substrate `0.0.6`, kagent OSS `0.9.7`, `ateom-gvisor:v0.0.6`.

## Design & plan

- Spec: `../docs/superpowers/specs/2026-06-23-agent-substrate-kagent-lab-design.md`
- Plan: `../docs/superpowers/plans/2026-06-23-agent-substrate-kagent-lab.md`

> Agent Substrate is pre-1.0; gVisor checkpoint/restore on kind requires a live VM test.
```

- [ ] **Step 3: Lint every script in the track at once**

Run: `find agent-substrate-kagent -name '*-server' -exec shellcheck {} +`
Expected: no errors across all setup/check/solve scripts.

- [ ] **Step 4: Validate all assignment.md frontmatter parses and ids are unique**

Run:
```bash
for f in agent-substrate-kagent/*/assignment.md; do
  awk 'NR==1&&/^---/{f=1;next} f&&/^---/{exit} f&&/^id:/{print FILENAME": "$0}' "$f"
done | sort -t: -k3 | awk '{print $NF}' | sort | uniq -d
```
Expected: empty output (no duplicate ids). Each file should print exactly one `id:` line above.

- [ ] **Step 5: Commit**

```bash
git add agent-substrate-kagent/README.md README.md
git commit -m "docs: add agent-substrate-kagent track README and repo index entry"
```

- [ ] **Step 6: Live VM acceptance test (Instruqt environment — REQUIRED before merge)**

This is the only step that cannot be verified locally (per spec §8.1: gVisor checkpoint/restore on kind-in-a-VM is unverified by research). Push the track to a test Instruqt environment and confirm, in order:

1. Track setup completes; `kubectl get nodes` shows `Ready`; `kind`/`helm`/`grpcurl` present.
2. Challenge 2 substrate install: all `ate-system` deployments become `Available`; `valkey-cluster-0..5` and `rustfs` `Running`.
3. Challenge 3 kagent install: `kagent-controller` `Available`, WorkerPool `kagent/kagent-default` exists, `/api/substrate/status` → `"enabled": true`.
4. Challenge 4: `SandboxAgent/hello-substrate` reaches `Ready` (golden snapshot builds).
5. Challenge 5: UI chat returns the expected identity sentence; `grpcurl ListActors` shows the actor; `ResumeActor` flips it to `RUNNING`. **If gVisor checkpoint/restore fails here**, capture the error, confirm challenges 1–4 still pass, and decide (per spec) whether to soften challenge 5 to CLI-only inspection.
6. Each challenge's **Solve** button drives the challenge to a passing **Check**.

Record results; fix any script/version drift found, then re-run.

---

## Self-Review (completed by plan author)

**Spec coverage:** Goal/audience → Tasks 1,3 + recap. Concept teaching (§2) → Task 3 + notes throughout. Release-mode deploy + OSS kagent + pinned versions (§3) → Tasks 4,5. Approach A, learner-runs-installs + solve/check (§3) → Tasks 4–7. Infra n1-standard-8 + OPENAI secret + tools incl. grpcurl (§4) → Tasks 1,2. 6-challenge arc (§5) → Tasks 3–8. Repo layout (§6) → Task 1 + per-challenge tasks. Fuller kagent flags + status API + grpcurl lifecycle (§7, Levan) → Tasks 5,7. Risks/mitigations (§8): empty-key footgun → Task 2 validation + Task 5 check; helm cold-start race → Task 5 wait fallback; cert/24h + scaling + AgentHarness out-of-scope → Task 8 recap; gVisor unknown → Task 9 Step 6. All sections covered.

**Placeholder scan:** No TBD/TODO; every script and manifest is shown in full; check/solve logic is concrete.

**Type/name consistency:** Cluster `kagent-substrate`, namespaces `ate-system`/`kagent`, WorkerPool `kagent-default`, agent `hello-substrate`, manifest path `/root/hello-substrate.yaml`, UI svc `kagent-ui` port 8080, ate-api svc `api` port 443→18443, controller status `kagent-controller:8083/api/substrate/status`, marker `SUBSTRATE_LAB_DONE` — all consistent across tasks. Tab `id`s are unique per challenge.
```
