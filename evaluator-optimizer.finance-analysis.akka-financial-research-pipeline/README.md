# Akka Sample: Financial Research Multi-Agent

A planner agent breaks a research query into sub-tasks; a search agent retrieves source material; an analyst agent synthesises findings; a writer agent produces the final report; a verifier agent quality-checks the analyst and writer output before the report is released. Demonstrates the **evaluator-optimizer** coordination pattern with durable workflow retries, sector-aware sanitisation, an output guardrail on the final report, and a compliance review queue for human-on-the-loop oversight.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound query surface and the outbound report distribution are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.finance-analysis.akka-financial-research-pipeline  ~/my-projects/financial-research-pipeline
cd ~/my-projects/financial-research-pipeline
```

(Optional) Edit `SPEC.md` to change the sector scope, the verification rubric, the confidence threshold, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that decomposes an incoming research query into an ordered list of sub-tasks, each with a scope and a sector tag.
- **SearchAgent** — AutonomousAgent that executes a single sub-task's retrieval step, returning a `SourceBundle` of ranked excerpts.
- **AnalystAgent** — AutonomousAgent that synthesises source bundles into a structured `AnalysisSection` with claims and supporting evidence.
- **WriterAgent** — AutonomousAgent that assembles analysis sections into a prose `ReportDraft` observing a word-count ceiling.
- **VerifierAgent** — AutonomousAgent that quality-checks the draft against factual consistency, source coverage, and sector-appropriateness rules; returns `APPROVE` or `REVISE` with structured notes.
- **ResearchWorkflow** — Workflow that runs plan → search → analyse → write → verify → revise loop up to a configurable retry ceiling; halts with the best-scoring draft on ceiling exhaustion.
- **ResearchEntity** — EventSourcedEntity that holds the full pipeline lifecycle: every sub-task, every source bundle, every analysis section, every draft, every verification verdict, and the final outcome.
- **QueryQueue** — EventSourcedEntity that logs each submitted query for replay and audit.
- **ReportsView** — read-side projection the UI lists and streams via SSE.
- **QueryConsumer** — Consumer that starts a workflow per inbound query.
- **QuerySimulator** — TimedAction that drips a sample query every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records per-verification eval events each cycle (control E1).
- **ResearchEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Report` record fields (e.g., raise `maxVerifyAttempts`, change the word-count ceiling).
- `prompts/planner.md` — narrow the sector scope or change how sub-tasks are ordered.
- `prompts/verifier.md` — tighten or relax the verification rubric.
- `eval-matrix.yaml` — adjust enforcement level on the output guardrail or the HOTL queue threshold.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → report progresses `PLANNING` → `RESEARCHING` → `ANALYSING` → `WRITING` → `VERIFYING` → `APPROVED` within the retry ceiling.
2. Force-fail the rubric → report hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every draft and every verification verdict.
3. The sector sanitiser strips disallowed financial identifiers before the writer sees the analysis; the sanitised field is visible in the App UI.
4. The output guardrail blocks a report draft whose word count exceeds the ceiling; the writer re-drafts before the verifier runs.
5. The HOTL queue receives an entry for every report that the verifier marks `APPROVE`; a compliance reviewer can inspect or override before release.

## License

Apache 2.0.
