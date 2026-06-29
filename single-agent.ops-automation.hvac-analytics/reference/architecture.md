# Architecture — hvac-analytics

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `QueryEndpoint` accepts a question submission and writes a `QueryInitiated` event onto `QueryEntity`. The `TelemetryStore` Consumer subscribes, assembles the relevant telemetry snapshot from the in-process data store (filtering to the requested zone scope and stripping equipment serial numbers), and writes the snapshot back via `attachSnapshot`. The same Consumer then starts a `QueryWorkflow` instance. The workflow's `analysisStep` calls `HvacAnalyticsAgent` — the single AutonomousAgent — with the question as `TaskDef.instructions(...)` and the telemetry snapshot as a `TaskDef.attachment(...)`. Once an answer returns, the workflow writes `AnswerRecorded` and runs `AnswerQualityScorer` in `evalStep`. The score lands as `EvaluationScored`. `QueryView` projects every entity event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent and no guardrail on the agent's response. The quality evaluator (`AnswerQualityScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct pauses:

1. The `TelemetryStore` subscription lag between `QueryInitiated` and `SnapshotAttached` — sub-second in normal operation because the data store is in-process.
2. The `awaitSnapshotStep` polling loop inside the workflow — polls `QueryEntity` every 1 s up to its 15 s timeout, advancing as soon as `query.snapshot().isPresent()` returns true.

The agent call itself is bounded by `analysisStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The key paths:

- The happy path is `INITIATED → SNAPSHOT_READY → ANALYSING → ANSWER_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a store error during `INITIATED`, and an agent error (or max-iteration exhaustion) during `ANALYSING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state for the engineer.
- There is no `ACTIONED` or `RESOLVED` state. The answer is advisory; the building engineer reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits six event types. `QueryView` projects every event into a row used by the UI. `TelemetryStore` subscribes to entity events to assemble the snapshot. `QueryWorkflow` both reads (`getQuery`) and writes (`markAnalysing`, `recordAnswer`, `recordEvaluation`, `fail`) on the entity. The relationship between `HvacAnalyticsAgent` and `AnalyticsAnswer` is "returns" — the agent's task result is the answer record.

## Governance flow

For every answer that lands in the entity log:

1. **Equipment identifier stripping** — `TelemetryStore` removes serial-number fields from every snapshot before the agent ever sees the data. Zone labels are the only identifiers the model works with.
2. **HvacAnalyticsAgent** — one model call, one structured output with evidence data points.
3. **Answer quality evaluator** — `AnswerQualityScorer` scores every well-formed answer for evidence quality (non-empty data points, actionable recommendation, plausible trend classification). The score is stored and displayed so the engineer knows at a glance which answers to scrutinise before acting.

The single control (E1) is non-blocking by design: the answer lands in the entity log regardless of score, and the evaluator's role is to surface quality gaps rather than gate operations. This reflects the advisory nature of the system — a building engineer makes the call.
