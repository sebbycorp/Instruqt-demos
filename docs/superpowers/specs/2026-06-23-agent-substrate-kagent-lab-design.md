# Design: "Agent Substrates with kagent" — Instruqt Lab

**Date:** 2026-06-23
**Branch:** `substrate`
**Status:** Approved (design phase)

## 1. Goal & audience

Teach a Kubernetes-literate learner **what an Agent Substrate is and why it exists**, then
have them **deploy kagent + Agent Substrate on a `kind` cluster** and run a real
`SandboxAgent` whose state is checkpointed/restored by gVisor.

The payoff ("aha") moment: watching a per-session actor transition `Suspended` → `Running`
across a single chat request, proving that stateful agent sessions are being frozen to and
thawed from object storage on demand.

**Audience:** familiar with Kubernetes basics (kubectl, Helm, pods, services). No prior
kagent or substrate experience required.

## 2. What Agent Substrate is (the concept being taught)

Agent Substrate (`github.com/kagent-dev/substrate`, also `agent-substrate/substrate`) is a
Kubernetes-native runtime for highly-multiplexed, stateful "actor" workloads. It is **not**
an agent SDK — it is the execution substrate *underneath* an agent runtime like kagent.

**Problem it solves:** agent workloads are bursty and idle most of the time, but plain
Kubernetes wastes resources on this pattern (idle pods still consume resources; the API
server can't handle millions of resources; pod cold-start is seconds; high-churn volumes
overwhelm persistent storage).

**Core idea:** decouple actor lifecycle from pod lifecycle. Map a large set of logical
**actors** onto a small pool of pre-warmed **worker** pods. Instead of destroying idle
actors, *suspend* them via gVisor (`runsc`) checkpoint/restore, snapshot their full state
(RAM + writable filesystem) to object storage, and *resume* them sub-second on demand. This
yields heavy oversubscription (demo: ~250 sessions on 8 pods).

**Mental model for the learner:** serverless / scale-to-zero, but for *stateful* agent
sessions — the actor resumes exactly where it left off.

### Core components (taught by inspecting running pods)

| Component | Type | Role |
|---|---|---|
| `ate-api-server` (`ateapi`) | Deployment, gRPC :443 | Control plane: actor lifecycle, scheduling, suspend/resume. Bypasses kube-scheduler. |
| `ate-controller` | Deployment | Reconciles `WorkerPool` + `ActorTemplate` CRDs into Deployments. |
| `atelet` | DaemonSet, gRPC :8085 | Node supervisor: pulls images, manages sandbox lifecycle, the only component that touches object storage. |
| `atenet-router` | Deployment | Envoy L7 router; resolves the target worker per request (`ResumeActor`). |
| `valkey-cluster-{0..5}` | StatefulSet | State store: actor/worker records, locks. |
| `rustfs` | Deployment | S3-compatible object storage holding zstd snapshot images. |

### Key resources (taught at deploy time)

- **`WorkerPool`** (`ate.dev/v1alpha1`) — pool of warm standby pods (`replicas`, `ateomImage`, `sandboxClass`).
- **`ActorTemplate`** (`ate.dev/v1alpha1`) — immutable actor "class"; creating one triggers a **Golden Snapshot** (version 0).
- **`Actor`** / **`Worker`** — dynamic records (in Valkey, not CRDs): an Actor is one instance (`RUNNING`/`SUSPENDED`); a Worker is one pod (`IDLE`/`BUSY`), hosting at most one actor.
- **`SandboxAgent`** (`kagent.dev/v1alpha2`, `platform: substrate`) — kagent's per-session declarative agent that runs as a substrate actor. **Runtime must be Go** (Python ADK is incompatible with gVisor checkpointing).

## 3. Deployment approach (locked decisions)

- **Release mode** — published OCI Helm charts only; no source builds, no repo clones, no
  local registry mirror, no `PodCertificate` feature gate. Stock kind cluster + JWT auth.
  Based on `christian-posta/kagent@ceposta-substrate-demo:demo/substrate-demo-release.md`.
- **OSS kagent** — `ghcr.io/kagent-dev/kagent` v0.9.7 (first release with substrate PR #1981).
- **Versions pinned:** substrate `0.0.6`, kagent `0.9.7`, `ateom-gvisor:v0.0.6`.
- **Who runs the installs (Approach A):** `track_scripts/setup-server` pre-bakes the slow/
  fragile parts (tooling install, kind cluster creation, image pre-pull). The **learner runs
  the meaningful `helm install` commands** for substrate and kagent inside the challenges —
  that is the lesson. Each challenge has a `solve-server` with the exact commands (Instruqt's
  "solve" button) and a `check-server` that validates progress.

## 4. Infrastructure

- **VM:** single GCP VM named `server`, machine type **n1-standard-8** (substrate runs
  valkey-cluster ×6 + rustfs + atelet DaemonSet + kagent + worker pool; n1-standard-4 is too
  tight). Docker is assumed present on the base image; if the chosen image lacks it, setup
  installs Docker.
- **Tools:** `track_scripts/setup-server` installs/pins `kind`, `kubectl`, `helm`, and
  `grpcurl` (used in challenge 5 to drive the actor lifecycle directly), and creates the
  `kagent-substrate` kind cluster, then pre-pulls the substrate/kagent images.
- **Secret:** `OPENAI_API_KEY` — provided by Instruqt's secrets mechanism (declared in
  `config.yml` under `secrets:`), injected as an env var into the VM. This is the only secret
  required (no `AGENTGATEWAY_LICENSE_KEY` — we are on OSS kagent). The setup script exports
  it on its own line and validates it is non-empty before any helm install.

