# Akka Sample: SQL Generator

An agent translates a natural-language question into a SQL query, then a validator agent runs `EXPLAIN` against the target schema; if the query fails validation, structured feedback is returned to the generator and the loop retries until the query passes or the attempt ceiling is reached. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint models the schema registry and the EXPLAIN validator inside the service; no external database is required.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.sql-gen-validator-loop  ~/my-projects/sql-gen-validator-loop
cd ~/my-projects/sql-gen-validator-loop
```

(Optional) Edit `SPEC.md` to change the sample schemas, the mutation-block list, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that translates a natural-language question into a SQL query, accepting prior validator feedback when revising.
- **ValidatorAgent** — AutonomousAgent that runs a schema-aware `EXPLAIN` check on the query and returns either `VALID` or `INVALID` with a typed `ValidationNotes` payload.
- **SqlWorkflow** — Workflow that drives the generate → guardrail → validate → revise loop up to a configurable retry ceiling, transitioning the request to `ACCEPTED` on validator approval or to `FAILED_FINAL` when the ceiling is hit.
- **QueryRequestEntity** — EventSourcedEntity that holds the full request lifecycle, every attempt's SQL, every validation result, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **QueryRequestView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample natural-language question every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **SqlEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `QueryRequest` record fields (e.g., raise `maxAttempts`, add schema tables).
- `prompts/generator.md` — constrain the SQL dialect (e.g., Postgres only, no CTEs).
- `prompts/validator.md` — change the EXPLAIN parsing logic or add additional semantic rules.
- `eval-matrix.yaml` — tighten enforcement on the mutation guardrail (currently blocking) or the eval-event recorder (currently non-blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a natural-language question → request progresses `GENERATING` → `VALIDATING` → `ACCEPTED` within the retry ceiling.
2. Force-fail validation → request hits `FAILED_FINAL` after the configured number of attempts; the entity preserves every attempt and every validation result for audit.
3. The mutation guardrail blocks a query containing `DROP`, `DELETE`, `UPDATE`, or `INSERT` before the validator runs.
4. Each completed cycle emits a `ValidationEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
