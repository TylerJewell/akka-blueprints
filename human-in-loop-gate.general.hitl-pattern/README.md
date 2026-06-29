# Akka Sample: Human-in-the-Loop

A task is submitted by a user, processed by an agent, paused at a human approval gate, and finalized by a second agent only after a person approves it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.hitl-pattern  ~/my-projects/hitl-pattern
cd ~/my-projects/hitl-pattern
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `TaskProcessorAgent` (AutonomousAgent) — processes a submitted task request and returns a typed `ProcessedTask{summary, recommendation}`.
- `TaskFinalizerAgent` (AutonomousAgent) — finalizes an approved task and returns a typed `FinalizedTask{outcome, finalizedAt}`.
- `ApprovalWorkflow` (Workflow) — a 3-task graph: process → await approval → finalize.
- `TaskEntity` (EventSourcedEntity) — the task lifecycle and its events.
- `TasksView` (View) — a read model the UI queries and streams over SSE.
- `TaskEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/task-processor-agent.md` — the processing instructions and domain context.

## What gets validated

- Submitting a request processes a task that appears in `PROCESSED` with a non-empty recommendation.
- Approving a `PROCESSED` task drives it to `FINALIZED` within ~30 s.
- Rejecting a `PROCESSED` task drives it to terminal `REJECTED` with the reason shown.
- The finalize step never runs unless the task is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
