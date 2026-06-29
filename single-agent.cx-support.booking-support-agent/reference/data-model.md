# Data model — booking-support-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerMessage` | `sessionId` | `String` | no | Session this message belongs to. |
| | `turnId` | `String` | no | UUID minted by `SupportEndpoint`. |
| | `customerId` | `String` | no | Customer placing the request. |
| | `rawMessage` | `String` | no | Pre-sanitization message text. Audit-only. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `sanitizedText` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["payment-card","email","phone","govt-id"]`. |
| `BookingRecord` | `bookingRef` | `String` | no | Stable booking reference (e.g., `BK-001`). |
| | `customerId` | `String` | no | Owner of the booking. |
| | `flightNumber` | `String` | no | IATA flight number. |
| | `origin` | `String` | no | IATA origin airport code. |
| | `destination` | `String` | no | IATA destination airport code. |
| | `departureAt` | `Instant` | no | Scheduled departure (UTC). |
| | `arrivalAt` | `Instant` | no | Scheduled arrival (UTC). |
| | `seatNumber` | `String` | no | Assigned seat. |
| | `bookingStatus` | `BookingStatus` | no | Enum value. |
| | `fareClass` | `String` | no | Single-letter fare code. |
| | `cancellationPermitted` | `boolean` | no | Whether fare allows cancellation. |
| | `seatChangePermitted` | `boolean` | no | Whether fare allows seat changes. |
| `ToolCall` | `toolName` | `String` | no | Name of the tool called. |
| | `targetBookingRef` | `String` | no | Booking the call targeted. |
| | `requestedChange` | `String` | no | Human-readable description of the requested operation. |
| | `outcome` | `ToolCallOutcome` | no | `ALLOWED` or `BLOCKED`. |
| | `guardrailReason` | `String` | yes | `OWNERSHIP_VIOLATION`, `FARE_RULE_VIOLATION`, `DUPLICATE_MUTATION`, or null when ALLOWED. |
| `AgentReply` | `replyText` | `String` | no | Natural-language reply to the customer. |
| | `toolCalls` | `List<ToolCall>` | no | One entry per tool call attempted this turn. |
| | `updatedBooking` | `Optional<BookingRecord>` | yes | Set when a booking mutation occurred; absent otherwise. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `SessionTurn` | `turnId` | `String` | no | — |
| | `message` | `Optional<CustomerMessage>` | yes | Populated after `CustomerMessageReceived`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after `AgentTurnCompleted`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When the turn was created. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `BookingSession` (entity state) | `sessionId` | `String` | no | — |
| | `customerId` | `String` | no | Customer owning the session. |
| | `turns` | `List<SessionTurn>` | no | All turns in the session; empty on creation. |
| | `sessionStatus` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `SessionTurn` and `BookingSession` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BookingStatus`: `CONFIRMED`, `MODIFIED`, `CANCELLED`, `REFUND_PENDING`.
`ToolCallOutcome`: `ALLOWED`, `BLOCKED`.
`TurnStatus`: `RECEIVED`, `SANITIZED`, `AGENT_REPLIED`, `FAILED`.
`SessionStatus`: `OPEN`, `CLOSED`, `FAILED`.

## Events (`BookingSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `customerId` | Session OPEN |
| `CustomerMessageReceived` | `message: CustomerMessage` | Turn → RECEIVED |
| `MessageSanitized` | `turnId`, `sanitized: SanitizedMessage` | Turn → SANITIZED |
| `AgentTurnCompleted` | `turnId`, `reply: AgentReply` | Turn → AGENT_REPLIED |
| `TurnFailed` | `turnId`, `reason: String` | Turn → FAILED |
| `SessionClosed` | — | Session → CLOSED |

`emptyState()` returns `BookingSession.initial("")` with `turns = emptyList()`, `sessionStatus = OPEN`, `closedAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `BookingSession`. The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`BookingSupportTasks.java`)

```java
public final class BookingSupportTasks {
  public static final Task<AgentReply> HANDLE_SUPPORT_TURN = Task
      .name("Handle support turn")
      .description("Answer the customer's question or carry out the requested booking change, using only the booking tools provided")
      .resultConformsTo(AgentReply.class);

  private BookingSupportTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
