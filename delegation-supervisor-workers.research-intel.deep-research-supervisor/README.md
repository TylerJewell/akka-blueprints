# Akka Sample: Deep Research Supervisor

A supervisor agent decomposes a research question into targeted subqueries, delegates each to parallel worker agents (search, extract, summarise), then synthesises the collected results into a cited deep-research report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the search corpus and extraction tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.deep-research-supervisor  ~/my-projects/deep-research
cd ~/my-projects/deep-research
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or report schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchSupervisor** — AutonomousAgent that decomposes a question into subqueries and synthesises the final report.
- **SearchWorker** — AutonomousAgent that retrieves raw passages for a subquery (modelled with seeded tools).
- **ExtractionWorker** — AutonomousAgent that distils key claims from raw passages.
- **SummaryWorker** — AutonomousAgent that produces a concise summary for one subquery's extracted claims.
- **DeepResearchWorkflow** — Workflow that fans subquery work out to SearchWorker, ExtractionWorker, and SummaryWorker in a pipeline per subquery, then calls ResearchSupervisor for synthesis.
- **ResearchReportEntity** — EventSourcedEntity holding the report's full lifecycle.
- **ReportView** — projection the UI streams via SSE.
- **ReportEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the question topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ResearchReport` record fields (e.g., add `confidenceScore`).
- `prompts/search-worker.md` — narrow the worker to a specific knowledge domain.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real search API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a research question → report enters `PLANNING`, then `IN_PROGRESS`, then `SYNTHESISED`.
2. Worker failure → if the SearchWorker times out for any subquery, the report enters `DEGRADED` with partial subquery results.
3. Eval-event sampling captures one synthesis decision and surfaces the citation-grounding score on the App UI.

## License

Apache 2.0.
