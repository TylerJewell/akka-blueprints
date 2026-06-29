# Architecture — akka-llm-client-basic

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one direct LLM call. `ConversationEndpoint` accepts a turn submission, writes `TurnSubmitted` onto `ConversationEntity`, then immediately calls `ConversationAgent` — the single AutonomousAgent — with the user's prompt and prior session turns as `TaskDef.instructions(...)`. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate response. Once a reply passes, the endpoint writes `TurnReplied` back to the entity and returns `{ turnId }` to the caller.

No Workflow sits between the endpoint and the agent. This is a deliberate design choice for the baseline: the caller blocks on the agent call, the endpoint is the orchestrator, and the entity is the durable record. Subsequent blueprints in the quickstart series layer in Workflows when the interaction requires durable multi-step coordination.

`ConversationView` projects every entity event into a session row; `ConversationEndpoint` serves the read model over REST and SSE so the UI can render turn history without polling the entity directly.

The graph has exactly one component that talks to a model: `ConversationAgent`. `ReplyGuardrail` is a supporting class — it runs inside the agent's before-response hook and does not make its own model call.

## Interaction sequence

The sequence traces the happy path (J1). Two moments stand out:

1. The `submitTurn` write on `ConversationEntity` happens before the agent call — so the turn is durably recorded as `PENDING` before any model latency is incurred.
2. The `before-agent-response` hook runs inside the agent's loop. If `ReplyGuardrail` rejects a candidate, the rejection stays inside the agent process; the endpoint sees only the eventual passing reply (or an error if all iterations are exhausted).

The agent call is bounded by a 30 s server-side timeout on the component-client call inside the endpoint handler.

## State machine

The state machine has two levels: session-level (`ACTIVE` / `CLOSED`) and turn-level (`PENDING` / `REPLIED` / `FAILED`). Sessions accumulate turns without changing state — every `TurnSubmitted`, `TurnReplied`, and `TurnFailed` event lands while the session remains `ACTIVE`. A session moves to `CLOSED` only on an explicit `closeSession` command. This model allows the UI to display the full turn history for an active session without re-querying the entity.

The two terminal turn states (`REPLIED`, `FAILED`) are only reachable from `PENDING`. There is no retry at the session level — a failed turn is recorded as `FAILED` and the user submits a new turn if they want to try again.

## Entity model

`ConversationEntity` is the source of truth for both session metadata and turn history. It emits four event types. `ConversationView` projects every event into a session row. `ConversationAgent` does not read from the entity directly — its context is assembled by the endpoint handler before the agent call.

The relationship between `ConversationAgent` and `ConversationReply` is "returns": the agent's task result is the reply record. `ReplyGuardrail` guards the agent and is not a separate entity-level component.

## Single-agent invariant

For any reply that lands in the entity log, one model call was made through one component — `ConversationAgent`. The guardrail is a structural gate on that call's output, not a second LLM. Session state management and view projection involve no model calls. This makes the component graph a clean baseline for understanding what Akka's Conversation API does before any domain-specific logic or multi-agent coordination is introduced.
