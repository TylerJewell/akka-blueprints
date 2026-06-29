# Data model — akka-stateful-memory-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemoryBlock` | `blockId` | `String` | no | Well-known name: `"persona"` or `"human"`. |
| | `content` | `String` | no | Full text of the block at this point in time. |
| | `lastUpdatedAt` | `Instant` | no | When the last patch was applied. |
| `MemoryPatch` | `blockId` | `String` | no | Which block to replace. `"persona"` or `"human"`. |
| | `newContent` | `String` | no | Full replacement text. The existing block is overwritten. |
| `TurnMessage` | `role` | `String` | no | `"user"` or `"agent"`. |
| | `text` | `String` | no | Message body. |
| | `sentAt` | `Instant` | no | When the message was recorded on the entity. |
| `AgentTurnResult` | `replyText` | `String` | no | The agent's response to the user. |
| | `proposedPatches` | `List<MemoryPatch>` | no | Zero or more memory patches. May be empty list. |
| | `producedAt` | `Instant` | no | When the agent returned. |
| `DriftAnnotation` | `riskScore` | `double` | no | 0.0–1.0. |
| | `reason` | `String` | no | One sentence naming the dominant demographic cluster. |
| | `atTurnNumber` | `int` | no | Turn number when the eval fired (always a multiple of 10). |
| | `evaluatedAt` | `Instant` | no | When `DriftEvaluator` finished. |
| `Conversation` (entity state) | `conversationId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| | `personaBlock` | `MemoryBlock` | no | Always present; bootstrapped on `ConversationCreated`. |
| | `humanBlock` | `MemoryBlock` | no | Always present; starts empty and fills over turns. |
| | `history` | `List<TurnMessage>` | no | Ordered, oldest first. |
| | `latestDrift` | `Optional<DriftAnnotation>` | yes | Null until the first drift eval fires. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationCreated` emitted. |
| | `lastActivityAt` | `Instant` | no | Updated on every new event. |

`latestDrift` is the only `Optional<T>` field on `Conversation`. The view's table updater wraps it with `Optional.of(...)`; callers use `.orElse(null)` or `.isPresent()` (Lesson 6).

## Enums

`ConversationStatus`: `ACTIVE`, `ARCHIVED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationCreated` | `initialPersona: String` | → ACTIVE |
| `UserMessageReceived` | `userMessage: String, submittedBy: String` | ACTIVE → ACTIVE |
| `AgentResponseReady` | `result: AgentTurnResult` | ACTIVE → ACTIVE |
| `MemoryPatchApplied` | `patch: MemoryPatch` | ACTIVE → ACTIVE |
| `DriftEvaluated` | `annotation: DriftAnnotation` | ACTIVE → ACTIVE |
| `ConversationArchived` | — | ACTIVE → ARCHIVED (terminal) |

`emptyState()` returns `Conversation.initial("")` with `personaBlock` and `humanBlock` seeded as empty `MemoryBlock` instances, `history = List.of()`, `latestDrift = Optional.empty()`, and `status = ACTIVE`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `Conversation` with `history` truncated to the last 50 `TurnMessage` entries. The full history is available on `ConversationEntity` via `GET /api/conversations/{id}`. The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AgentTasks.java`)

```java
public final class AgentTasks {
  public static final Task<AgentTurnResult> GENERATE_TURN = Task
      .name("Generate turn")
      .description("Read the persona and human memory blocks and the recent message history, "
          + "reply to the user's message, and propose minimal memory patches for durable new facts")
      .resultConformsTo(AgentTurnResult.class);

  private AgentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