## 5. Challenge arc (6 challenges, balanced teaching)

1. **What is an Agent Substrate?** — concept-heavy notes (actors vs workers, golden
   snapshots, suspend/resume, the request path, drawn from the "atlas" learning material).
   Hands-on: confirm the kind cluster + tools are ready (`kubectl get nodes`).
2. **Install Agent Substrate** — `helm install substrate-crds` + `substrate` into
   `ate-system`; explore the architecture by inspecting the running pods and mapping each
   back to its role.
3. **Install kagent, wired to substrate** — `helm install kagent 0.9.7` with the
   `controller.substrate.*` + `substrateWorkerPool.*` flags; explain each flag and the
   `WorkerPool` / worker-slot model. Includes the documented `kubectl wait
   deploy/kagent-controller` fallback for the helm cold-start race. `check-server` verifies
   the integration via the controller's status API
   (`http://kagent-controller:8083/api/substrate/status` → `"enabled": true`).
4. **Deploy a SandboxAgent** — apply `hello-substrate.yaml`; explain `SandboxAgent` /
   `platform: substrate` / `ActorTemplate` / golden snapshot; `kubectl wait` for Ready
   (first golden snapshot takes ~60–90s). Inspect the generated `ActorTemplate`
   (`kubectl get actortemplates.ate.dev -A`).
5. **Chat & watch suspend/resume** — open the kagent UI (systemd port-forward + a `service`
   tab) and chat with the agent on the `/substrate` page. Then make the lifecycle explicit
   from the CLI: `kubectl get workerpools.ate.dev -A`, mint a short-lived token
   (`kubectl create token kagent-controller -n kagent --audience=api.ate-system.svc`),
   port-forward `svc/api`, and call `ateapi.Control/ListActors` + `ResumeActor` via
   `grpcurl` to watch an actor go `SUSPENDED` → `RUNNING`. This CLI path is the robust
   primary verification; the UI is the visual layer on top. The payoff moment.
6. **Recap & what's next** — scaling the WorkerPool (`kubectl scale workerpool`), the
   long-lived `AgentHarness` path, where substrate is headed, and cleanup notes.

## 6. Repo layout (matches existing tracks)

```
agent-substrate-kagent/
  track.yml          # metadata, tags, idle_timeout 1800, timelimit ~5400
  config.yml         # n1-standard-8 VM + OPENAI_API_KEY secret
  track_scripts/setup-server          # tools + kind cluster + image pre-pull
  01-what-is-substrate/{assignment.md,setup-server,check-server,solve-server}
  02-install-substrate/{assignment.md,setup-server,check-server,solve-server}
  03-install-kagent/{assignment.md,setup-server,check-server,solve-server}
  04-deploy-sandboxagent/{assignment.md,setup-server,check-server,solve-server}
  05-chat-suspend-resume/{assignment.md,setup-server,check-server,solve-server}
  06-recap/{assignment.md}
```

Each `assignment.md` uses the same frontmatter style as the existing kagent workshop
(`slug`, `id`, `title`, `teaser`, `notes`, `tabs`: Terminal, Code Editor, kagent UI service
tab). The kagent UI (`svc/kagent-ui`, container port 8080) is exposed via a systemd
port-forward to host port 8080 (same pattern as the enterprise workshop), and the `service`
tab points at host port 8080. Instruqt-assigned `id`s/`checksum` are generated by the Instruqt CLI after
authoring (same as prior tracks); the design uses placeholders.

## 7. Reference install commands (release mode)

These are the verbatim commands the learner runs (mirrored in each `solve-server`):

