# Akka Sample: DevOps Multi-Agent

A DevOps supervisor delegates infrastructure, deployment, and observability subtasks to three specialist agents running in parallel, then consolidates their outputs into a unified change-readiness report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance controls for destructive operations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound pipeline trigger and the tool models are wired inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.ops-automation.devops-team  ~/my-projects/devops-multi-agent
cd ~/my-projects/devops-multi-agent
```

(Optional) Edit `SPEC.md` to change the pipeline trigger interval, model provider, or approval threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OpsCoordinator** — AutonomousAgent that parses a change request, delegates subtasks, and consolidates results into a change-readiness report.
- **InfraAgent** — AutonomousAgent that checks infrastructure drift, quota availability, and dependency health.
- **DeployAgent** — AutonomousAgent that evaluates deployment manifests, rollback posture, and canary gate status.
- **ObservabilityAgent** — AutonomousAgent that reviews alert thresholds, SLO burn rates, and on-call coverage.
- **OpsWorkflow** — Workflow that fans work to the three specialists in parallel, joins their results, runs a pre-execution guardrail, and parks the plan for human approval on production changes.
- **ChangeRequestEntity** — EventSourcedEntity holding the full change lifecycle.
- **OpsView** — projection the UI streams via SSE.
- **OpsEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the pipeline trigger interval, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ChangeRequest` record fields (e.g., add `changeWindow`).
- `prompts/infra-agent.md` — narrow the agent to a specific cloud provider.
- `eval-matrix.yaml` — tighten the guardrail to run on specific tool categories.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a change request → request progresses `PLANNING → IN_PROGRESS → AWAITING_APPROVAL` (production) or `APPROVED` (non-production).
2. Guardrail blocks a request with destructive tool calls flagged before execution.
3. An operator issues a halt signal; any in-flight workflow enters `HALTED` immediately.
4. A production change parked for human approval advances to `APPROVED` after the approval API call.

## License

Apache 2.0.
