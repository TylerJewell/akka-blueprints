# Data model

Every record, event, enum, and view row the generated system defines.

## `Document` (DocumentEntity state + DocumentsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Document id; equals the workflow id and entity id |
| `documentUrl` | `Optional<String>` | yes | The submitted PDF document URL |
| `status` | `DocumentStatus` | no | Lifecycle status |
| `confidence` | `Optional<Double>` | yes | Aggregate extraction confidence (0.0–1.0) |
| `submittedAt` | `Optional<Instant>` | yes | When the extraction request was received |
| `extractedAt` | `Optional<Instant>` | yes | When ExtractionAgent completed |
| `rawFields` | `Optional<Map<String,String>>` | yes | Extracted field values before redaction |
| `redactedFields` | `Optional<Map<String,String>>` | yes | PII-scrubbed field values for display |
| `reviewedAt` | `Optional<Instant>` | yes | When the human reviewer decided |
| `reviewedBy` | `Optional<String>` | yes | Reviewer identity |
| `reviewerComment` | `Optional<String>` | yes | Reviewer note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Rejection reason |
| `postedAt` | `Optional<String>` | yes | ISO-8601 timestamp when posted downstream |
| `postTarget` | `Optional<String>` | yes | Identifier or URL of the downstream target |

Every nullable lifecycle field is `Optional<T>` because `Document` is the view row type (Lesson 6). `emptyState()` returns `Document.initial("")` with no `commandContext()` reference (Lesson 3).

## `DocumentStatus` enum

`EXTRACTED`, `PENDING_REVIEW`, `APPROVED`, `REJECTED`, `POSTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `DocumentExtracted` | `recordExtraction` after ExtractionAgent returns | sets `extractedAt`, `rawFields`, `confidence`, status → `EXTRACTED` |
| `DocumentRedacted` | `recordRedaction` after RedactionAgent returns | sets `redactedFields` (status unchanged) |
| `DocumentPendingReview` | `setPendingReview` when confidence < 0.85 | status → `PENDING_REVIEW` |
| `DocumentApproved` | `approve` command (human via API, or auto via workflow) | sets `reviewedAt`, `reviewedBy`, `reviewerComment`, status → `APPROVED` |
| `DocumentRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `DocumentPosted` | `recordPost` after simulated post tool runs | sets `postedAt`, `postTarget`, status → `POSTED` |

## Domain records

- `ExtractionResult(Map<String,String> fields, double confidence)` — ExtractionAgent result.
- `RedactedResult(Map<String,String> fields)` — RedactionAgent result.
- `ReviewDecision(String reviewedBy, String comment)` — approve payload.
- `PostReceipt(String postTarget, String postedAt)` — simulated downstream post result.

## View row type

`DocumentsView` rows are `Document`. One query: `getAllDocuments` → `SELECT * AS documents FROM documents_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ExtractionTasks.java)

- `EXTRACT` — `Task.name(...).resultConformsTo(ExtractionResult.class)`.
- `REDACT` — `Task.name(...).resultConformsTo(RedactedResult.class)`.
