# Data model — docreview

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ReviewInstruction` | `instructionId` | `String` | no | Stable id supplied by the user. |
| | `text` | `String` | no | What the reviewer must check. |
| | `severityFloor` | `String` | no | Lowest severity the agent may assign (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL`). |
| `ReviewRequest` | `reviewId` | `String` | no | UUID minted by `ReviewEndpoint`. |
| | `documentTitle` | `String` | no | User-supplied label. |
| | `rawDocument` | `String` | no | Pre-sanitization document body. Audit-only. |
| | `instructions` | `List<ReviewInstruction>` | no | Submitted list (1–N). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedDocument` | `redactedDocument` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","ssn","payment-card-number","address","person-name"]`. |
| `Finding` | `instructionId` | `String` | no | MUST equal a submitted `instructionId`. |
| | `severity` | `Severity` | no | Enum value. |
| | `documentSection` | `String` | no | e.g. `"§4.2"`. |
| | `quote` | `String` | no | Short verbatim passage from the document. |
| | `recommendation` | `String` | no | Actionable verb-phrase. |
| `ReviewVerdict` | `decision` | `Decision` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `findings` | `List<Finding>` | no | One entry per submitted instruction. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `EvaluationScorer` finished. |
| `Review` (entity state) | `reviewId` | `String` | no | — |
| | `request` | `Optional<ReviewRequest>` | yes | Populated after `DocumentSubmitted`. |
| | `sanitized` | `Optional<SanitizedDocument>` | yes | Populated after `DocumentSanitized`. |
| | `verdict` | `Optional<ReviewVerdict>` | yes | Populated after `VerdictRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `ReviewStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DocumentSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Review` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Severity`: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.
`Decision`: `PASS`, `NEEDS_REVISION`, `FAIL`.
`ReviewStatus`: `SUBMITTED`, `SANITIZED`, `REVIEWING`, `VERDICT_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`ReviewEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentSubmitted` | `request` | → SUBMITTED |
| `DocumentSanitized` | `sanitized` | → SANITIZED |
| `ReviewStarted` | — | → REVIEWING |
| `VerdictRecorded` | `verdict` | → VERDICT_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `ReviewFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Review.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ReviewRow` mirrors `Review` minus `request.rawDocument` (the audit log keeps that). The UI fetches the raw document on demand via `GET /api/reviews/{id}` and reads `request.rawDocument` from the JSON.

The view declares ONE query: `getAllReviews: SELECT * AS reviews FROM review_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ReviewTasks.java`)

```java
public final class ReviewTasks {
  public static final Task<ReviewVerdict> REVIEW_DOCUMENT = Task
      .name("Review document")
      .description("Read the attached sanitized document and produce a ReviewVerdict per submitted instruction")
      .resultConformsTo(ReviewVerdict.class);

  private ReviewTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
