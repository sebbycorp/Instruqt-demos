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
