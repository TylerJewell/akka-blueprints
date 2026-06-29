# Akka Sample: Sub-Question Query Engine

A query supervisor decomposes a multi-part question into focused sub-questions, dispatches each to a dedicated index tool, then synthesises a combined answer from all individual results. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the question decomposition, index tools, and answer synthesis are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.sub-question-supervisor  ~/my-projects/sub-question-supervisor
cd ~/my-projects/sub-question-supervisor
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QuerySupervisor** — AutonomousAgent that decomposes a question into sub-questions and synthesises the combined answer.
- **IndexWorker** — AutonomousAgent that retrieves a result from a designated index for one sub-question.
- **QueryOrchestrationWorkflow** — Workflow that fans sub-questions out to IndexWorker instances in parallel, collects results, and calls QuerySupervisor for synthesis.
- **QuerySessionEntity** — EventSourcedEntity holding the full query lifecycle.
- **IndexCallQueue** — EventSourcedEntity logging each submitted question for audit.
- **QuerySessionView** — projection the UI streams via SSE.
- **QueryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the question topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `QuerySession` record fields (e.g., add `confidenceScore`).
- `prompts/supervisor.md` — narrow the decomposition strategy (e.g., enforce exactly three sub-questions).
- `eval-matrix.yaml` — add an output guardrail if you surface answers to end-users outside a human-review step.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a complex question → session enters `DECOMPOSING`, then `RETRIEVING`, then `SYNTHESISED`.
2. A sub-question tool call is intercepted by the before-tool-call guardrail; malformed sub-questions are blocked before the index is queried.
3. Worker timeout → session enters `PARTIAL` with whichever index results arrived.
4. Eval-event sampling scores one synthesis decision and surfaces the score on the App UI.

## License

Apache 2.0.
