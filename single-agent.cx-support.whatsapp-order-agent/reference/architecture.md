# Architecture — whatsapp-order-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `OrderEndpoint` accepts a session start or a new turn, writes the appropriate command onto `SessionEntity`, and triggers `OrderWorkflow`. The workflow's `agentStep` calls `OrderAgent` — the single AutonomousAgent — with the conversation history and current message as `TaskDef.instructions(...)` and the product catalog as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`ToolGuardrail`) validates each tool invocation before it executes. A passing tool call reaches `OrderEntity`; a blocked call returns a structured rejection to the agent loop for re-planning.

Once the agent produces an `AgentReply`, the workflow checks `hitlRequired`. If false, it proceeds directly to `completeStep` where `TurnScorer` runs a deterministic quality check. If true, the workflow enters `hitlStep`, which emits `ApprovalRequested` and waits up to 300 s for an operator to act via the `POST /api/hitl/*` endpoints. On approval the workflow resumes and completes the order; on rejection the session closes with a failure reason.

After each turn, `ConversationSanitizer` subscribes to `TurnCompleted` events, redacts PII, and writes the sanitized form back via `attachSanitizedTurn`. `SessionView` projects both `SessionEntity` and `OrderEntity` events into a combined read-model row; `OrderEndpoint` serves that row to the UI over REST and SSE.

The graph deliberately has no second agent. `TurnScorer` is a deterministic rule-based component — not an LLM. Exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy-path single-item order (J1). Two key moments stand out:

1. The `ConversationSanitizer` subscription lag between `TurnCompleted` and `ConversationSanitized` — sub-second in normal operation, but the sanitized form appears asynchronously; the UI poll waits for the `ConversationSanitized` SSE event before showing PII chips.
2. The `hitlStep` polling loop — polls `SessionEntity` every 2 s up to 300 s. This step is only entered when `hitlRequired == true`, which the agent sets when the pending order total exceeds `hitl.order.threshold`.

The agent call is bounded by `agentStep`'s 60 s timeout. `completeStep` runs `TurnScorer` synchronously and finishes in milliseconds.

## State machine

Six states on `SessionEntity`. The interesting paths:

- The happy path per turn is `IDLE → ACTIVE → COMPLETING → CLOSED`. For a multi-turn session the entity cycles through `ACTIVE → COMPLETING → ACTIVE` until the customer or operator closes the session.
- The HITL path detours: `ACTIVE → AWAITING_APPROVAL → COMPLETING → CLOSED` on approval, or `ACTIVE → AWAITING_APPROVAL → FAILED` on rejection or timeout.
- Two failure transitions land in `FAILED`: a guardrail-exhaustion error during `ACTIVE`, and an operator rejection during `AWAITING_APPROVAL`. Failed session data is preserved — the UI shows the partial conversation history for the operator.
- There is no automatic `CLOSED` transition from the agent side; the workflow drives `SessionCompleted` explicitly after `completeStep` succeeds.

## Entity model

`SessionEntity` is the source of truth for the conversation. `OrderEntity` is the source of truth for each individual order. `SessionView` projects both into a combined read model used by the UI. `ConversationSanitizer` subscribes to session events to compute the sanitized turn. `OrderWorkflow` both reads (`getSession`) and writes (`recordTurnCompleted`, `requestApproval`, `grantApproval`, `rejectApproval`, `complete`, `fail`) on the session entity, and writes draft/confirm/cancel commands on `OrderEntity`.

## Defence-in-depth governance flow

For any order that lands confirmed in the entity log, the request passed through:

1. **ToolGuardrail** — every write tool call validated against catalog data and ownership rules before execution. Bad calls are blocked and the agent re-plans.
2. **OrderAgent** — one model call, one structured output per turn. Product catalog delivered as an attachment so the agent has authoritative stock and pricing data.
3. **PII sanitizer** — phone numbers, addresses, and payment tokens stripped from the stored conversation context after each turn. The audit trail retains the raw text; the projected view does not.
4. **HITL gate** — orders above the configured threshold require explicit operator approval before `OrderEntity` receives the `OrderConfirmed` event.

Each layer is independent. Removing any one of them opens a gap the others do not silently cover.
