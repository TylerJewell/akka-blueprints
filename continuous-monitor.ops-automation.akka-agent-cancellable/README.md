# Akka Sample: Activity Interrupt / Cancellation

A continuous background worker accepts long-running automation tasks, executes them via an AI agent, and exposes operator-initiated cancellation at any point mid-flight. Demonstrates the **continuous-monitor** coordination pattern wired with two halt mechanisms: an operator-stop control that surfaces a clean cancellation API, and a graceful-degradation path that runs compensating cleanup steps whenever a task is interrupted before completion.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.akka-agent-cancellable  ~/my-projects/activity-interrupt
cd ~/my-projects/activity-interrupt
```

(Optional) Edit `SPEC.md` to point the `TaskQueue` at a real task source (a database table, a message bus) or to keep the built-in simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskPoller** — TimedAction firing every 20 s that pulls pending tasks from a simulated task source.
- **TaskRunnerAgent** — AutonomousAgent that executes a task step-by-step, checking for a cancellation signal between iterations.
- **CleanupAgent** — Agent (typed) that produces a structured cleanup plan when a task is cancelled mid-flight.
- **ActivityWorkflow** — Workflow per task; steps: startExecution → executeStep (looped) → cancel branch → cleanup → finalise.
- **ActivityEntity** — EventSourcedEntity holding the full lifecycle per task (QUEUED → RUNNING → CANCELLING → CANCELLED / COMPLETED / FAILED).
- **ActivityView + ActivityEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **StaleActivityReaper** — TimedAction running every 5 minutes; finds activities stuck in RUNNING beyond a configured timeout and emits a cancellation event.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated task source for a real queue (Kafka, SQS, database).
- `SPEC.md §5` — extend the `ActivityTask` record with domain-specific fields (`priority`, `correlationId`, `tenantId`).
- `prompts/task-runner.md` — narrow the agent to a specific automation class (infra provisioning, report generation, data pipeline).
- `eval-matrix.yaml` — wire a real cancellation hook by updating the halt control's implementation block.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated task arrives → it enters RUNNING and step progress is visible in the UI.
2. An operator clicks Cancel mid-run → the task enters CANCELLING, cleanup runs, the task ends in CANCELLED.
3. A task that runs to completion reaches COMPLETED without cleanup.
4. A task stuck past the timeout is auto-cancelled by `StaleActivityReaper`.

## License

Apache 2.0.
