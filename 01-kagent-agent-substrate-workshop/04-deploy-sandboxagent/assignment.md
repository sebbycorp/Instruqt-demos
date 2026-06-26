---
slug: deploy-sandboxagent
id: a05yn8u6sbdr
type: challenge
title: Deploy a SandboxAgent
teaser: Deploy a declarative SandboxAgent that runs inside a gVisor actor on the substrate,
  and watch its golden snapshot build.
notes:
- type: text
  contents: "# \U0001F680 SandboxAgent\n\nkagent's `SandboxAgent` (`platform: substrate`)
    is a per-session declarative agent\nthat runs as a substrate **actor**.\n\nThis
    one is a real **Kubernetes agent**: it calls the `k8s_*` tools on kagent's\nbuilt-in
    `kagent-tool-server` (a `RemoteMCPServer`) over MCP — so a single agent\nrunning
    inside a gVisor actor can actually inspect your cluster.\n\nWhen you create one,
    kagent generates an `ActorTemplate`, which triggers the\n**golden snapshot** —
    the version-0 frozen image every session is restored from.\nThe first snapshot
    takes ~60–90s.\n\n> The runtime must be **Go** — Python ADK isn't compatible with
    gVisor checkpointing.\n> The MCP tool wiring is identical for Go and Python agents.\n\nThe
    manifest is pre-written to `/root/hello-substrate.yaml` (open it in the Code\nEditor
    tab).\n"
tabs:
- id: pmy2ofmq4zy9
  title: Terminal
  type: terminal
  hostname: server
- id: n0pjxl727ydy
  title: Code Editor
  type: code
  hostname: server
  path: /root/hello-substrate.yaml
- id: yb3qixaevgdk
  title: kagent UI
  type: service
  hostname: server
  port: 8080
difficulty: ""
enhanced_loading: null
---

# Deploy a SandboxAgent

## Step 1: Confirm the MCP tool server is available

kagent ships a built-in `kagent-tool-server` (a `RemoteMCPServer`) that exposes the
Kubernetes (`k8s_*`) tools. Your agent will call it over MCP:

```bash,run
kubectl get remotemcpservers.kagent.dev -n kagent
```

You should see `kagent-tool-server`.

## Step 2: Review the manifest

Open `/root/hello-substrate.yaml` in the Code Editor tab. Note the `tools` block — it points
the agent at the `kagent-tool-server` MCP server and selects a few read-only `k8s_*` tools:

```yaml
apiVersion: kagent.dev/v1alpha2
kind: SandboxAgent
metadata:
  name: hello-substrate
  namespace: kagent
spec:
  type: Declarative
  description: A Kubernetes assistant running inside a substrate gVisor actor
  declarative:
    runtime: go
    modelConfig: default-model-config
    systemMessage: |
      You are a helpful Kubernetes assistant running inside an Agent Substrate
      gVisor actor. Use the Kubernetes tools to answer questions about the
      cluster. When asked who you are, say "I am a Kubernetes agent running
      inside a gVisor actor on Agent Substrate." Keep answers concise.
    tools:
    - type: McpServer
      mcpServer:
        name: kagent-tool-server
        kind: RemoteMCPServer
        apiGroup: kagent.dev
        toolNames:
        - k8s_get_resources
        - k8s_describe_resource
        - k8s_get_pod_logs
        - k8s_get_events
  platform: substrate
  substrate:
    workerPoolRef:
      name: kagent-default
```

## Step 3: Apply it

```bash,run
kubectl apply -f /root/hello-substrate.yaml
```

## Step 4: Wait for it to be Ready

```bash,run
kubectl wait sandboxagent/hello-substrate -n kagent --for=condition=Ready --timeout=5m
```

The first golden snapshot takes ~60–90s.

## Step 5: Inspect the generated substrate resources

```bash,run
kubectl get sandboxagent -n kagent
kubectl get actortemplates.ate.dev -A
```

kagent generated an `ActorTemplate` for your agent (owned by the SandboxAgent).

## ✅ What you've learned

- A `SandboxAgent` with `platform: substrate` runs as a substrate actor.
- A declarative agent attaches MCP tools via `declarative.tools` → `mcpServer`, referencing a
  `RemoteMCPServer` and selecting `toolNames` — here the built-in `kagent-tool-server` `k8s_*` tools.
- Creating it generates an `ActorTemplate` and builds a golden snapshot.
- Declarative substrate agents must use the Go runtime.
