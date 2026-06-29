# Data model — guardrails-demo

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TurnRequest` | `turnId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `userMessage` | `String` | no | Free-text message from the user. |
| | `sessionId` | `String` | no | Parent session identifier. |
| | `sentAt` | `Instant` | no | When the endpoint received the request. |
| `TopicCheckResult` | `outcome` | `TopicCheckOutcome` | no | ALLOWED or BLOCKED. |
| | `matchedTopic` | `String` | yes | Non-null only when outcome is BLOCKED. |
| | `reason` | `String` | no | Human-readable explanation of the check result. |
| `AgentReply` | `replyText` | `String` | no | The agent's response text. |
| | `contentPolicyIterations` | `int` | no | Number of before-agent-response iterations required (1 = first pass succeeded). |
| | `repliedAt` | `Instant` | no | When the reply was produced. |
| `TurnRecord` | `turnId` | `String` | no | — |
| | `userMessage` | `String` | no | Preserved for audit regardless of outcome. |
| | `topicCheck` | `Optional<TopicCheckResult>` | yes | Populated after `TopicChecked` event. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after `ReplyRecorded` event. Null when BLOCKED or FAILED. |
| | `status` | `TurnStatus` | no | See enum. |
| | `startedAt` | `Instant` | no | When `TurnReceived` emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `userId` | `String` | no | User identifier from `CreateSessionRequest`. |
| | `turns` | `List<TurnRecord>` | no | Ordered list; never null, may be empty. |
| | `status` | `SessionStatus` | no | See enum. |
| | `openedAt` | `Instant` | no | When `SessionOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Populated after `SessionClosed` event. |

Every nullable field uses `Optional<T>`. The view table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TopicCheckOutcome`: `ALLOWED`, `BLOCKED`.
`TurnStatus`: `RECEIVED`, `CHECKING_INPUT`, `BLOCKED`, `GENERATING`, `COMPLETED`, `FAILED`.
`SessionStatus`: `OPEN`, `CLOSED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `userId` | → OPEN (session level) |
| `TurnReceived` | `turnRequest` | turn → RECEIVED |
| `TopicChecked` | `topicCheckResult` | turn → CHECKING_INPUT |
| `TurnBlocked` | `turnId, topic, reason` | turn → BLOCKED (terminal) |
| `GenerationStarted` | `turnId` | turn → GENERATING |
| `ReplyRecorded` | `turnId, reply` | turn → COMPLETED (terminal happy) |
| `TurnFailed` | `turnId, reason` | turn → FAILED (terminal) |
| `SessionClosed` | — | session → CLOSED (terminal) |

`emptyState()` returns `Session.initial("")` with an empty `turns` list, `status = OPEN`, and `closedAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` including the full embedded `List<TurnRecord>` turns. The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SessionTasks.java`)

```java
public final class SessionTasks {
  public static final Task<AgentReply> REPLY_TO_MESSAGE = Task
      .name("Reply to message")
      .description("Read the user message and session context and produce an AgentReply")
      .resultConformsTo(AgentReply.class);

  private SessionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Guardrail rejection payloads

`TopicPolicyGuardrail` rejection (returned to `SessionWorkflow.exchangeStep`):

```json
{
  "type": "topic-blocked",
  "matchedTopic": "financial-advice",
  "reason": "Message matched blocked topic: financial-advice"
}
```

`ContentPolicyGuardrail` rejection (returned to the agent loop):

```json
{
  "type": "content-policy-violation",
  "violatedCheck": "disallowed-phrase",
  "excerpt": "...the offending phrase excerpt..."
}
```

The agent loop receives the content-policy rejection as a structured error and retries on the next iteration.
