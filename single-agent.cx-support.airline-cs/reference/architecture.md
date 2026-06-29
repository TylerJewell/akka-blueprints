# Architecture — airline-cs

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call in a ReAct loop. `ServiceEndpoint` accepts a customer service request and writes a `RequestReceived` event onto `RequestEntity`. The `MessageSanitizer` Consumer subscribes, redacts PII from the customer message, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `ServiceWorkflow` instance.

The workflow's `serveStep` calls `CustomerServiceAgent` — the single AutonomousAgent — with the sanitized message and booking context as `TaskDef.instructions(...)`. The agent reasons over four tools: `searchBooking`, `requestConfirmation`, `modifyReservation`, and `cancelReservation`. Every call to `modifyReservation` or `cancelReservation` passes through `ModificationGuardrail`, the `before-tool-call` hook, which blocks the call unless a `confirmationToken` is present in the arguments.

When the agent calls `requestConfirmation`, the workflow transitions to `hitlStep` and the entity records `ConfirmationRequested`. Execution pauses until the customer POSTs a confirm or cancel token. On confirm, the workflow emits `ConfirmationReceived` and passes the token back into the running agent task; the next `modifyReservation` call carries the token and the guardrail accepts it. On cancel, the entity transitions to `CANCELLED`.

Once the agent returns a `ServiceOutcome`, the workflow writes `RequestCompleted`. `RequestView` projects every entity event into a read-model row. `ServiceEndpoint` serves the read model over REST and SSE.

The graph has no second agent. `BookingService` is a plain Java class. `ModificationGuardrail` is a guardrail hook class. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the seat-change happy path (J1), which includes the HITL pause. There are three distinct moments where the system waits:

1. The `MessageSanitizer` subscription lag between `RequestReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop — polls `RequestEntity` every 1 s up to its 15 s timeout.
3. The `hitlStep` suspension — the workflow is fully paused waiting for the customer's confirmation POST. This can take seconds to minutes; the hitlStep timeout is 3600 s.

The agent's ReAct loop itself is bounded by `serveStep`'s 120 s timeout. For complaint-only flows, the hitlStep is never entered.

## State machine

Seven states. The interesting paths:

- The happy modification path is `RECEIVED → SANITIZED → PROCESSING → AWAITING_CONFIRMATION → PROCESSING → COMPLETED`. The workflow re-enters `PROCESSING` after the customer confirms, because the agent task resumes and may call additional tools.
- The direct resolution path (search or complaint only) is `RECEIVED → SANITIZED → PROCESSING → COMPLETED` — no HITL pause.
- The cancellation path is `RECEIVED → SANITIZED → PROCESSING → AWAITING_CONFIRMATION → CANCELLED` when the customer posts `action=cancel`.
- Three failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`; an agent or tool error during `PROCESSING`; a hitlStep timeout during `AWAITING_CONFIRMATION`.

There is no `APPROVED` state separate from `COMPLETED` — the customer's confirmation is the approval, and the completed outcome records it.

## Entity model

`RequestEntity` is the source of truth. It emits eight event types. `RequestView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized message. `ServiceWorkflow` both reads (`getRequest`) and writes (`markProcessing`, `requestConfirmation`, `receiveConfirmation`, `complete`, `cancel`, `fail`) on the entity. The relationship between `CustomerServiceAgent` and `ServiceOutcome` is "returns" — the agent's task result is the outcome record.

## Governance flow

For any reservation write that executes in the system, the customer's request passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **CustomerServiceAgent ReAct loop** — the agent reasons over tools, one model call per iteration.
3. **before-tool-call guardrail** — blocks `modifyReservation` and `cancelReservation` without a confirmation token, forcing the agent through the confirmation path.
4. **Application HITL gate** — the workflow pauses and waits for an explicit customer confirmation POST before the token is issued and the destructive write can proceed.

Each step is independent. The guardrail and the HITL gate address different failure modes: the guardrail handles the agent attempting a shortcut; the HITL gate handles the scenario where the agent called `requestConfirmation` but the customer has not yet responded.
