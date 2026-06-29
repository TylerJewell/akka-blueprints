# Architecture — games-sales

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SalesEndpoint` accepts a query submission and writes a `SessionStarted` event onto `SessionEntity`. `SessionWorkflow` is started at session creation; its `queryStep` calls `SalesAssistantAgent` — the single AutonomousAgent — with the customer query and preferences as `TaskDef.instructions(...)` and, for follow-up queries, the prior `SalesRecommendation` as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`RecommendationGuardrail`) validates each candidate response against the loaded game catalog before the recommendation leaves the agent loop. Once a recommendation passes, the workflow writes `RecommendationRecorded` to the entity and the session transitions to `RECOMMENDED`. `RecommendationView` projects every entity event into a read-model row; `SalesEndpoint` serves the read model to the UI over REST and SSE.

The follow-up path reuses the same agent instance (same `sessionId`-scoped conversation context) but attaches the prior recommendation so the model can refine its output rather than starting from scratch.

There is no second agent. This is a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces two turns: the initial query (J1 happy path) and a follow-up query (J3). Two moments where the system waits:

1. The `queryStep` poll: `SessionWorkflow` calls `runSingleTask` and then awaits the result via `forTask(taskId).result(RECOMMEND_GAMES)`. LLM latency is covered by the 60 s step timeout.
2. The follow-up attachment assembly: the workflow fetches the session's existing `SalesRecommendation` via `componentClient.forEventSourcedEntity(SessionEntity.class, sessionId).call(SessionEntity::getSession)`, serialises it to JSON, and passes it as the attachment before calling the agent again.

The guardrail runs inside the agent loop on every iteration. A rejection adds one iteration and re-enters the same LLM call with the structured error message appended to the conversation.

## State machine

Five states. Key paths:

- The happy path is `QUERYING → RECOMMENDED`.
- A follow-up extends this to `RECOMMENDED → FOLLOW_UP_QUERYING → FOLLOW_UP_RECOMMENDED`.
- `FAILED` is reachable from both `QUERYING` and `FOLLOW_UP_QUERYING` if all three guardrail iterations are exhausted or if the agent step times out. A `FAILED` session's prior data is preserved on the entity.
- There is no `PURCHASED` or `CONVERTED` state. The recommendation is advisory; the customer acts in the retailer's checkout flow, which is outside this system.

## Entity model

`SessionEntity` is the source of truth. It emits five event types. `RecommendationView` projects every event into a row used by the UI. `SessionWorkflow` both reads (`getSession`) and writes (`recordRecommendation`, `recordFollowUp`, `fail`) on the entity. The relationship between `SalesAssistantAgent` and `SalesRecommendation` is "returns" — the agent's task result is the recommendation record.

## Guardrail enforcement flow

For any recommendation that lands in the entity log, the response passed through:

1. **SalesAssistantAgent** — one model call per iteration (up to 3), one structured output.
2. **RecommendationGuardrail** — catalog ID validation, confidence score range check, non-empty suggestions and summary — catches bad parses and hallucinated catalog entries before the recommendation leaves the agent loop.

The guardrail is the only governance mechanism in this baseline. Removing it means a hallucinated `catalogId` would reach the UI and the entity log — directing a customer to a product that does not exist in the catalog. The guardrail closes that gap with no LLM overhead.
