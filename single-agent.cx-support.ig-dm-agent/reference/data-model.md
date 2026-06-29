# Data model — ig-dm-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BrandProfile` | `profileId` | `String` | no | Stable id for the brand config. |
| | `name` | `String` | no | Human-readable brand name. |
| | `voiceDescription` | `String` | no | Prose description of the brand's desired tone. |
| | `prohibitedTerms` | `List<String>` | no | Terms that must not appear in any reply. |
| | `competitorNames` | `List<String>` | no | Competitor brand names that must not be mentioned. |
| | `maxReplyChars` | `int` | no | Hard character ceiling for any outgoing reply. |
| | `outOfScopeTopics` | `List<String>` | no | Topics the agent must redirect rather than answer. |
| `InboundMessage` | `messageId` | `String` | no | UUID minted by `DmEndpoint`. |
| | `senderId` | `String` | no | Social-channel user identifier. |
| | `rawText` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `profileId` | `String` | no | Which brand profile governs this message. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `sanitizedText` | `String` | no | PII stripped; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","url","person-name"]`. |
| `DmReply` | `replyText` | `String` | no | Outbound message text. |
| | `toneLabel` | `ToneLabel` | no | Enum value. |
| | `toneConfidenceScore` | `int` | no | 1–5, agent's self-assessed tone alignment. |
| | `decidedAt` | `String` | no | ISO-8601 timestamp when the agent returned. |
| `DmMessage` (entity state) | `messageId` | `String` | no | — |
| | `inbound` | `Optional<InboundMessage>` | yes | Populated after `MessageReceived`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `reply` | `Optional<DmReply>` | yes | Populated after `ReplyRecorded`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `DmMessage` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToneLabel`: `FRIENDLY`, `PROFESSIONAL`, `EMPATHETIC`, `NEUTRAL`.
`MessageStatus`: `RECEIVED`, `SANITIZED`, `REPLYING`, `REPLIED`, `FAILED`.

## Events (`DmEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `inbound` | → RECEIVED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `ReplyStarted` | — | → REPLYING |
| `ReplyRecorded` | `reply` | → REPLIED (terminal happy) |
| `MessageFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `DmMessage.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DmRow` mirrors `DmMessage` minus `inbound.rawText` (the audit log keeps that). The UI shows the raw text in the right-pane detail via `GET /api/messages/{id}` which returns the full entity state including `inbound.rawText`.

The view declares ONE query: `getAllMessages: SELECT * AS messages FROM dm_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`DmTasks.java`)

```java
public final class DmTasks {
  public static final Task<DmReply> REPLY_TO_MESSAGE = Task
      .name("Reply to message")
      .description("Read the attached sanitized inbound DM and produce a DmReply matching the brand voice profile")
      .resultConformsTo(DmReply.class);

  private DmTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
