# Architecture — realtime-conversational-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call per conversation turn. `SessionEndpoint` accepts a session-start request, writes a `SessionOpened` event onto `SessionEntity`, and starts a `SessionWorkflow` instance. The workflow's `greetStep` calls `ConversationalAgent` — the single AutonomousAgent — with a GREET_CUSTOMER task. The agent's `before-agent-response` guardrail (`ResponseGuardrail`) validates each candidate reply. Once the greeting passes, the workflow suspends in `activeStep` and waits for customer input.

When the customer sends a message, `SessionEndpoint` signals the workflow via `submitTurn`. The workflow's `converseTurnStep` calls `ConversationalAgent` again, passing the full conversation history as the task's instruction context and the customer's latest message as a `TaskDef.attachment(...)`. The guardrail runs on every candidate reply. Once a reply passes, the workflow writes `AgentReplied` to the entity and suspends again for the next customer turn.

When the customer ends the session, `SessionEndpoint` calls `requestEnd` on the workflow. The `summarizeStep` calls `TurnSummarizer` — a deterministic, rule-based scorer with no LLM call — which produces a `SessionSummary`. The summary lands on the entity as `SessionSummarized`. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The session summarizer (`TurnSummarizer`) is not an LLM — it is a formula applied to turn counts and guardrail event counts. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces a two-turn happy path (J1). Note three moments where the system pauses:

1. The `greetStep` call to the agent — bounded by a 15 s step timeout; fast in normal operation.
2. The `converseTurnStep` call to the agent — bounded by a 60 s step timeout; the main LLM latency budget.
3. The `activeStep` suspension waiting for customer input — unbounded by design; the session stays open until the customer sends a message or ends the session.

The `summarizeStep` runs in milliseconds — no external service, no LLM call.

## State machine

Eight states. The interesting paths:

- The happy path is `GREETING → ACTIVE → WAITING_FOR_AGENT → ACTIVE → ... → SUMMARIZING → CLOSED`.
- `WAITING_FOR_AGENT → ACTIVE` loops for each turn in the conversation.
- Three failure transitions land in `FAILED`: a greet error during `GREETING`, a guardrail-exhaustion during `WAITING_FOR_AGENT`, and a summarize error during `SUMMARIZING`. A `FAILED` session's prior turns are preserved on the entity — the operator can review the partial history.
- There is no `ESCALATED` or `TRANSFERRED` state in the blueprint. Those are deployer-added transitions; the pattern stops at `CLOSED` or `FAILED`.

## Entity model

`SessionEntity` is the source of truth. It emits eight event types. `SessionView` projects every event into a row used by the UI. `SessionWorkflow` both reads (`getSession`) and writes (`recordGreeting`, `receiveCustomerTurn`, `recordAgentReply`, `recordSummary`, `fail`) on the entity. The relationship between `ConversationalAgent` and `AgentTurn` is "returns" — the agent's task result is the turn record, which the workflow forwards to the entity command.

## Defence-in-depth governance flow

For any agent reply that reaches the customer:

1. **ConversationalAgent** — one model call per turn, one structured `AgentTurn` output.
2. **before-agent-response guardrail** — policy violations (prohibited content, competitor names, token overrun) are caught before the reply leaves the agent loop; the customer never sees the rejected text.
3. **TurnSummarizer** — every completed session gets a 1–5 quality score so operators know which sessions to review first; sessions with guardrail events on every turn score 1 and are flagged.

Each step is independent. Removing the guardrail exposes the customer to whatever the model produces; removing the summarizer removes the signal operators use to find problem sessions in large-volume deployments.
