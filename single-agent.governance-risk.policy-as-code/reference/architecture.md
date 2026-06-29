# Architecture — policy-as-code

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PolicyEndpoint` accepts a change submission and writes a `ChangeSubmitted` event onto `ChangeRequestEntity`. The `ChangeValidator` Consumer subscribes, normalizes the payload, and writes the validated form back via `attachValidated`. The same Consumer then starts an `EvaluationWorkflow` instance. The workflow's `enforceStep` calls `PolicyEnforcementAgent` — the single AutonomousAgent — with the policy rules as `TaskDef.instructions(...)` and the normalized change payload as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`ToolCallGuardrail`) intercepts any tool call before it executes and validates it against an allowlist. Once a decision passes, the workflow writes `DecisionRecorded` and runs `GateEvaluator` in `gateStep`. The gate result lands as `GateEvaluated`. `PolicyView` projects every entity event into a read-model row; `PolicyEndpoint` serves the read model to the UI over REST, SSE, and the dedicated gate endpoint.

The graph deliberately has no second agent. The CI gate (`GateEvaluator`) is a deterministic rule-based component — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system pauses:

1. The `ChangeValidator` subscription lag between `ChangeSubmitted` and `ChangeValidated` — sub-second in normal operation.
2. The `awaitValidatedStep` polling loop inside the workflow — polls `ChangeRequestEntity` every 1 s up to its 15 s timeout, advancing as soon as `change.validated().isPresent()` returns true.

The agent call itself is bounded by `enforceStep`'s 60 s timeout. The `gateStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → VALIDATED → ENFORCING → DECISION_RECORDED → GATED`.
- Two failure transitions land in `FAILED`: a validator error during `SUBMITTED`, and an agent error (or timeout) during `ENFORCING`. A `FAILED` evaluation's prior data is preserved on the entity — the UI shows the partial state for the operator.
- There is no `DEPLOYED` or `REJECTED` state. The gate result is consumed by the CI system; the blueprint stops at `GATED`.

## Entity model

`ChangeRequestEntity` is the source of truth. It emits six event types. `PolicyView` projects every event into a row used by the UI. `ChangeValidator` subscribes to entity events to produce the validated form. `EvaluationWorkflow` both reads (`getChange`) and writes (`markEnforcing`, `recordDecision`, `recordGate`, `fail`) on the entity. The relationship between `PolicyEnforcementAgent` and `PolicyDecision` is "returns" — the agent's task result is the decision record.

## Defence-in-depth governance flow

For any decision that lands in the entity log, the change passed through:

1. **ChangeValidator** — the payload is structurally verified, comment-stripped, and normalized before any LLM call. The raw payload is preserved on the entity for audit.
2. **PolicyEnforcementAgent** — one model call, one structured output mapping every breached rule to a typed Violation.
3. **before-tool-call guardrail** — any attempt to call a disallowed external tool is blocked before it executes; the agent must reason from the attached payload and the permitted tools only.
4. **CI gate** — every well-formed decision still goes through a deterministic gate check; a DENY or CRITICAL violation produces a BLOCKED gate that the CI pipeline must honour.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
