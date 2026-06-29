# Akka Sample: Parallel Best-of-N

A supervisor fans a task out to N parallel worker agents (illustrated here with diverse translation variants), then a synthesizer selects the best result. Demonstrates the **delegation-supervisor-workers** coordination pattern with concurrent branch execution, durable join, output guardrail, and decision eval-event logging.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the task queue and translation tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.akka-parallel-best-of-n  ~/my-projects/parallel-best-of-n
cd ~/my-projects/parallel-best-of-n
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or number of parallel branches.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TranslationSupervisor** — AutonomousAgent that decomposes a translation job into N variant instructions and later selects the best completed variant.
- **TranslationWorker** — AutonomousAgent that produces one translation variant given a style instruction.
- **TranslationWorkflow** — Workflow that fans the job out to N `TranslationWorker` branches in parallel, collects results, and calls `TranslationSupervisor` for selection.
- **TranslationJobEntity** — EventSourcedEntity holding the job's lifecycle from submission through selection.
- **TranslationView** — projection the UI streams via SSE.
- **TranslationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the number of parallel branches (default 3) or the style instructions given to each worker.
- `SPEC.md §5` — adjust the `TranslationJob` record fields (e.g., add `targetAudience`).
- `prompts/supervisor.md` — swap translation for a different generative task.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real terminology API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a translation job → job enters `PENDING`, then `IN_PROGRESS`, then `SELECTED`.
2. Workers fail-fast → if any branch times out, the job enters `PARTIAL` and the supervisor selects from the remaining variants.
3. Output guardrail catches a policy violation and transitions the job to `BLOCKED`.
4. Eval-event sampling captures one selection decision and surfaces the score on the App UI.

## License

Apache 2.0.
