# Architecture — currency-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ConversionEndpoint` accepts a conversion request and writes a `ConversionRequested` event onto `ConversionEntity`. The `RateFetcher` Consumer subscribes, loads the matching rate snapshot from the bundled seed file, and writes it back via `attachRate`. The same Consumer then starts a `ConversionWorkflow` instance. The workflow's `convertStep` calls `ExchangeRateAgent` — the single AutonomousAgent — with the conversion parameters as `TaskDef.instructions(...)` and the rate snapshot JSON as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ResultGuardrail`) validates each candidate response. Once a result passes, the workflow writes `ResultRecorded` and runs `FreshnessScorer` in `evalStep`. The freshness score lands as `FreshnessScored`. `ConversionView` projects every entity event into a read-model row; `ConversionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The freshness evaluator (`FreshnessScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses occur:

1. The `RateFetcher` subscription lag between `ConversionRequested` and `RateAttached` — sub-second because the rate data is loaded from an in-process file, not a network call.
2. The `awaitRateStep` polling loop inside the workflow — polls `ConversionEntity` every 1 s up to its 15 s timeout, advancing as soon as `conversion.rateSnapshot().isPresent()` returns true.

The agent call is bounded by `convertStep`'s 60 s timeout. The `evalStep` is synchronous and deterministic — no external service, no LLM call, completes in milliseconds.

## State machine

Six states. The interesting paths:

- The happy path is `REQUESTED → RATE_ATTACHED → CONVERTING → RESULT_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a fetcher error during `REQUESTED`, and an agent error (or guardrail-exhaustion) during `CONVERTING`. A `FAILED` conversion's prior state is preserved on the entity — the UI shows the partial state for diagnosis.
- There is no `EXECUTED` or `COMMITTED` state. The conversion result is informational; the user reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`ConversionEntity` is the source of truth. It emits six event types. `ConversionView` projects every event into a row used by the UI. `RateFetcher` subscribes to entity events to load and attach the rate snapshot. `ConversionWorkflow` both reads (`getConversion`) and writes (`markConverting`, `recordResult`, `recordEvaluation`, `fail`) on the entity. The relationship between `ExchangeRateAgent` and `ConversionResult` is "returns" — the agent's task result is the conversion result record.

## Baseline governance flow

For any result that lands in the entity log, the conversion request passed through:

1. **RateFetcher** — the model receives the rate as a structured attachment, not free-form text; the source and timestamp are explicit.
2. **ExchangeRateAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — null amounts, blank currencies, and missing timestamps are caught before the response leaves the agent loop.
4. **FreshnessScorer** — every result carries a 1–5 staleness score so the user knows at a glance whether the rate data backing the conversion was fresh.

This is the baseline tier — no blocking policy controls. A deployer can add controls (rate-staleness threshold, precision validator, PII check on the user-supplied `requestedBy` field) by editing `eval-matrix.yaml` and extending the generated components.
