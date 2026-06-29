# Akka Sample: Orchestrator-Workers

A central orchestrator decomposes a multi-part task at runtime, fans out sub-tasks to specialist worker LLMs, and synthesises their results into a unified answer. Demonstrates the **delegation-supervisor-workers** coordination pattern with a plan-validation guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the task queue and the worker tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.orchestrator-workers-fanout  ~/my-projects/orchestrator-workers
cd ~/my-projects/orchestrator-workers
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskOrchestrator** — AutonomousAgent that decomposes a task request into a dynamic plan and synthesises worker results.
- **WriterWorker** — AutonomousAgent that handles text-production sub-tasks.
- **AnalyzerWorker** — AutonomousAgent that handles reasoning and evaluation sub-tasks.
- **TaskWorkflow** — Workflow that fans the plan out to workers in parallel, then calls the Orchestrator for synthesis.
- **TaskRequestEntity** — EventSourcedEntity holding the full task lifecycle.
- **TaskView** — projection the UI streams via SSE.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task types the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `TaskRequest` record fields (e.g., add `priorityLevel`).
- `prompts/writer-worker.md` — narrow the worker to a single output format.
- `eval-matrix.yaml` — add an `after-agent-invocation` guardrail if you add external tool calls.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task → request enters `PLANNING`, then `IN_PROGRESS`, then `COMPLETED`.
2. Workers fail-fast → if any worker times out, the request enters `DEGRADED` with partial output.
3. The plan-validation guardrail blocks a malformed plan before workers are dispatched.
4. The eval sampler scores a completed task and the score appears in the App UI.

## License

Apache 2.0.
