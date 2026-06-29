# Architecture — guardrails-demo

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one decision-making LLM call surrounded by two guardrail hooks at different lifecycle moments. `SessionEndpoint` accepts a session-create request and records a `SessionOpened` event on `SessionEntity`; it also accepts turn-send requests that trigger `SessionWorkflow.exchangeStep`. The `exchangeStep` first passes the user message through `TopicPolicyGuardrail` at the `before-llm-call` hook. If the topic is blocked, the workflow records `TurnBlocked` on the entity — no model call is made. If the topic is allowed, the workflow calls `ConversationAgent` — the single AutonomousAgent — with the user message as `TaskDef.instructions(...)` and the session context (last three turns) as a `TaskDef.attachment(...)`. Before any candidate reply leaves the agent loop, `ContentPolicyGuardrail` validates it at the `before-agent-response` hook. A reply that fails the check triggers a retry within the agent's 3-iteration budget. A passing reply is returned to the workflow, which records `ReplyRecorded` on the entity. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model over REST and SSE.

The graph deliberately has one agent. Both guardrails are supporting classes — they do not make independent LLM calls. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1): a user message on an allowed topic that passes both guardrails on the first attempt. Note the two distinct guardrail intercept points:

1. `TopicPolicyGuardrail` runs synchronously at the before-llm-call hook — before any model token is generated. Its decision takes microseconds and either terminates the exchange (BLOCKED) or passes control to the agent.
2. `ContentPolicyGuardrail` runs at the before-agent-response hook — after the model has generated a candidate reply but before the reply exits the agent loop. A rejection causes the loop to retry without the reply ever reaching the entity or the user.

The agent call itself is bounded by `exchangeStep`'s 60 s timeout. That budget covers up to 3 content-policy iterations.

## State machine

The state machine is shown at the per-turn level, not the per-session level. A session contains many turns; each turn has its own lifecycle. The interesting transitions:

- Happy path: `RECEIVED → CHECKING_INPUT → GENERATING → COMPLETED`.
- Topic block: `RECEIVED → CHECKING_INPUT → BLOCKED` (terminal; session stays OPEN for the next turn).
- Content-policy exhaustion: `RECEIVED → CHECKING_INPUT → GENERATING → FAILED` (3 consecutive before-agent-response rejections — the session stays OPEN).
- There is no `APPROVED` or `REJECTED` state at the session level. Sessions close explicitly via `POST /api/sessions/{sessionId}/close` or time out; they are not rejected wholesale by a guardrail.

## Entity model

`SessionEntity` is the source of truth. It emits eight event types. `SessionView` projects every event into a row that embeds the full turn list for the UI. `SessionWorkflow` both reads (`getSession`) and writes (`receiveTurn`, `recordTopicCheck`, `recordBlock`, `markGenerating`, `recordReply`, `failTurn`) on the entity. The relationship between `ConversationAgent` and `AgentReply` is "returns" — the agent's task result is the reply record.

## Two-guardrail defence in depth

For any reply that reaches the user, the message passed through:

1. **TopicPolicyGuardrail (before-llm-call)** — blocked topics never reach the model; the session log records the matched topic for audit.
2. **ConversationAgent** — one model call, one structured output.
3. **ContentPolicyGuardrail (before-agent-response)** — disallowed phrases, scope contradictions, and refusal-then-compliance patterns are caught before the reply exits the loop.

The two hooks are independent. A topic that is allowed by the input guardrail can still be caught by the output guardrail if the model's reply violates content policy. Removing either hook opens a gap the other does not cover.
