# Architecture — youtube-analyst

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `AnalysisEndpoint` accepts a request and writes an `AnalysisRequested` event onto `ChannelEntity`. The `MetricsFetcher` Consumer subscribes, loads the matching channel fixture from seeded JSON, computes the period-scoped deltas and video rankings, and writes the result back via `attachMetrics`. The same Consumer then starts an `AnalysisWorkflow` instance. The workflow's `analyseStep` calls `ChannelAnalystAgent` — the single AutonomousAgent — with the focus area and period as `TaskDef.instructions(...)` and the serialized `ChannelMetrics` as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ReportGuardrail`) validates each candidate response. Once a report passes, the workflow writes `ReportRecorded` and runs `AnalysisEvaluator` in `evalStep`. The score lands as `EvaluationScored`. `AnalysisView` projects every entity event into a read-model row; `AnalysisEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The on-decision evaluator (`AnalysisEvaluator`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses:

1. The `MetricsFetcher` subscription lag between `AnalysisRequested` and `MetricsFetched` — sub-second in normal operation, as the fixture data is in-process.
2. The `awaitMetricsStep` polling loop inside the workflow — polls `ChannelEntity` every 1 s up to its 15 s timeout, advancing as soon as `analysis.metrics().isPresent()` returns true.

The agent call itself is bounded by `analyseStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The key paths:

- The happy path is `REQUESTED → METRICS_FETCHED → ANALYSING → REPORT_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a fetcher error during `REQUESTED`, and an agent error (or guardrail-exhaustion) during `ANALYSING`. A `FAILED` analysis retains its partial data on the entity — the UI shows the metrics that were fetched and the stage at which the failure occurred.
- There is no `APPROVED` or `ACTIONED` state. The report is advisory; the strategist reads it and decides outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`ChannelEntity` is the source of truth. It emits six event types. `AnalysisView` projects every event into a row used by the UI. `MetricsFetcher` subscribes to entity events to compute and attach the metrics. `AnalysisWorkflow` both reads (`getAnalysis`) and writes (`markAnalysing`, `recordReport`, `recordEvaluation`, `fail`) on the entity. The relationship between `ChannelAnalystAgent` and `ChannelReport` is "returns" — the agent's task result is the report record.

## Governance flow

For any report that lands in the entity log, the request passed through:

1. **MetricsFetcher** — the model receives structured JSON metrics, not raw API payloads, so any data-shape issues are caught before the LLM call.
2. **ChannelAnalystAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — reports referencing invented videoIds or non-existent metric fields are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed report still gets a 1–5 completeness score so the strategist knows which reports are analytically solid.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
