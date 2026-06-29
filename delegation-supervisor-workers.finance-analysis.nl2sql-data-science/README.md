# Akka Sample: Multi-Agent NL2SQL Data Science

A query coordinator translates a natural-language financial question into SQL, delegates data retrieval and statistical modeling to two specialist agents running **in parallel**, then merges their outputs into a unified data science report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance for SQL execution safety.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the BigQuery schema and the BQML modeling tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.nl2sql-data-science  ~/my-projects/nl2sql-data-science
cd ~/my-projects/nl2sql-data-science
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QueryCoordinator** — AutonomousAgent that translates a natural-language question into a SQL plan and synthesises worker outputs into the final report.
- **DataRetriever** — AutonomousAgent that executes SQL against the modelled BigQuery schema and returns a structured dataset.
- **StatisticsModeler** — AutonomousAgent that applies BQML-style statistical operations and returns a model result.
- **AnalysisWorkflow** — Workflow that fans the work out to DataRetriever and StatisticsModeler in parallel, then calls QueryCoordinator for synthesis.
- **AnalysisJobEntity** — EventSourcedEntity holding the full analysis lifecycle.
- **QueryQueue** — EventSourcedEntity logging each submitted question for audit.
- **AnalysisView** — projection the UI streams via SSE.
- **AnalysisEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulator's seed questions, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AnalysisJob` record fields (e.g., add `confidenceInterval`).
- `prompts/coordinator.md` — narrow the SQL dialect or restrict the schema the coordinator may query.
- `eval-matrix.yaml` — add additional guardrail hooks if you wire a live BigQuery connection.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a financial question → job enters `TRANSLATING`, then `RUNNING`, then `COMPLETE`.
2. Workers fail-fast → if either DataRetriever or StatisticsModeler times out, the job enters `DEGRADED` with whichever partial output exists.
3. SQL guardrail blocks a destructive query → the SQL plan is rejected before execution and the job enters `BLOCKED`.
4. Eval-event sampling captures one NL2SQL translation and surfaces the correctness score on the App UI.

## License

Apache 2.0.