```bash
# Challenge 2 — substrate
helm upgrade --install substrate-crds \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate-crds \
  --version 0.0.6 --namespace ate-system --create-namespace --wait
helm upgrade --install substrate \
  oci://ghcr.io/kagent-dev/substrate/helm/substrate \
  --version 0.0.6 --namespace ate-system --wait --timeout 10m

# Challenge 3 — kagent wired to substrate
helm upgrade --install kagent-crds \
  oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --version 0.9.7 --namespace kagent --create-namespace --wait
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

The `controller.substrate.defaultWorkerPool.*` settings (from Levan's guide) register
`kagent/kagent-default` as the controller's default pool — without them, creating a
substrate agent fails because no pool is resolvable. They also let a `SandboxAgent` omit
`spec.substrate.workerPoolRef` entirely. `atenetRouterURL` and `ateApiTokenFile` wire the
controller to the substrate router and its ServiceAccount-token auth path.

> **Fallback only:** if the published controller/ui images fail to resolve in this
> environment, Levan's guide adds explicit `controller.image.*` / `ui.image.*` overrides.
> We omit these by default (the canonical release path resolves images fine) and document
> them as a troubleshooting fallback, not the happy path.

> **Provider note:** Levan's guide uses Anthropic (`ANTHROPIC_API_KEY`); we use OpenAI
> because that is the secret Instruqt provides. To switch, set `providers.default=anthropic`
> and `providers.anthropic.apiKey=...`.

```yaml
# Challenge 4 — hello-substrate.yaml
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

`workerPoolRef` is kept explicit here for teaching clarity. Because challenge 3 configures
`controller.substrate.defaultWorkerPool`, the shorter `substrate: {}` form also works.

```bash
# Challenge 5 — make suspend/resume explicit from the CLI (from resume-actor.md)
kubectl get workerpools.ate.dev -A
kubectl port-forward -n ate-system svc/api 18443:443 &     # separate terminal
TOKEN=$(kubectl create token kagent-controller -n kagent \
  --audience=api.ate-system.svc --duration=15m)
grpcurl -insecure -H "authorization: Bearer $TOKEN" -d '{}' \
  localhost:18443 ateapi.Control/ListActors          # note the SUSPENDED actor + its id
grpcurl -insecure -H "authorization: Bearer $TOKEN" \
  -d '{"actor_id":"<ACTOR_ID>"}' \
  localhost:18443 ateapi.Control/ResumeActor         # watch it flip to RUNNING
```

## 8. Risks & mitigations (flagged honestly)

1. **gVisor checkpoint/restore on kind-in-a-VM is the real unknown.** Substrate is pre-1.0
   and needs privileged worker pods + host kernel support. This cannot be fully de-risked
   from research; it requires a live test on the actual Instruqt VM. **Mitigation:** the lab
   is structured so challenges 1–4 (concept, deploy substrate, deploy kagent, deploy agent)
   still teach successfully even if challenge 5's live suspend/resume misbehaves. Challenge 5
   itself has two layers — the `grpcurl` `ListActors`/`ResumeActor` CLI path (robust,
   scriptable, used by `check-server`) and the UI `/substrate` page (visual). If gVisor
   checkpoint/restore is unavailable on the VM, the CLI still demonstrates actor state and we
   degrade gracefully. We test on a real VM before finalizing.
2. **Helm 10-min cold-start race** (controller restarts waiting on postgres). **Mitigation:**
   bake the `kubectl wait deploy/kagent-controller --for=condition=Available --timeout=10m`
   fallback into the assignment notes and `check-server`.
3. **Empty OpenAI key footgun** — inlining the env assignment on the same line as the helm
   `--set` evaluates to empty. **Mitigation:** `export` on its own line + validate non-empty
   in setup; document the warning in the assignment.
4. **TLS certs expire ~24h on idle clusters** — fine for a single lab session; noted in the
   recap challenge.
5. **Resource pressure** — valkey-cluster ×6 + rustfs + kagent + workers. **Mitigation:**
   n1-standard-8 VM; `substrateWorkerPool.replicas=1`.

## 9. Out of scope (YAGNI)

- The `AgentHarness` (OpenClaw / long-lived) path — mentioned in recap only.
- From-source builds, `ko`, local registry mirroring, `PodCertificate` mode.
- AAuth / identity minting deep-dive.
- Solo Enterprise kagent, AgentGateway, ClickHouse tracing.

## 10. Sources

- `christian-posta/kagent@ceposta-substrate-demo` — `demo/substrate-demo-release.md`,
  `demo/agents/hello-substrate.yaml`, `demo/up.sh`, `demo/down.sh`.
- `agent-substrate/substrate@main` — `README.md`, `docs/architecture.md`, `docs/api-guide.md`,
  `hack/*.sh`.
- `christian-posta/substrate@ceposta-learn` — `atlas/` learning docs (topology, flows, concepts).
- `AdminTurnedDevOps/agentic-demo-repo@main` — `substrate/kagent-substrate/`:
  `kagent-substrate-install-and-configure.md` (fuller install flag set incl.
  `defaultWorkerPool`, controller status API), `kagent-substrate.md` (AgentHarness path —
  out of scope, needs GCS), `resume-actor.md` (grpcurl `ListActors`/`ResumeActor` — adopted
  for challenge 5).
- Existing repo patterns: `kagent-enterprise-workshop/` (track.yml, config.yml, setup/check/solve structure).
