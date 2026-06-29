# Akka Sample: Parallel Task Decomposition

A task coordinator decomposes an incoming job into independent subtasks and dispatches them to two worker agents running **in parallel**, then merges their outputs into one consolidated result. Demonstrates the **delegation-supervisor-workers** coordination pattern in the general domain with no external service dependencies.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound job stream and worker tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.parallel-fanout  ~/my-projects/parallel-fanout
cd ~/my-projects/parallel-fanout
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskCoordinator** — AutonomousAgent that decomposes an incoming job into two parallel work items and merges the workers' results into a consolidated report.
- **SubtaskWorkerA** — AutonomousAgent that processes the first subtask dimension (structural analysis).
- **SubtaskWorkerB** — AutonomousAgent that processes the second subtask dimension (contextual enrichment).
- **TaskExecutionWorkflow** — Workflow that fans work out to SubtaskWorkerA and SubtaskWorkerB in parallel, then asks TaskCoordinator to consolidate.
- **TaskJobEntity** — EventSourcedEntity holding the full job lifecycle.
- **TaskJobView** — projection the UI streams via SSE.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the job payloads the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `TaskJob` record fields (e.g., add `confidenceScore`).
- `prompts/coordinator.md` — narrow the coordinator to a specific domain.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real processing API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job → job progresses `QUEUED → PROCESSING → CONSOLIDATED` within 60 s; UI reflects each transition via SSE.
2. Workers fail-fast → if either SubtaskWorkerA or SubtaskWorkerB times out, the job enters `DEGRADED` with whichever partial output exists.
3. Output validation blocks a malformed consolidation → job enters `REJECTED` with a failure reason surfaced in the App UI.

## License

Apache 2.0.
