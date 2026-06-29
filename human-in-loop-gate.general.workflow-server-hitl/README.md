# Akka Sample: WorkflowServer (REST + Streaming + HITL)

A workflow is exposed as a REST API. Requests stream progress events back to the caller, and the workflow pauses at a human decision gate before proceeding to completion.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.workflow-server-hitl  ~/my-projects/workflow-server-hitl
cd ~/my-projects/workflow-server-hitl
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `AnalysisAgent` (AutonomousAgent) — analyzes a submitted job request and returns a typed `AnalysisSummary{findings, riskLevel}`.
- `CompletionAgent` (AutonomousAgent) — finalizes an approved job and returns a typed `JobResult{output, completedAt}`.
- `JobWorkflow` (Workflow) — a 3-task graph: analyze → await approval → complete.
- `JobEntity` (EventSourcedEntity) — the job lifecycle and its events.
- `JobsView` (View) — a read model the UI queries and streams over SSE.
- `WorkflowEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/analysis-agent.md` — the analysis instructions and scope.

## What gets validated

- Submitting a job request triggers analysis that appears in `ANALYZED` with non-empty findings.
- Approving an `ANALYZED` job drives it to `COMPLETED` with a non-null output.
- Rejecting an `ANALYZED` job drives it to terminal `REJECTED` with the reason shown.
- The completion step never runs unless the job is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
