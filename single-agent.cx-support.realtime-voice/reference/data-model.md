# Data model — realtime-conversational-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerMessage` | `turnId` | `String` | no | UUID minted by `SessionEndpoint` at turn submission. |
| | `text` | `String` | no | The customer's message text. `"<<GREETING>>"` sentinel for greeting tasks. |
| | `receivedAt` | `Instant` | no | When the endpoint received the turn POST. |
| `AgentTurn` | `turnId` | `String` | no | Matches the `CustomerMessage.turnId`. |
| | `replyText` | `String` | no | The agent's reply text as delivered to the customer. |
| | `guardrailTriggered` | `boolean` | no | `true` if at least one iteration was rejected by `ResponseGuardrail`. |
| | `iterationsUsed` | `int` | no | Number of agent iterations consumed on this turn (1–3). |
| | `repliedAt` | `Instant` | no | When the agent returned the accepted reply. |
| `ConversationTurn` | `turnId` | `String` | no | Stable id for this turn. |
| | `customer` | `CustomerMessage` | no | The customer's input. |
| | `agent` | `Optional<AgentTurn>` | yes | Present after `AgentReplied` event; absent while `WAITING_FOR_AGENT`. |
| | `status` | `TurnStatus` | no | Enum value. |
| | `startedAt` | `Instant` | no | When `CustomerTurnReceived` was emitted. |
| `SessionSummary` | `qualityScore` | `int` | no | 1–5 from `TurnSummarizer`. |
| | `summaryText` | `String` | no | 1–3 sentence template string. |
| | `totalTurns` | `int` | no | Count of all turns including the greeting. |
| | `guardrailEvents` | `int` | no | Count of turns where `guardrailTriggered = true`. |
| | `summarizedAt` | `Instant` | no | When `TurnSummarizer.summarize` finished. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `customerId` | `String` | no | Supplied by the caller. |
| | `agentPersona` | `String` | no | Persona config label, e.g. `"retail-support"`. |
| | `turns` | `List<ConversationTurn>` | no | Ordered turn history; empty at session open. |
| | `summary` | `Optional<SessionSummary>` | yes | Populated after `SessionSummarized`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `openedAt` | `Instant` | no | When `SessionOpened` was emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Session` and `ConversationTurn` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `WAITING_FOR_AGENT`, `AGENT_REPLIED`, `FAILED`.

`SessionStatus`: `GREETING`, `ACTIVE`, `WAITING_FOR_AGENT`, `SUMMARIZING`, `CLOSED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `customerId`, `agentPersona`, `openedAt` | → GREETING |
| `GreetingEmitted` | `agentTurn` | → ACTIVE |
| `CustomerTurnReceived` | `customerMessage` | → WAITING_FOR_AGENT |
| `AgentReplied` | `turnId`, `agentTurn` | → ACTIVE |
| `TurnFailed` | `turnId`, `reason: String` | stays in FAILED |
| `SessionEndRequested` | — | → SUMMARIZING |
| `SessionSummarized` | `summary` | → CLOSED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with `turns = List.of()`, `summary = Optional.empty()`, `closedAt = Optional.empty()`, and `status = GREETING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session`. The view omits the raw text of each `CustomerMessage` from the list projection (the API returns it in full on `GET /api/sessions/{id}`). The view row stores enough to render the session list: `sessionId`, `customerId`, `agentPersona`, `status`, `openedAt`, `closedAt`, `turns` (as summarized metadata: `turnId`, `status`, `agentTurn.guardrailTriggered`), and `summary`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`ConversationTasks.java`)

```java
public final class ConversationTasks {
  public static final Task<AgentTurn> REPLY_TO_CUSTOMER = Task
      .name("Reply to customer")
      .description("Read the conversation history and the customer message; produce an AgentTurn reply")
      .resultConformsTo(AgentTurn.class);

  public static final Task<AgentTurn> GREET_CUSTOMER = Task
      .name("Greet customer")
      .description("Produce an opening greeting AgentTurn for this session")
      .resultConformsTo(AgentTurn.class);

  private ConversationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
