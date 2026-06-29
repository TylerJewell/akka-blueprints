# Akka Sample: Plan Approval Gate (HITL)

An agent produces an execution plan; the workflow pauses at a human approval gate; the human reviews, edits, or cancels the plan through the API; execution resumes with full memory preserved only after approval.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.plan-approval-gate  ~/my-projects/plan-approval-gate
cd ~/my-projects/plan-approval-gate
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `PlanningAgent` (AutonomousAgent) — generates a step-by-step execution plan for the given goal, returns a typed `AgentPlan{steps, rationale}`.
- `ExecutionAgent` (AutonomousAgent) — carries out an approved plan and returns a typed `ExecutionResult{outcome, completedAt}`.
- `PlanWorkflow` (Workflow) — a 3-task graph: plan → await approval → execute.
- `PlanEntity` (EventSourcedEntity) — holds the plan lifecycle and all of its events.
- `PlansView` (View) — a CQRS read model the UI queries and streams over SSE.
- `PlanEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/planning-agent.md` — the planning instructions and output format.

## What gets validated

- Submitting a goal produces a plan that appears in `PLANNED` with non-empty steps within ~30 s.
- Approving a `PLANNED` plan drives it to `EXECUTED` with a non-null outcome.
- Cancelling a `PLANNED` plan drives it to a terminal `CANCELLED` state with the reason shown.
- Editing a plan before approval persists the edited steps and keeps status in `PLANNED`.
- The execute step never runs unless the plan is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
