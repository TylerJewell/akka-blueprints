# Architecture — restaurant-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SessionEndpoint` accepts a customer message, records it on `SessionEntity`, and triggers `SessionWorkflow`. The workflow's `agentStep` calls `RestaurantAssistantAgent` — the single AutonomousAgent — with the formatted session context and incoming message as `TaskDef.instructions(...)`.

Two guardrail classes sit on either side of the agent's output:

- `ResponseGuardrail` (before-agent-response) validates the candidate reply against the in-process `MenuCatalog` before any text is forwarded to the workflow or displayed to the customer.
- `ToolCallGuardrail` (before-tool-call) validates reservation or order arguments before the workflow commits a write.

Once the agent's response passes both guardrails, the workflow dispatches: if no tool was called, `recordAgentReply` appends the assistant message to the entity; if a tool was called, `commitStep` calls `confirmReservation` or `commitOrder` on the entity. `SessionView` projects every entity event into a read-model row. `SessionEndpoint` serves the read model to the UI over REST and SSE.

There is no second agent. `MenuCatalog` is a deterministic in-process lookup loaded from a JSONL file — not an LLM call. That is what keeps this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the reservation path (J2). Note the two guardrail intercepts:

1. `ResponseGuardrail` fires on the agent's candidate `AssistantResponse` before the workflow sees it. In the happy path, it passes through immediately. On a catalog-inaccuracy, it rejects and the agent retries.
2. `ToolCallGuardrail` fires on the `makeReservation` arguments before the workflow's `commitStep` is invoked. A past date or out-of-range party size is rejected here; the agent self-corrects on retry.

The `agentStep` is bounded by a 60 s timeout to accommodate LLM latency plus up to 3 guardrail retry iterations. The `commitStep` is bounded at 10 s — it is a single entity command with no external I/O.

## State machine

Seven states. The interesting paths:

- The happy menu-query path is `OPEN → ACTIVE → ACTIVE` (the state stays `ACTIVE` for pure question-answering turns; each `AgentReplied` event re-enters the same state).
- The reservation path is `OPEN → ACTIVE → RESERVATION_HELD`. After the reservation is held, the session stays open for follow-up messages and can still transition to `ORDER_PLACED`.
- The order path is `ACTIVE → ORDER_PLACED → CLOSED`.
- A `SessionFailed` event (guardrail exhaustion or workflow error) transitions to `FAILED`. The partial message history is preserved on the entity.

There is no `APPROVED` or `CONFIRMED` by an external authority. The system takes reservation and order actions on the customer's behalf within the configured guardrail limits; a human staff member reviews the resulting entity state via the operations dashboard.

## Entity model

`SessionEntity` is the source of truth for the session lifecycle, message history, and committed reservation/order. It emits seven event types. `SessionView` projects every event into a row used by the UI's live list. `SessionWorkflow` both reads (`getSession`) and writes (via commands) on the entity. The relationship between `RestaurantAssistantAgent` and `AssistantResponse` is "returns" — the agent's task result drives the workflow's next step.

## Dual-guardrail governance flow

For any write that lands in the entity log, the customer's request passed through:

1. **RestaurantAssistantAgent** — one model call producing a structured `AssistantResponse`.
2. **ResponseGuardrail** — the candidate reply is checked for menu accuracy before it is forwarded; inaccuracies trigger retries inside the agent loop.
3. **ToolCallGuardrail** — if a tool was called, the arguments are validated against business rules before the workflow commits the write; invalid arguments trigger retries inside the agent loop.

Each guardrail is an independent check. Removing one opens a specific gap the other does not cover: removing G1 allows inaccurate text to reach customers; removing G2 allows invalid writes to land in the entity.
