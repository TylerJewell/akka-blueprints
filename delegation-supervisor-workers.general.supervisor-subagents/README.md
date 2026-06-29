# Akka Sample: Supervisor with Subagents

A supervisor agent receives a task, decides which specialized subagents to invoke, delegates subtasks to them, and assembles their outputs into a final result. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound task stream and the subagent tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.supervisor-subagents  ~/my-projects/supervisor-subagents
cd ~/my-projects/supervisor-subagents
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskSupervisor** — AutonomousAgent that classifies an incoming task and routes it to the appropriate subagents.
- **DataSubagent** — AutonomousAgent that retrieves and formats structured data for a subtask.
- **SummarySubagent** — AutonomousAgent that produces a human-readable narrative from structured inputs.
- **TaskOrchestrationWorkflow** — Workflow that runs the supervisor routing decision, fans subtasks out to the selected subagents, then reassembles their outputs.
- **TaskEntity** — EventSourcedEntity holding the full task lifecycle.
- **TaskView** — projection the UI streams via SSE.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample tasks the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `TaskResult` record fields (e.g., add `confidenceScore`).
- `prompts/supervisor.md` — narrow the routing logic to a specific task taxonomy.
- `eval-matrix.yaml` — add additional guardrails if you extend the subagent roster.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → request enters `RECEIVED`, moves to `ROUTING`, then `IN_PROGRESS`, then `COMPLETED`.
2. Subagent fails or times out → the task enters `DEGRADED` with whatever partial output was collected.
3. The routing guardrail blocks a task whose routing decision violates policy → task enters `BLOCKED`.
4. The eval-event sampler scores a completed routing decision and the score appears in the App UI.

## License

Apache 2.0.
