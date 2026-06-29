# Data model — mem0-react-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UserMessage` | `sessionId` | `String` | no | Session this message belongs to. |
| | `userId` | `String` | no | Caller-supplied user identifier. |
| | `text` | `String` | no | Raw user message text. |
| | `sentAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCall` | `toolName` | `String` | no | One of `recall-memories`, `store-memory`, `web-search`, `calculate`. |
| | `input` | `String` | no | Argument(s) passed to the tool, serialised as a string. |
| | `output` | `String` | no | Tool's return value, serialised as a string. |
| | `calledAt` | `Instant` | no | When the agent invoked this tool. |
| `AgentAnswer` | `turnId` | `String` | no | UUID minted by `AgentEndpoint`. |
| | `answer` | `String` | no | Agent's response to the user. |
| | `toolCalls` | `List<ToolCall>` | no | Ordered list of tool invocations made during this turn. |
| | `answeredAt` | `Instant` | no | When the agent task completed. |
| `MemoryFact` | `factId` | `String` | no | UUID minted by `AgentEndpoint` when the store-memory tool is called. |
| | `userId` | `String` | no | Owner of this fact. |
| | `rawText` | `String` | no | Text as provided by the agent, before sanitization. Audit-only. |
| | `requestedAt` | `Instant` | no | When `MemoryEntity.requestStore` was called. |
| `SanitizedFact` | `cleanText` | `String` | no | PII-redacted text; this is what gets persisted and recalled. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","ssn","person-name"]`. |
| `DriftSignal` | `userId` | `String` | no | User whose fact count triggered the signal. |
| | `factCount` | `int` | no | Count at the moment of signalling. |
| | `threshold` | `int` | no | Configured threshold value. |
| | `signaledAt` | `Instant` | no | When `DriftMonitor` computed the breach. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `userId` | `String` | no | — |
| | `turns` | `List<UserMessage>` | no | All turns in this session, in order. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Memory` (entity state) | `factId` | `String` | no | — |
| | `userId` | `String` | no | — |
| | `request` | `Optional<MemoryFact>` | yes | Populated after `FactStoreRequested`. |
| | `sanitized` | `Optional<SanitizedFact>` | yes | Populated after `FactSanitized`. |
| | `status` | `MemoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `FactStoreRequested` emitted. |
| | `persistedAt` | `Optional<Instant>` | yes | Terminal happy-path timestamp. |

Every nullable field on `Memory` and `Session` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SessionStatus`: `OPEN`, `ACTIVE`, `CLOSED`.
`MemoryStatus`: `SANITIZE_PENDING`, `SANITIZED`, `PERSISTED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `userId` | → OPEN |
| `TurnStarted` | `turnId, text` | → ACTIVE |
| `TurnAnswered` | `answer: AgentAnswer` | ACTIVE (turn complete, session remains open) |
| `SessionClosed` | — | → CLOSED (terminal) |

`emptyState()` returns `Session.initial("")` with empty turns list and `status = OPEN`. Never references `commandContext()` (Lesson 3).

## Events (`MemoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `FactStoreRequested` | `request: MemoryFact` | → SANITIZE_PENDING |
| `FactSanitized` | `sanitized: SanitizedFact` | → SANITIZED |
| `FactPersisted` | — | → PERSISTED (terminal happy) |
| `FactFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Memory.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SANITIZE_PENDING`.

## View rows

**`SessionRow`** mirrors `Session`. The UI renders full turn history from this row.

**`MemoryRow`** mirrors `Memory` minus `request.rawText` — the audit log keeps the raw text on the entity; the view only exposes `cleanText`. The UI never displays `rawText`.

**`DriftSignalRow`** mirrors `DriftSignal`. Queried separately via `getDriftSignals`.

### Queries

`SessionView`:
- `getAllSessions: SELECT * AS sessions FROM session_view` — no status filter (Lesson 2).

`MemoryView`:
- `getFactsByUser: SELECT * AS facts FROM memory_view WHERE userId = :userId`
- `countByUserId: SELECT COUNT(*) FROM memory_view WHERE userId = :userId AND status = 'PERSISTED'` — used by `DriftMonitor`.
- `getDriftSignals: SELECT * AS signals FROM drift_signal_view`

## Task definition (`AgentTasks.java`)

```java
public final class AgentTasks {
  public static final Task<AgentAnswer> ANSWER_TURN = Task
      .name("Answer turn")
      .description("Reason over the user message using available tools; recall and store memory facts as needed; return a complete answer")
      .resultConformsTo(AgentAnswer.class);

  private AgentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
