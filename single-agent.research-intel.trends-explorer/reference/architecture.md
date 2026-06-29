# Architecture — google-trends-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `TrendEndpoint` accepts a submission and writes a `TrendQuerySubmitted` event onto `TrendRequestEntity`. The `TrendDataFetcher` Consumer subscribes, looks up the matching seeded dataset for the requested region/window/category combination, and writes the assembled payload back via `attachTrendData`. The same Consumer then starts a `TrendRequestWorkflow` instance. The workflow's `synthesizeStep` calls `TrendsExplorerAgent` — the single AutonomousAgent — with the query parameters as `TaskDef.instructions(...)` and the raw trend payload as a `TaskDef.attachment("trend-data.json", ...)`. The agent's `before-agent-response` guardrail (`ReportQualityGuardrail`) validates each candidate response. Once a report passes, the workflow writes `ReportRecorded` and runs `ReportEvaluationScorer` in `evalStep`. The score lands as `EvaluationScored`. `TrendReportView` projects every entity event into a read-model row; `TrendEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The on-decision evaluator (`ReportEvaluationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses on something:

1. The `TrendDataFetcher` subscription lag between `TrendQuerySubmitted` and `TrendDataReady` — sub-second in normal operation, since the fetcher reads from in-process seed data.
2. The `awaitDataStep` polling loop inside the workflow — polls `TrendRequestEntity` every 1 s up to its 15 s timeout, advancing as soon as `request.rawPayload().isPresent()` returns true.

The agent call itself is bounded by `synthesizeStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The notable paths:

- The happy path is `SUBMITTED → DATA_READY → SYNTHESIZING → REPORT_READY → EVALUATED`.
- Two failure transitions land in `FAILED`: a fetcher error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `SYNTHESIZING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state.
- There is no `ACTIONED` or `PUBLISHED` state. The report is informational; the analyst reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`TrendRequestEntity` is the source of truth. It emits six event types. `TrendReportView` projects every event into a row used by the UI. `TrendDataFetcher` subscribes to entity events to assemble the raw trend payload. `TrendRequestWorkflow` both reads (`getTrendRequest`) and writes (`markSynthesizing`, `recordReport`, `recordEvaluation`, `fail`) on the entity. The relationship between `TrendsExplorerAgent` and `TrendReport` is "returns" — the agent's task result is the report record.

## Quality signal flow

For any report that lands in the entity log, the request passed through:

1. **TrendDataFetcher** — the agent receives the seeded payload as an attachment, not as a raw parameter. The attachment boundary keeps prompt injection out of the parameter fields.
2. **TrendsExplorerAgent** — one model call, one structured output. Topics, ranks, and volume indices are drawn only from the attachment.
3. **before-agent-response guardrail** — empty rationales, duplicate ranks, and missing breakout summaries are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed report still gets a 1–5 score so the analyst knows which reports to inspect at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
