# Data model — bidi-demo

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChannelMessage` | `messageId` | `String` | no | UUID minted by `ChannelEndpoint`. |
| | `channelId` | `String` | no | Parent channel. |
| | `content` | `String` | no | The raw message text. |
| | `sentBy` | `String` | no | User identifier. |
| | `sentAt` | `Instant` | no | When the endpoint received the message. |
| `ResponseFrame` | `turnId` | `String` | no | MUST match the current turn id. |
| | `frameIndex` | `int` | no | 0-based, monotonically increasing. |
| | `content` | `String` | no | One sentence of the agent's reply; empty string only when `done == true`. |
| | `done` | `boolean` | no | `true` only on the terminal frame. |
| | `emittedAt` | `Instant` | no | When the frame was produced. |
| `ResponseFrameList` | `frames` | `List<ResponseFrame>` | no | Wrapper for agent task result. |
| `TurnSummary` | `turnId` | `String` | no | — |
| | `status` | `TurnStatus` | no | See enum. |
| | `frameCount` | `int` | no | Number of frames published so far. |
| | `completedAt` | `Optional<Instant>` | yes | Populated on `TurnCompleted`. |
| `Turn` | `turnId` | `String` | no | Minted by `MessageForwarder`. |
| | `message` | `ChannelMessage` | no | The inbound message. |
| | `frames` | `List<ResponseFrame>` | no | Accumulated frames; grows as `FramePublished` events apply. |
| | `status` | `TurnStatus` | no | See enum. |
| | `startedAt` | `Instant` | no | When `TurnStarted` emitted. |
| | `completedAt` | `Optional<Instant>` | yes | Populated on `TurnCompleted`. |
| `Channel` (entity state) | `channelId` | `String` | no | — |
| | `channelName` | `String` | no | User-supplied label. |
| | `turns` | `List<Turn>` | no | Ordered list; grows per turn. |
| | `status` | `ChannelStatus` | no | See enum. |
| | `turnBudget` | `int` | no | Max turns before auto-close; default 10. |
| | `openedAt` | `Instant` | no | When `ChannelOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Populated on `ChannelClosed`. |
| `ChannelRow` (view row) | `channelId` | `String` | no | — |
| | `channelName` | `String` | no | — |
| | `turns` | `List<TurnSummary>` | no | Summaries only; no frame content. |
| | `status` | `ChannelStatus` | no | — |
| | `turnBudget` | `int` | no | — |
| | `openedAt` | `Instant` | no | — |
| | `closedAt` | `Optional<Instant>` | yes | — |

Every nullable field on `Channel` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `PROCESSING`, `COMPLETE`, `FAILED`.
`ChannelStatus`: `OPEN`, `ACTIVE`, `CLOSED`, `FAILED`.

## Events (`ChannelEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ChannelOpened` | `channelId, channelName, turnBudget, openedBy, openedAt` | → OPEN |
| `MessageReceived` | `messageId, channelId, content, sentBy, sentAt` | OPEN → ACTIVE (first); ACTIVE stays ACTIVE |
| `TurnStarted` | `turnId, messageId` | adds Turn in PROCESSING |
| `FramePublished` | `turnId, frame: ResponseFrame` | appends frame to turn.frames |
| `TurnCompleted` | `turnId, completedAt` | turn → COMPLETE |
| `TurnFailed` | `turnId, reason: String` | turn → FAILED |
| `ChannelClosed` | `reason: String, closedAt: Instant` | → CLOSED (budget) or FAILED (error) |

`emptyState()` returns `Channel.initial("")` with `turns = List.of()`, `status = OPEN`, `turnBudget = 10`, and `closedAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ChannelRow` mirrors `Channel` but replaces the full `List<Turn>` (which includes frame content) with `List<TurnSummary>` — carrying only `turnId`, `status`, `frameCount`, and `completedAt`. The UI fetches the full turn and frame detail on demand via `GET /api/channels/{id}`.

The view declares ONE query: `getAllChannels: SELECT * AS channels FROM channel_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChannelTasks.java`)

```java
public final class ChannelTasks {
  public static final Task<ResponseFrameList> RESPOND_TO_MESSAGE = Task
      .name("Respond to message")
      .description("Read the channel history attachment and the new user message, then return a list of ResponseFrame objects ending with done==true")
      .resultConformsTo(ResponseFrameList.class);

  private ChannelTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
