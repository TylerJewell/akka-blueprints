# Data model — rag-support-bot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `QueryRequest` | `conversationId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| | `rawQuery` | `String` | no | Pre-sanitization customer message. Audit-only. |
| | `topicFilter` | `String` | no | `ANY` / `ORDERS` / `RETURNS` / `ACCOUNT` / `TECHNICAL`. |
| | `submittedBy` | `String` | no | Session or user identifier. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedQuery` | `redactedQuery` | `String` | no | PII stripped; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","order-id","payment-card-number"]`. |
| `KbPassage` | `passageId` | `String` | no | Stable id: `<articleId>-p<index>`. |
| | `articleId` | `String` | no | Parent article identifier. |
| | `articleTitle` | `String` | no | Human-readable article title. |
| | `passageText` | `String` | no | 200-word segment of the article body. |
| | `topic` | `String` | no | `ORDERS` / `RETURNS` / `ACCOUNT` / `TECHNICAL`. |
| `RetrievalResult` | `passages` | `List<KbPassage>` | no | Top-5 passages matched by keyword retrieval. |
| | `passageCount` | `int` | no | Convenience count (equal to `passages.size()`). |
| `SupportReply` | `answerText` | `String` | no | 50–600 characters. |
| | `sourcePassageIds` | `List<String>` | no | Passage ids from retrieval result that ground the answer. |
| | `escalation` | `boolean` | no | `true` iff `outcome == ESCALATE`. |
| | `escalationReason` | `String` | no | Empty string when escalation is false. |
| | `outcome` | `ReplyOutcome` | no | Enum value. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `GroundingEval` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `GroundingScorer` finished. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `query` | `Optional<QueryRequest>` | yes | Populated after `QueryReceived`. |
| | `sanitized` | `Optional<SanitizedQuery>` | yes | Populated after `QuerySanitized`. |
| | `retrieved` | `Optional<RetrievalResult>` | yes | Populated after `RetrievalCompleted`. |
| | `reply` | `Optional<SupportReply>` | yes | Populated after `ReplyRecorded`. |
| | `eval` | `Optional<GroundingEval>` | yes | Populated after `EvaluationScored`. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ReplyOutcome`: `ANSWERED`, `PARTIALLY_ANSWERED`, `ESCALATE`.
`ConversationStatus`: `RECEIVED`, `SANITIZED`, `RETRIEVING`, `REPLYING`, `REPLY_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QueryReceived` | `query` | → RECEIVED |
| `QuerySanitized` | `sanitized` | → SANITIZED |
| `RetrievalCompleted` | `retrieved` | → RETRIEVING |
| `ReplyStarted` | — | → REPLYING |
| `ReplyRecorded` | `reply` | → REPLY_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `ConversationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Conversation.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## Events (`KnowledgeBase`)

| Event | Payload | Transition |
|---|---|---|
| `ArticleLoaded` | `article: KbArticle` | article indexed |

`KbArticle` record fields: `articleId`, `title`, `body`, `topic`, `passages: List<KbPassage>`. Passages are computed in the event-applier by splitting `body` into 200-word segments.

## View row

`ConversationRow` mirrors `Conversation` minus `query.rawQuery` (the audit log keeps that). The UI fetches the raw query on demand via `GET /api/conversations/{id}` and reads `query.rawQuery` from the JSON.

The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SupportTasks.java`)

```java
public final class SupportTasks {
  public static final Task<SupportReply> ANSWER_QUERY = Task
      .name("Answer customer query")
      .description("Read the retrieved passages and compose a SupportReply anchored in those passages")
      .resultConformsTo(SupportReply.class);

  private SupportTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
