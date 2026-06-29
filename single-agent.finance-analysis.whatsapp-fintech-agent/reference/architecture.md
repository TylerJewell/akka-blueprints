# Architecture — whatsapp-fintech-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `MessageEndpoint` accepts a raw inbound message and writes a `MessageReceived` event onto `MessageEntity`. The `MessageSanitizer` Consumer subscribes, redacts PII from the message text, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `MessageWorkflow` instance.

The workflow's `respondStep` calls `FintechQueryAgent` — the single AutonomousAgent — with the sanitized customer context and message text as `TaskDef.instructions(...)`. The agent's `before-tool-call` guardrail (`PaymentGuardrail`) intercepts every tool call before it fires. Once a valid `AgentResponse` returns, the workflow writes `ResponseReady` and enters `hitlStep`.

`HitlGate` evaluates whether a `PendingTransaction` is present and its amount exceeds the configured threshold. If so, the workflow emits `ApprovalRequested`, transitions the entity to `AWAITING_APPROVAL`, and suspends. An operator uses `POST /api/messages/{id}/approve` or `/reject` to resume the workflow. If no HITL is required, the workflow advances directly to `executeStep`.

`MessageView` projects every entity event into a read-model row; `MessageEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `PaymentGuardrail` and `HitlGate` are supporting classes with no LLM calls. This makes the blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the large-transfer HITL path (J3). Three distinct pause points exist:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop — polls `MessageEntity` every 1 s up to its 15 s timeout.
3. The `hitlStep` suspension — the workflow is indefinitely paused at `AWAITING_APPROVAL` until an operator command arrives. There is no polling in this step; the workflow resumes via a callback triggered by the `approve` or `reject` entity command.

The agent call is bounded by `respondStep`'s 60 s timeout. `executeStep` is deterministic and finishes in milliseconds.

## State machine

Eight states. Notable paths:

- **Happy path (balance query or small transfer):** `RECEIVED → SANITIZED → RESPONDING → RESPONSE_READY → EXECUTED`.
- **HITL approve path (large transfer):** `RECEIVED → SANITIZED → RESPONDING → RESPONSE_READY → AWAITING_APPROVAL → EXECUTED`.
- **HITL reject path:** `RECEIVED → SANITIZED → RESPONDING → RESPONSE_READY → AWAITING_APPROVAL → HITL_REJECTED`.
- **Failure paths:** a sanitizer error during `RECEIVED` and an agent error (or guardrail-exhaustion) during `RESPONDING` both transition to `FAILED`.

There is no `PENDING` or `QUEUED` state; the workflow resumes immediately when the operator acts. A `FAILED` message retains all partial state on the entity for audit.

## Entity model

`MessageEntity` is the source of truth. It emits nine event types. `MessageView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `MessageWorkflow` both reads (`getMessage`) and writes (`markResponding`, `recordResponse`, `requestApproval`, `markExecuted`, `fail`) on the entity. The relationship between `FintechQueryAgent` and `AgentResponse` is "returns" — the agent's task result is the response record, populated into the entity by `recordResponse`.

## Defence-in-depth governance flow

For any transaction that reaches `EXECUTED`, the inbound message passed through:

1. **PII sanitizer** — the model never sees raw identifiers; the audit log retains the raw form.
2. **FintechQueryAgent** — one model call, one structured output.
3. **before-tool-call guardrail** — invalid destination accounts, out-of-range amounts, and wrong-session-state tool calls are caught before any payment API fires.
4. **HITL gate** — above-threshold transfers require a named operator to approve, creating a durable approval record on the entity before execution.

Each layer is independent. Removing any one of them opens an explicit gap the others do not silently cover.
