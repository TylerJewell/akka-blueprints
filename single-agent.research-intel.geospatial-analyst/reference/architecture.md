# Architecture — earth-engine-geospatial

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `AnalysisEndpoint` accepts a spatial query and writes a `QuerySubmitted` event onto `AnalysisEntity`. The `DatasetFetcher` Consumer subscribes, validates the bounding-box area, assembles a `DatasetSnapshot` from the seeded indicator data, and writes the snapshot back via `attachDataset`. The same Consumer then starts an `AnalysisWorkflow` instance. The workflow's `analysisStep` calls `GeospatialAnalystAgent` — the single AutonomousAgent — with indicator definitions as `TaskDef.instructions(...)` and the assembled snapshot as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ReportGuardrail`) validates each candidate response. Once a report passes, the workflow writes `ReportRecorded` and runs `EvaluationScorer` in `evalStep`. The score lands as `EvaluationScored`. `AnalysisView` projects every entity event into a read-model row; `AnalysisEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The on-decision evaluator (`EvaluationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses occur before the agent call:

1. The `DatasetFetcher` subscription lag between `QuerySubmitted` and `DatasetReady` — sub-second in normal operation because the dataset is assembled from in-process seeded fixtures.
2. The `awaitDatasetStep` polling loop inside the workflow — polls `AnalysisEntity` every 1 s up to its 15 s timeout, advancing as soon as `analysis.dataset().isPresent()` returns true.

The agent call itself is bounded by `analysisStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → DATASET_READY → ANALYSING → REPORT_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a bounds-check rejection during `SUBMITTED` (the entity never leaves that state), and an agent error or guardrail-exhaustion during `ANALYSING`. A `FAILED` analysis's prior data is preserved on the entity — the UI shows the partial state for the researcher along with the failure reason.
- There is no `APPROVED` or `REJECTED` state. The report is advisory; the researcher reads it and decides outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`AnalysisEntity` is the source of truth. It emits six event types. `AnalysisView` projects every event into a row used by the UI. `DatasetFetcher` subscribes to entity events to compute the assembled snapshot or to raise a bounds failure. `AnalysisWorkflow` both reads (`getAnalysis`) and writes (`markAnalysing`, `recordReport`, `recordEvaluation`, `fail`) on the entity. The relationship between `GeospatialAnalystAgent` and `AnalysisReport` is "returns" — the agent's task result is the report record.

## Defence-in-depth governance flow

For any report that lands in the entity log, the query passed through:

1. **Coordinate-bounds sanitizer** — the bounding box area is verified before dataset assembly; oversized queries fail fast with a clear reason.
2. **GeospatialAnalystAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, missing findings, and out-of-range confidence values are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed report still gets a 1–5 evidence score so the researcher knows which reports to trust at a glance.

Each step is independent. Removing one opens an explicit gap the others do not silently cover.
