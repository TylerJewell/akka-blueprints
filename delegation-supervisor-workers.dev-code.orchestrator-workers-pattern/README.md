# Akka Sample: Orchestrator-Workers Workflow

A central orchestrator receives a multi-file code-edit task, decomposes it into per-file work items, dispatches each to a dedicated worker agent, and synthesises the final changeset — demonstrating the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the task queue and the file-edit tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.dev-code.orchestrator-workers-pattern  ~/my-projects/orchestrator-workers
cd ~/my-projects/orchestrator-workers
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or file-edit schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Orchestrator** — AutonomousAgent that decomposes an edit task into per-file instructions and synthesises the final changeset from worker outputs.
- **FileEditor** — AutonomousAgent that applies targeted edits to one file at a time.
- **EditReviewer** — AutonomousAgent that checks a single edited file for correctness and consistency.
- **EditWorkflow** — Workflow that fans work out to FileEditor agents in parallel, collects reviews, then calls Orchestrator for synthesis.
- **EditTaskEntity** — EventSourcedEntity holding the full edit-task lifecycle.
- **EditJobQueue** — EventSourcedEntity logging submitted tasks for replay and audit.
- **EditTaskView** — projection the UI streams via SSE.
- **EditEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulator topics, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `EditTask` record fields (e.g., add `targetLanguage`).
- `prompts/orchestrator.md` — narrow the orchestrator to a specific programming language or coding style guide.
- `eval-matrix.yaml` — add additional guardrail controls if you wire a real code-execution sandbox.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a code-edit task → task enters `PLANNING`, then `IN_PROGRESS`, then `COMPLETED`; UI reflects each transition via SSE.
2. A worker's tool call is intercepted by the before-tool-call guardrail; forbidden write paths are blocked and the task enters `BLOCKED`.
3. Eval-event sampling captures one synthesis decision and surfaces the quality score in the App UI.

## License

Apache 2.0.
