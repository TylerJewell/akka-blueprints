# Architecture — time-series-forecasting (java)

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ForecastEndpoint` accepts a series submission and writes a `SeriesSubmitted` event onto `ForecastEntity`. The `SeriesValidator` Consumer subscribes, runs data-quality and stationarity checks, and writes the quality summary back via `attachValidated`. The same Consumer then starts a `ForecastWorkflow` instance. The workflow's `forecastStep` calls `ForecastingAgent` — the single AutonomousAgent — with the forecast configuration as `TaskDef.instructions(...)` and the validated CSV series as a `TaskDef.attachment("series.csv", ...)`. The agent's `before-agent-response` guardrail (`ForecastGuardrail`) validates each candidate response. Once a result passes, the workflow writes `ForecastCompleted` and runs `DriftEvaluator` in `driftEvalStep`. The drift score lands as `DriftEvaluated`. `ForecastView` projects every entity event into a read-model row; `ForecastEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The drift evaluator (`DriftEvaluator`) is a deterministic MAPE computation — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses:

1. The `SeriesValidator` subscription lag between `SeriesSubmitted` and `SeriesValidated` — sub-second in normal operation for an in-process series.
2. The `awaitValidatedStep` polling loop inside the workflow — polls `ForecastEntity` every 1 s up to its 15 s timeout, advancing as soon as `forecast.quality().isPresent()` returns true.

The agent call itself is bounded by `forecastStep`'s 60 s timeout. The `driftEvalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → VALIDATED → FORECASTING → FORECAST_COMPLETED → DRIFT_EVALUATED`.
- Two failure transitions land in `FAILED`: a validator error during `SUBMITTED` (bad data format or insufficient rows), and an agent error (or guardrail-exhaustion) during `FORECASTING`. A `FAILED` forecast's prior data is preserved on the entity — the UI shows the partial state and the failure reason.
- There is no `ACCEPTED` or `REJECTED` state. The forecast is advisory input to downstream models and human analysts. The blueprint deliberately stops at `DRIFT_EVALUATED`.

## Entity model

`ForecastEntity` is the source of truth. It emits six event types. `ForecastView` projects every event into a row used by the UI. `SeriesValidator` subscribes to entity events to compute the quality summary. `ForecastWorkflow` both reads (`getForecast`) and writes (`markForecasting`, `recordForecast`, `recordDriftEval`, `fail`) on the entity. The relationship between `ForecastingAgent` and `ForecastResult` is "returns" — the agent's task result is the forecast record.

## Defence-in-depth governance flow

For any forecast result that lands in the entity log, the series passed through:

1. **SeriesValidator** — the model only sees quality-checked, stationary-assessed data; malformed rows and insufficient lengths are rejected before any LLM call.
2. **ForecastingAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — interval inversions, horizon-count mismatches, and null point values are caught before the response leaves the agent loop.
4. **Drift-fairness-watch evaluator** — every well-formed forecast still gets a MAPE-backed drift status so the analyst knows which results to trust when the model's distribution may have shifted.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
