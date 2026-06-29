# Architecture — antom-payment

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PaymentEndpoint` accepts a payment instruction and writes a `PaymentRequested` event onto `PaymentEntity`, then starts a `PaymentWorkflow` instance. The workflow's `authStep` runs `AuthorizationGuardrail` in-process against the instruction's currency, method, recipient, and operator identity. On pass it writes `PaymentAuthorized`; on fail it writes `PaymentRejected` and the workflow ends.

If authorized, `hitlGateStep` compares the amount to the high-value threshold. Above-threshold payments emit `ApprovalRequired`, and the workflow pauses. `PaymentEndpoint.approve` / `.deny` resume or terminate it. Sub-threshold and approved payments advance to `executeStep`, which calls `PaymentActionAgent` — the single AutonomousAgent. The agent selects an Antom tool (`initiatePayment`, `queryPaymentStatus`, or `initiateRefund`); each tool call passes through the `before-tool-call` hook of the same `AuthorizationGuardrail`, providing a second enforcement layer inside the agent loop. The simulated Antom calls are dispatched to `AntomApiSimulator`. The agent returns a `PaymentResult`, which the workflow writes back via `PaymentEntity.settle` or `.query`.

In parallel, `FraudSignalConsumer` subscribes to all `PaymentEntity` events. On `FraudSignalDetected` it calls `PaymentEntity.halt(signalType)` immediately, regardless of which workflow step is running.

`PaymentView` projects every entity event into a read-model row; `PaymentEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `AuthorizationGuardrail` and `AntomApiSimulator` are pure Java helpers — neither makes an LLM call. That is what makes this a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1): a sub-threshold payment that passes the authorization guardrail, skips the HITL gate, is executed by the agent, and lands in `SETTLED`. Two governance enforcement points appear in the sequence:

1. `authStep` runs before the agent starts — an in-process call to `AuthorizationGuardrail` that gates the entire workflow.
2. The `before-tool-call` hook fires when the agent prepares its tool invocation — a second, independent check inside the agent loop at the moment the tool call is about to execute.

If the payment were above-threshold, the sequence would pause between `authStep` and `executeStep` until an operator's approve/deny arrives via `PaymentEndpoint`.

## State machine

Nine states plus two terminal error states. Key paths:

- **Happy path (sub-threshold):** `REQUESTED → AUTHORIZED → EXECUTING → SETTLED`.
- **Happy path (high-value):** `REQUESTED → AUTHORIZED → AWAITING_APPROVAL → EXECUTING → SETTLED`.
- **Guardrail rejection:** `REQUESTED → REJECTED` (terminal).
- **Operator deny:** `AWAITING_APPROVAL → REJECTED` (terminal).
- **Fraud halt:** `EXECUTING → HALTED` (terminal). This transition can occur from any executing state when `FraudSignalDetected` fires.
- **Agent failure:** `EXECUTING → FAILED` (terminal).
- `QUERIED` is a terminal state for QUERY-type instructions where the Antom status is `PENDING` or `PROCESSING` — the record is complete, and a follow-up QUERY instruction would be a separate payment entity.

There is no `APPROVED` or `COMPLETED` state as a separate lifecycle stage. `SETTLED` is the terminal success state for both INITIATE and REFUND instructions.

## Entity model

`PaymentEntity` is the source of truth. It emits twelve event types covering the full lifecycle including the conditional HITL branch and the unconditional fraud halt. `PaymentView` projects every event into a row used by the UI. `FraudSignalConsumer` subscribes to entity events to fire the halt. `PaymentWorkflow` both reads (`getPayment`) and writes (`authorize`, `requireApproval`, `startExecution`, `settle`, `query`, `halt`, `fail`) on the entity. The relationship between `PaymentActionAgent` and `PaymentResult` is "returns" — the agent's task result is the payment result record.

## Governance flow

For any instruction that proceeds through execution, it passed through:

1. **AuthorizationGuardrail (authStep)** — currency, method, recipient, and operator identity verified before the workflow advances.
2. **HITL gate (hitlGateStep)** — amounts above the threshold require a named operator's explicit `approve` before the agent starts.
3. **AuthorizationGuardrail (before-tool-call)** — same four checks run again inside the agent loop at tool-invocation time, catching any instruction drift between authorization and execution.
4. **FraudSignalConsumer (halt)** — unconditional stop on any fraud signal, wired as a separate Consumer so it fires independently of the workflow's own progress.

Each enforcement point is independent. The before-tool-call hook would catch a case where the agent chose a tool the authStep did not anticipate. The halt fires even if the HITL gate has already passed. Removing any one of them opens an explicit gap the others do not silently cover.
