# Akka Sample: Multi-Agent Collaboration (Supervisor)

A supervisor agent routes incoming tasks to specialist workers — a researcher and a chart generator — and synthesises their outputs into a unified response. Demonstrates the **delegation-supervisor-workers** coordination pattern with an embedded before-tool-call guardrail that scopes what each worker is permitted to do.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound task stream and the worker tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.supervisor-workers  ~/my-projects/supervisor-workers
cd ~/my-projects/supervisor-workers
```

(Optional) Edit `SPEC.md` to change the task prompts the simulator drips, swap a worker, or extend the data model.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskSupervisor** — AutonomousAgent that classifies an incoming task and routes it to the appropriate worker(s).
- **ResearchWorker** — AutonomousAgent that runs web-search-style lookups (seeded tool, no real external call).
- **ChartWorker** — AutonomousAgent that generates structured chart data from a description (seeded tool).
- **CollaborationWorkflow** — Workflow that sequences the supervisor's routing decision, the parallel worker runs, and the final synthesis call.
- **TaskEntity** — EventSourcedEntity holding the full task lifecycle.
- **TaskRequestQueue** — EventSourcedEntity that logs each submitted task for replay and audit.
- **TaskView** — projection the UI streams via SSE.
- **TaskRequestConsumer** — Consumer that starts a workflow per queue event.
- **TaskSimulator** — TimedAction that drips sample tasks every 60 seconds.
- **EvalSampler** — TimedAction that scores completed tasks every 5 minutes.
- **CollaborationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `CollaborationTask` record fields (e.g., add a `priority` field).
- `prompts/supervisor.md` — narrow the routing logic to a specific set of worker types.
- `eval-matrix.yaml` — adjust the guardrail hook if you add a real external tool call.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → record enters `ROUTING`, then `IN_PROGRESS`, then `COMPLETED`.
2. A worker tool call is blocked by the before-tool-call guardrail → task enters `BLOCKED` with a reason.
3. Eval-event sampling captures one synthesis decision and surfaces the score on the App UI.

## License

Apache 2.0.
