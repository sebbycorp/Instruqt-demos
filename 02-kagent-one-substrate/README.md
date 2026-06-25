# Agent Substrate with kagent (Single Lab)

A **single-challenge** Instruqt track teaching Agent Substrate, then deploying Agent
Substrate + OSS kagent on a `kind` cluster and running a `SandboxAgent` inside a gVisor
actor — all in one end-to-end lab.

This is the condensed edition of the 6-challenge
[`01-kagent-agent-substrate-workshop`](../01-kagent-agent-substrate-workshop) track:
same content and infra, collapsed into one challenge.

## The lab

**Agent Substrate with kagent, end to end** walks through, in five parts:

1. Confirm the `kind` cluster and tooling.
2. **Install Agent Substrate** — substrate 0.0.6 into `ate-system`.
3. **Install kagent, wired to substrate** — kagent 0.9.7 + default `WorkerPool`.
4. **Deploy a SandboxAgent** — a declarative Go agent running as a substrate actor.
5. **Chat & watch suspend/resume** — UI chat + `grpcurl` actor lifecycle, plus scaling
   and what's next.

## Infra

- Single GCP VM (`server`, n1-standard-8) running a `kind` cluster.
- Secret: `OPENAI_API_KEY` (Instruqt-provided).
- Versions: substrate `0.0.6`, kagent OSS `0.9.7`, `ateom-gvisor:v0.0.6`.

## Design & plan

- Spec: `../docs/superpowers/specs/2026-06-23-agent-substrate-kagent-lab-design.md`
- Plan: `../docs/superpowers/plans/2026-06-23-agent-substrate-kagent-lab.md`

> Agent Substrate is pre-1.0; gVisor checkpoint/restore on kind requires a live VM test.
