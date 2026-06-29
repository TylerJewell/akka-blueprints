# Architecture — games-sales-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SessionEndpoint` accepts a new session creation and turn submissions. On a POST turn, it triggers `SessionWorkflow.assistStep`. The workflow calls `SalesAssistantAgent` — the single AutonomousAgent — passing the shopper's question as `TaskDef.instructions(...)` and the serialised `ShopperContext` (order history, previous turns) as a `TaskDef.attachment("context.json", ...)`. The agent's `before-agent-response` guardrail (`ResponseGuardrail`) validates each candidate response against the in-process `CatalogIndex`. Once a response passes, the workflow records `TurnAnswered` on `SessionEntity`. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model over REST and SSE.

`CatalogConsumer` is a separate Consumer that subscribes to `CatalogUpdated` events and hydrates the `CatalogIndex`. The index is the guardrail's single source of truth for title-existence checks. There is no database outside the Akka runtime.

The graph deliberately has no second agent. The `ResponseGuardrail` is a rule-based Java class — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system processes asynchronously:

1. The `CatalogConsumer` subscription runs at startup to load the seeded catalog — this finishes before the first shopper session begins.
2. The `assistStep` inside the workflow awaits the LLM response and guardrail validation, bounded by a 45 s step timeout.

The entity write (`recordTurnAnswered`) and the SSE push to the UI happen immediately after the response passes the guardrail — no additional async hop.

## State machine

The `SessionEntity` state machine has four states. The interesting paths:

- The normal conversation path is `CREATED → ACTIVE → ... → CLOSED`. The entity stays `ACTIVE` while turns cycle through `TurnStarted → TurnAnswered` (or `TurnRejected` if the guardrail exhausts all iterations).
- Two failure paths land in `FAILED`: an init error during `CREATED`, and an unrecoverable agent error during `ACTIVE`. A `FAILED` session's prior turns are preserved on the entity for debugging.
- `CLOSED` is reached explicitly via `SessionWorkflow.closeStep`, which the UI triggers when the shopper ends the conversation or after a configured idle timeout.

## Entity model

`SessionEntity` is the source of truth. It emits seven event types. `SessionView` projects every event into a lean row (session id, shopper id, turn count, last-turn status) used by the UI. `SessionWorkflow` both reads (`getSession` for context) and writes (`recordTurnAnswered`, `recordTurnRejected`, `fail`) on the entity. The relationship between `SalesAssistantAgent` and `AssistantResponse` is "returns" — the agent's task result is the response record that the workflow writes onto the entity.

## Guardrail defence

For any `AssistantResponse` that lands in the entity log, the response passed through:

1. **SalesAssistantAgent** — one model call, one structured output.
2. **ResponseGuardrail (before-agent-response)** — title existence validated against `CatalogIndex`, forbidden topics scanned, promotional claims cross-referenced against catalog `shortPitch` fields.

Each check is independent. A passing response means all three checks cleared — there is no partial pass.

The guardrail does not run after the response is recorded. It runs inside the agent loop before the response leaves the agent. This is the correct placement for customer-facing recommendation safety: stopping a bad response before the shopper sees it, not flagging it after.
