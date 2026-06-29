# Architecture — docreview

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ReviewEndpoint` accepts a submission and writes a `DocumentSubmitted` event onto `ReviewEntity`. The `DocumentSanitizer` Consumer subscribes, redacts PII, and writes the sanitized document back via `attachSanitized`. The same Consumer then starts a `ReviewWorkflow` instance. The workflow's `reviewStep` calls `DocumentReviewerAgent` — the single AutonomousAgent — with the review instructions as `TaskDef.instructions(...)` and the sanitized document as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`VerdictGuardrail`) validates each candidate response. Once a verdict passes, the workflow writes `VerdictRecorded` and runs `EvaluationScorer` in `evalStep`. The score lands as `EvaluationScored`. `ReviewView` projects every entity event into a read-model row; `ReviewEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`EvaluationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses on something:

1. The `DocumentSanitizer` subscription lag between `DocumentSubmitted` and `DocumentSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ReviewEntity` every 1 s up to its 15 s timeout, advancing as soon as `review.sanitized().isPresent()` returns true.

The agent call itself is bounded by `reviewStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → REVIEWING → VERDICT_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `REVIEWING`. A `FAILED` review's prior data is preserved on the entity — the UI shows the partial state for the reviewer.
- There is no `APPROVED` or `REJECTED` state. The verdict is advisory; the human reads it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`ReviewEntity` is the source of truth. It emits six event types. `ReviewView` projects every event into a row used by the UI. `DocumentSanitizer` subscribes to entity events to compute the sanitized form. `ReviewWorkflow` both reads (`getReview`) and writes (`markReviewing`, `recordVerdict`, `recordEvaluation`, `fail`) on the entity. The relationship between `DocumentReviewerAgent` and `ReviewVerdict` is "returns" — the agent's task result is the verdict record.

## Defence-in-depth governance flow

For any verdict that lands in the entity log, the document passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **DocumentReviewerAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — bad parses, missing findings, and out-of-enum severities are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed verdict still gets a 1–5 evidence score so the human knows which verdicts to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
