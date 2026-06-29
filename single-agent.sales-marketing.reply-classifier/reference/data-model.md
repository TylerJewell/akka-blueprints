# Data model — reply-classifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ReplySubmission` | `replyId` | `String` | no | UUID minted by `ReplyEndpoint`. |
| | `dealId` | `String` | no | Pipedrive deal identifier supplied by the user. |
| | `sender` | `String` | no | Sender email address. |
| | `rawReplyText` | `String` | no | Full inbound reply body. Audit-only; never shown in live list. |
| | `subject` | `String` | no | Email subject line. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `ReplyClassification` | `intent` | `ReplyIntent` | no | Enum value. |
| | `confidenceScore` | `int` | no | 0–100. |
| | `rationale` | `String` | no | 1–2 sentences naming the signal. |
| | `crmAction` | `CrmAction` | no | Proposed action. |
| | `classifiedAt` | `Instant` | no | When the agent returned. |
| `CrmAction` | `type` | `CrmActionType` | no | Enum value. |
| | `newStage` | `String` | yes | Populated only when `type == UPDATE_STAGE`. |
| | `skipReason` | `String` | yes | Populated only when `type == SKIP_CRM_UPDATE`. |
| `CrmUpdateResult` | `success` | `boolean` | no | Whether `PipedriveClient.updateDealStage` succeeded. |
| | `previousStage` | `String` | no | Stage before update. |
| | `newStage` | `String` | no | Stage after update (same as `previousStage` when `success == false`). |
| | `guardrailRejectionReason` | `String` | yes | Null when `success == true`. |
| | `updatedAt` | `Instant` | no | When the CRM update step completed. |
| `Reply` (entity state) | `replyId` | `String` | no | — |
| | `submission` | `Optional<ReplySubmission>` | yes | Populated after `ReplyReceived`. |
| | `classification` | `Optional<ReplyClassification>` | yes | Populated after `ReplyClassified`. |
| | `crmResult` | `Optional<CrmUpdateResult>` | yes | Populated after `CrmUpdateSucceeded` or `CrmUpdateSkipped`. |
| | `status` | `ReplyStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ReplyReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Reply` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ReplyIntent`: `INTERESTED`, `NOT_INTERESTED`, `OBJECTION`, `OUT_OF_OFFICE`, `UNSUBSCRIBE`.

`CrmActionType`: `UPDATE_STAGE`, `SKIP_CRM_UPDATE`.

`ReplyStatus`: `RECEIVED`, `CLASSIFYING`, `CLASSIFIED`, `CRM_UPDATED`, `CRM_SKIPPED`, `FAILED`.

## Events (`ReplyEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ReplyReceived` | `submission` | → RECEIVED |
| `ClassificationStarted` | — | → CLASSIFYING |
| `ReplyClassified` | `classification` | → CLASSIFIED |
| `CrmUpdateSucceeded` | `crmResult` | → CRM_UPDATED (terminal happy) |
| `CrmUpdateSkipped` | `skipReason: String` | → CRM_SKIPPED (terminal skip) |
| `ReplyFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Reply.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ReplyRow` mirrors `Reply` minus `submission.rawReplyText` — the audit log keeps the raw text on the entity. The UI fetches the full submission on demand via `GET /api/replies/{id}` and reads `submission.rawReplyText` from the JSON.

The view declares ONE query: `getAllReplies: SELECT * AS replies FROM reply_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ReplyTasks.java`)

```java
public final class ReplyTasks {
  public static final Task<ReplyClassification> CLASSIFY_REPLY = Task
      .name("Classify reply")
      .description("Read the attached email reply and return a ReplyClassification with intent, confidence, rationale, and a CRM action")
      .resultConformsTo(ReplyClassification.class);

  private ReplyTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Allowed stage transitions (`StageTransitions.java`)

```java
public final class StageTransitions {
  public static final Map<String, Set<String>> ALLOWED = Map.of(
    "PROSPECTING",  Set.of("QUALIFIED"),
    "QUALIFIED",    Set.of("PROPOSAL", "CLOSED_LOST"),
    "PROPOSAL",     Set.of("CLOSED_WON", "CLOSED_LOST"),
    "CLOSED_WON",   Set.of(),
    "CLOSED_LOST",  Set.of("PROSPECTING")
  );

  public static boolean isAllowed(String fromStage, String toStage) {
    return ALLOWED.getOrDefault(fromStage, Set.of()).contains(toStage);
  }

  private StageTransitions() {}
}
```

`CrmMutationGuardrail` uses `StageTransitions.isAllowed(currentStage, proposedStage)` as check (2) in its validation logic.
