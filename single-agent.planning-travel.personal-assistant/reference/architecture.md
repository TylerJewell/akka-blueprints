# Architecture — personal-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `AssistantEndpoint` accepts a request and writes a `RequestSubmitted` event onto `AssistantEntity`. The `ContextSanitizer` Consumer subscribes, redacts PII from the serialized context snapshot, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts an `AssistantWorkflow` instance. The workflow's `actStep` calls `AssistantAgent` — the single AutonomousAgent — with the natural-language request as `TaskDef.instructions(...)` and the sanitized context snapshot as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`WriteGuardrail`) validates each proposed write before it executes. Once an action passes, the workflow writes `ActionRecorded` and runs `ActionScorer` in `evalStep`. The score lands as `EvaluationScored`. `AssistantView` projects every entity event into a read-model row; `AssistantEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`ActionScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system waits:

1. The `ContextSanitizer` subscription lag between `RequestSubmitted` and `ContextSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `AssistantEntity` every 1 s up to its 15 s timeout, advancing as soon as `state.sanitized().isPresent()` returns true.

The agent call itself is bounded by `actStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → ACTING → ACTION_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ACTING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state.
- There is no `CONFIRMED` or `APPLIED` terminal state beyond `EVALUATED`. The action is applied to the in-process state immediately when `ActionRecorded` lands; the blueprint stops at `EVALUATED` so the user can review the change set in the UI before taking further action in an external system.

## Entity model

`AssistantEntity` is the source of truth. It emits six event types. `AssistantView` projects every event into a row used by the UI. `ContextSanitizer` subscribes to entity events to compute the sanitized form. `AssistantWorkflow` both reads (`getState`) and writes (`markActing`, `recordAction`, `recordEvaluation`, `fail`) on the entity. The relationship between `AssistantAgent` and `AssistantAction` is "returns" — the agent's task result is the action record.

## Defence-in-depth governance flow

For any action that lands in the entity log, the context snapshot passed through:

1. **PII sanitizer** — the model never sees contact identifiers; the audit log retains the raw form.
2. **AssistantAgent** — one model call, one structured output.
3. **before-tool-call guardrail** — past dates, empty titles, out-of-enum action types, and cross-context writes are caught before any tool executes.
4. **On-decision evaluator** — every well-formed action still gets a 1–5 groundedness score so the user knows which actions to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
