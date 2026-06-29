# Data model — voice-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TurnRequest` | `conversationId` | `String` | no | UUID of the owning session. |
| | `turnId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| | `callerId` | `String` | no | Caller identifier (opaque to the agent). |
| | `rawTranscript` | `String` | no | Pre-sanitization transcript. Audit-only. |
| | `audioFormat` | `String` | no | e.g. `"wav"`, `"mp3"`, `"simulated"`. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedTranscript` | `redactedTranscript` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["name","phone","account-number","email","address"]`. |
| `AgentReply` | `replyText` | `String` | no | ≤ 500 characters; spoken to the caller. |
| | `tone` | `VoiceTone` | no | Enum value. |
| | `topicClassification` | `String` | no | e.g. `"billing"`, `"technical"`, `"cancellation"`. |
| | `escalationFlag` | `boolean` | no | `true` when a human agent is required. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `AudioResponse` | `audioBytes` | `byte[]` | no | WAV (stub) or real TTS output. |
| | `audioFormat` | `String` | no | `"wav"` (stub default). |
| | `durationMs` | `int` | no | Approximate spoken duration in milliseconds. |
| | `synthesisedAt` | `Instant` | no | When `AudioSynthesiser` finished. |
| `Turn` (embedded in session state) | `turnId` | `String` | no | — |
| | `request` | `Optional<TurnRequest>` | yes | Populated after `AudioReceived`. |
| | `sanitized` | `Optional<SanitizedTranscript>` | yes | Populated after `TranscriptSanitized`. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after `ReplyRecorded`. |
| | `audio` | `Optional<AudioResponse>` | yes | Populated after `AudioSynthesised`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AudioReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `ConversationSession` (entity state) | `conversationId` | `String` | no | — |
| | `callerId` | `String` | no | — |
| | `turns` | `List<Turn>` | no | Ordered list of all turns; grows on each `AudioReceived`. |
| | `sessionStatus` | `SessionStatus` | no | See enum. |
| | `startedAt` | `Instant` | no | When the session was created. |
| | `closedAt` | `Optional<Instant>` | yes | Set on `SessionClosed`. |

Every nullable lifecycle field on `Turn` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`VoiceTone`: `WARM`, `NEUTRAL`, `FIRM`.
`TurnStatus`: `AUDIO_RECEIVED`, `TRANSCRIPT_SANITIZED`, `RESPONDING`, `REPLY_RECORDED`, `AUDIO_SYNTHESISED`, `FAILED`.
`SessionStatus`: `ACTIVE`, `CLOSED`, `FAILED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AudioReceived` | `request` | Adds new `Turn` in `AUDIO_RECEIVED` |
| `TranscriptSanitized` | `turnId`, `sanitized` | Turn → `TRANSCRIPT_SANITIZED` |
| `ResponseStarted` | `turnId` | Turn → `RESPONDING` |
| `ReplyRecorded` | `turnId`, `reply` | Turn → `REPLY_RECORDED` |
| `AudioSynthesised` | `turnId`, `audio` | Turn → `AUDIO_SYNTHESISED` (terminal happy) |
| `TurnFailed` | `turnId`, `reason: String` | Turn → `FAILED` (terminal) |
| `SessionClosed` | — | Session → `CLOSED` |

`emptyState()` returns `ConversationSession.initial("")` with an empty `turns` list, `sessionStatus = ACTIVE`, and `closedAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `ConversationSession` minus per-turn `rawTranscript` (the audit log keeps that). The UI fetches the raw transcript on demand via `GET /api/conversations/{id}` and reads `turns[i].request.rawTranscript` from the JSON.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ConversationTasks.java`)

```java
public final class ConversationTasks {
  public static final Task<AgentReply> VOICE_RESPOND = Task
      .name("Respond to caller")
      .description("Read the sanitized transcript attachment and produce an AgentReply for the caller")
      .resultConformsTo(AgentReply.class);

  private ConversationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
