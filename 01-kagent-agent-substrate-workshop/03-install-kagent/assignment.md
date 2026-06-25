---
slug: install-kagent
id: x6oajuycfl7g
type: challenge
title: Install kagent, wired to substrate
teaser: Install OSS kagent v0.9.7 with the substrate integration enabled, and create
  the default WorkerPool.
notes:
- type: text
  contents: "# \U0001F916 kagent on substrate\n\nkagent is the agent runtime/control
    plane. Substrate is the execution layer\n*underneath* it. kagent v0.9.7 is the
    first release with substrate support\n(PR #1981).\n\nThe substrate-specific install
    flags:\n\n- `controller.substrate.enabled=true` — turn on the integration\n- `controller.substrate.ateApiEndpoint`
    — where the substrate control plane lives\n- `controller.substrate.atenetRouterURL`
    — the substrate request router\n- `controller.substrate.defaultWorkerPool.*` —
    the pool agents land on by default\n- `substrateWorkerPool.create=true` + `.replicas=1`
    — create one warm worker\n\n⚠️ The kagent controller **hard-fails (crash-loops)**
    if the substrate API isn't\nreachable at startup — which is why substrate had
    to be installed first.\n"
tabs:
- id: n76nycbc1pjq
  title: Terminal
  type: terminal
  hostname: server
- id: 3vwfebpmrx0v
  title: Code Editor
  type: code
  hostname: server
  path: /root
difficulty: ""
enhanced_loading: null
---

# Install kagent, wired to substrate

## How kagent sits on top of substrate

kagent is the agent control plane; substrate is the execution layer *underneath* it.
The `controller.substrate.*` flags below are the wiring between the two:

```text
   kubectl / kagent UI
         │  apply SandboxAgent
         ▼
  ┌───────────────────────────────┐
  │ kagent  (namespace: kagent)   │
  │   controller ─ reconciles ─▶  ActorTemplate + WorkerPool
  │   ui                          │      (*.ate.dev CRDs)
  └───────────────┬───────────────┘
                  │ controller.substrate.*
                  │   ateApiEndpoint   → api.ate-system
                  │   atenetRouterURL  → atenet-router
                  ▼
  ┌───────────────────────────────┐
  │ Agent Substrate (ate-system)  │
  │   ate-api-server · router · atelet · valkey · rustfs
  └───────────────────────────────┘
```

> This is why order matters: the kagent controller crash-loops if the substrate API
> isn't reachable at startup, so substrate had to be installed first (challenge 2).

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
