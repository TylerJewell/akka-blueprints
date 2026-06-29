# Data model — with Signals & Queries

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TurnRecord` | `turnId` | `String` | no | UUID minted by the workflow at signal receipt. |
| | `prompt` | `String` | no | The user's or operator's input text. |
| | `reply` | `String` | no | The agent's reply text. Empty string until `TurnReplied` fires. |
| | `source` | `TurnSource` | no | Enum value: who sent this prompt. |
| | `sentAt` | `Instant` | no | When the signal was received by the workflow. |
| | `repliedAt` | `Instant` | no | When the agent returned `AgentReply`. |
| `AddTurnSignal` | `prompt` | `String` | no | Inbound prompt text; validated by `SignalValidator`. |
| | `source` | `TurnSource` | no | Defaults to `USER` from the API layer. |
| `PauseSignal` | `reason` | `String` | no | Human-readable explanation of why the session was paused. |
| `ResumeSignal` | `correctiveNote` | `String` | no | Operator-supplied note prepended to the next turn's context. May be empty string. |
| `AgentReply` | `replyText` | `String` | no | The agent's response. Must be non-empty. |
| | `generatedAt` | `Instant` | no | When the agent task completed. |
| `SessionState` (entity state) | `sessionId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `sessionName` | `String` | no | User-supplied label. |
| | `turns` | `List<TurnRecord>` | no | Ordered list of all turns. Empty until first `TurnAdded`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `pauseReason` | `Optional<String>` | yes | Set on `SessionPaused`; cleared on `SessionResumed`. |
| | `startedAt` | `Instant` | no | When `SessionOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `SessionRow` (view row) | `sessionId` | `String` | no | — |
| | `sessionName` | `String` | no | — |
| | `turnCount` | `int` | no | Derived from `turns.size()` in the view updater. |
| | `status` | `SessionStatus` | no | — |
| | `lastReply` | `Optional<String>` | yes | The most recent `TurnRecord.reply`; null until first reply. |
| | `startedAt` | `Instant` | no | — |
| | `closedAt` | `Optional<Instant>` | yes | — |

Every nullable field on `SessionState` and `SessionRow` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnSource`: `USER`, `OPERATOR`, `SYSTEM`.
`SessionStatus`: `STARTING`, `ACTIVE`, `PAUSED`, `CLOSING`, `CLOSED`, `FAILED`.

## Events (`ChatSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `sessionId`, `sessionName`, `startedAt` | → ACTIVE (from STARTING) |
| `TurnAdded` | `turnId`, `prompt`, `source`, `sentAt` | stays ACTIVE (partial turn) |
| `TurnReplied` | `turnId`, `replyText`, `repliedAt` | stays ACTIVE (turn complete) |
| `SessionPaused` | `reason` | → PAUSED |
| `SessionResumed` | `correctiveNote` | → ACTIVE |
| `SessionClosed` | `closedAt` | → CLOSED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `SessionState.initial("")` with `turns = Collections.emptyList()`, all `Optional` fields as `Optional.empty()`, and `status = STARTING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `SessionState` at a summary level — it does not include the full `turns` list (the UI fetches that from the entity directly via `GET /api/sessions/{id}`). The view carries only `lastReply` so the live list can show a preview without a separate entity call.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<AgentReply> PROCESS_TURN = Task
      .name("Process turn")
      .description("Continue the conversation given the accumulated context and the new prompt, returning an AgentReply")
      .resultConformsTo(AgentReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Workflow state (in-memory, not persisted as entity events)

`ChatSessionWorkflow` holds an in-memory copy of:
- `pendingPrompt: Optional<String>` — the prompt from the most recent `addTurnSignal`, cleared once `processTurnStep` picks it up.
- `pendingTurnId: Optional<String>` — the turn id minted for the pending prompt.
- `correctiveNote: Optional<String>` — populated on `resumeSignal`; consumed and cleared in the next `processTurnStep` call.
- `currentStatus: SessionStatus` — mirrors the entity status for signal-acceptance guards.

This state is local to the workflow actor. The entity's event log is the authoritative record; the workflow state is reconstructed from the in-memory activation. `getSessionQuery` returns a `SessionState` assembled from the entity's last known state plus the workflow's pending fields.
