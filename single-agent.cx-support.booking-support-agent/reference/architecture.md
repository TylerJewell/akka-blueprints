# Architecture — booking-support-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SupportEndpoint` accepts a customer message and writes a `CustomerMessageReceived` event onto `BookingSessionEntity`. The `PiiSanitizer` Consumer subscribes, redacts PII from the raw message text, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `BookingSessionWorkflow` instance keyed on the turn id. The workflow's `agentTurnStep` calls `BookingSupportAgent` — the single AutonomousAgent — with the sanitized message, session history, and customer id as `TaskDef.instructions(...)`. The agent calls booking tools (`lookUpBooking`, `modifyBooking`, `cancelBooking`, `issueRefundCredit`); each destructive call is intercepted by `ToolCallGuardrail` before it executes. Read-only lookups are resolved against `BookingStore`. Once the agent returns an `AgentReply`, the workflow writes `AgentTurnCompleted` and closes. `BookingSessionView` projects every entity event into a read-model row; `SupportEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ToolCallGuardrail` is a synchronous rule-based interceptor — not an LLM. `JudgeAssertions` run only in CI, not at runtime. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model during production operation.

## Interaction sequence

The sequence traces the J1 happy path (booking lookup). Two distinct moments where the system pauses:

1. The `PiiSanitizer` subscription lag between `CustomerMessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `BookingSessionEntity` every 1 s up to its 15 s timeout, advancing as soon as the current turn's sanitized field is present.

The agent call is bounded by `agentTurnStep`'s 60 s timeout. Within that call, the agent may make multiple tool calls; each destructive call passes through `ToolCallGuardrail` synchronously before `BookingStore` executes it.

## State machine

The diagram shows the per-turn lifecycle inside `BookingSessionEntity`. Four states:

- The happy path is `RECEIVED → SANITIZED → AGENT_REPLIED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (or guardrail exhaustion) during `SANITIZED`. A failed turn's prior data is preserved on the entity — the audit log retains the raw message and partial tool-call history.
- Sessions themselves hold a `SessionStatus` (OPEN / CLOSED / FAILED) but the turn is the granular lifecycle unit the UI tracks per message.

## Entity model

`BookingSessionEntity` is the source of truth for conversation history, tool-call outcomes, and booking mutations. It emits six event types. `BookingSessionView` projects every event into a row used by the UI. `PiiSanitizer` subscribes to entity events to sanitize incoming messages. `BookingSessionWorkflow` both reads (`getSession`) and writes (`attachSanitized`, `recordAgentTurn`, `failTurn`) on the entity. `BookingSupportAgent` is related to `AgentReply` as "returns" — the task result carries the structured reply.

## Defence-in-depth governance flow

For any agent reply that lands in the entity log, the session passed through:

1. **PII sanitizer** — the model never sees payment-card numbers, government ids, phone numbers, or emails; the audit log retains the raw form.
2. **BookingSupportAgent** — one model call per turn, producing a structured `AgentReply`.
3. **before-tool-call guardrail** — ownership violations, fare-rule violations, and duplicate mutations are caught before tool execution. The agent's `replyText` to the customer reflects the actual outcome.
4. **CI judge gate** — LLM-judge assertions in `JudgeAssertions.java` catch hallucinated booking details before any deployment target sees the build.

Each layer addresses a different failure mode. The PII sanitizer prevents data leakage. The guardrail prevents cross-customer mutations. The CI gate prevents semantic errors in agent replies. None of them silently covers the gap left by removing another.
