# Architecture — safety-plugins

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SafetyEndpoint` accepts a screening request and writes a `PayloadSubmitted` event onto `SafetyEntity`. The `PayloadSanitizer` Consumer subscribes, redacts PII, and writes the sanitized payload back via `attachSanitized`. The same Consumer then starts a `SafetyWorkflow` instance. The workflow's `screenStep` calls `SafetyAgent` — the single AutonomousAgent — with the safety rules as `TaskDef.instructions(...)` and the sanitized payload as a `TaskDef.attachment(...)`. Two guardrails wrap the agent call: `InputGuardrail` fires before the LLM call on the sanitized payload, and `OutputGuardrail` fires after the LLM returns a candidate decision. Once a decision passes `OutputGuardrail`, the workflow writes `DecisionRecorded` and runs `DecisionEvaluator` in `evalStep`. The score lands as `EvaluationScored`. `SafetyView` projects every entity event into a read-model row; `SafetyEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `InputGuardrail`, `OutputGuardrail`, and `DecisionEvaluator` are all deterministic classes that wrap or evaluate the one LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct pauses:

1. The `PayloadSanitizer` subscription lag between `PayloadSubmitted` and `PayloadSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `SafetyEntity` every 1 s up to its 15 s timeout, advancing as soon as `screening.sanitized().isPresent()` returns true.
3. The `InputGuardrail` check happens synchronously before the LLM call in `screenStep`. On a hard-block, the workflow short-circuits to the error step without any LLM call.

The agent call itself is bounded by `screenStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → SCREENING → DECISION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED` (rare — sanitizer is pure function), and a hard-block or agent error during `SCREENING`. The hard-block path is the more common FAILED path in adversarial deployments; the `ScreeningFailed.reason` carries the matching injection-pattern id.
- There is no `APPROVED` state. The safety decision is enforced inline by the calling system (which gates on `overallAction`); the blueprint records the decision and its quality score and stops.

## Entity model

`SafetyEntity` is the source of truth. It emits six event types. `SafetyView` projects every event into a row used by the UI. `PayloadSanitizer` subscribes to entity events to compute the sanitized form. `SafetyWorkflow` both reads (`getScreening`) and writes (`markScreening`, `recordDecision`, `recordEvaluation`, `fail`) on the entity. The relationship between `SafetyAgent` and `SafetyDecision` is "returns" — the agent's task result is the decision record.

## Defence-in-depth governance flow

For any decision that lands in the entity log, the payload passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **InputGuardrail (before-llm-call)** — known injection patterns and size violations are caught before any LLM credit is consumed.
3. **SafetyAgent** — one model call, one structured output.
4. **OutputGuardrail (after-llm-response)** — bad parses, missing findings, and out-of-enum actions are caught before the response leaves the agent loop.
5. **On-decision evaluator** — every well-formed decision still gets a 1–5 evidence score so operators know which decisions to audit.

Each layer is independent. Removing one of them opens an explicit gap the others do not silently cover. The two guardrails are on opposite sides of the LLM call and address different failure modes: `InputGuardrail` stops adversarial inputs; `OutputGuardrail` stops malformed outputs.
