# Akka Sample: Async HITL Tool Gate

An async workflow that pauses before a tool call so a human can review and approve the agent's planned action before it executes.

## Prerequisites

- Claude Code with the Akka plugin installed. Install docs: https://doc.akka.io/.
- An AI provider key, sourced one of five ways at generation time (env var already set, named env var, env file path, secrets-store URI, or a mock provider that needs no key). The key value is never written to disk.
- Host software for this integration form (Runs out of the box): None.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` (system name, model provider, sample request topics).
3. In Claude Code, from inside the folder, run:

```
/akka:specify @SPEC.md
```

`SPEC.md` Section 12 auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. You type one command total.

## What you'll get

- Two AutonomousAgents: `ActionAgent` (plans a tool call) and `ExecutorAgent` (runs the approved tool).
- One `ApprovalWorkflow` orchestrating plan → await-approval → execute.
- Two EventSourcedEntities: `ActionEntity` and `InboundRequestQueue`.
- One `ActionsView` (CQRS read model, SSE-streamed).
- One `RequestConsumer`, two TimedActions (`RequestSimulator`, `StuckActionMonitor`).
- Two HttpEndpoints (`ActionEndpoint`, `AppEndpoint`) and a `Bootstrap` service-setup.
- A single self-contained 5-tab UI at `static-resources/index.html`.

## Customise before generating

- System name and one-line pitch — `SPEC.md` Section 1.
- Model provider default — `SPEC.md` Section 11 (`application.conf` block).
- Sample request topics — the JSONL named in `SPEC.md` Section 11.

## What gets validated

The user journeys in `reference/user-journeys.md`:

- Submit a request and watch the agent plan an action.
- Approve a planned action and watch the tool execute.
- Reject a planned action.
- A planned action left untouched escalates automatically.

## License

Apache 2.0.
