# Akka Sample: User Confirmation Agents

An agent proposes an action and pauses. A user reads the pending action in the UI, confirms or cancels via the API, and only then does the action group execute. No action runs without an explicit human go-ahead.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.general.user-confirmation-agent  ~/my-projects/user-confirmation-agent
cd ~/my-projects/user-confirmation-agent
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `PlannerAgent` (AutonomousAgent) — receives a request, proposes one or more actions, returns a typed `ActionPlan{actions}`.
- `ExecutorAgent` (AutonomousAgent) — executes a confirmed action plan, returns a typed `ExecutionResult{summary, executedAt}`.
- `ConfirmationWorkflow` (Workflow) — a 3-task graph: plan → await confirmation → execute.
- `RequestEntity` (EventSourcedEntity) — holds the request lifecycle and its events.
- `RequestsView` (View) — a read model the UI queries and streams over SSE.
- `ConfirmationEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/planner-agent.md` — the action-planning instructions.

## What gets validated

- Submitting a request produces an action plan that appears in `PENDING_CONFIRMATION` with non-empty `proposedActions`.
- Confirming a `PENDING_CONFIRMATION` request drives it to `EXECUTED` with a non-null `executionSummary`.
- Cancelling a `PENDING_CONFIRMATION` request drives it to terminal `CANCELLED` and the reason is shown.
- The execute step never runs unless the request is `CONFIRMED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
