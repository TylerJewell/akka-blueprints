# Data model — schema-extractor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `FieldSchema` | `fieldName` | `String` | no | Stable name matching the document's field. |
| | `fieldType` | `String` | no | One of `string`, `number`, `boolean`, `date`. |
| | `required` | `boolean` | no | Whether the field must appear in the extracted record. |
| | `maxLength` | `int` | no | Maximum character length for string fields; 0 = no limit. |
| `TargetSchema` | `schemaId` | `String` | no | Stable id for the schema (e.g. `invoice-schema`). |
| | `schemaName` | `String` | no | Human-readable label. |
| | `fields` | `List<FieldSchema>` | no | Ordered list of fields to extract. |
| `ExtractionRequest` | `jobId` | `String` | no | UUID minted by `ExtractionEndpoint`. |
| | `documentTitle` | `String` | no | User-supplied label. |
| | `rawDocument` | `String` | no | Pre-sanitization document body. Audit-only. |
| | `targetSchema` | `TargetSchema` | no | The schema to extract against. |
| | `submittedBy` | `String` | no | User or pipeline identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedDocument` | `redactedDocument` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","address","payment-card-number"]`. |
| `ExtractedField` | `fieldName` | `String` | no | MUST equal a `fieldName` in the submitted `TargetSchema`. |
| | `fieldType` | `String` | no | MUST match the declared `fieldType`. |
| | `value` | `String` | no | Extracted value as string; `""` when `confidence = ABSENT`. |
| | `confidence` | `FieldConfidence` | no | Enum value. |
| `ExtractedRecord` | `schemaId` | `String` | no | MUST match the submitted `TargetSchema.schemaId`. |
| | `fields` | `List<ExtractedField>` | no | One entry per field in the TargetSchema. |
| | `schemaCoveragePercent` | `int` | no | Required fields with non-ABSENT confidence / total required × 100. |
| | `extractedAt` | `Instant` | no | When the agent returned. |
| `ExtractionJob` (entity state) | `jobId` | `String` | no | — |
| | `request` | `Optional<ExtractionRequest>` | yes | Populated after `DocumentSubmitted`. |
| | `sanitized` | `Optional<SanitizedDocument>` | yes | Populated after `DocumentSanitized`. |
| | `record` | `Optional<ExtractedRecord>` | yes | Populated after `RecordExtracted`. |
| | `status` | `ExtractionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DocumentSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ExtractionJob` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`FieldConfidence`: `HIGH`, `MEDIUM`, `LOW`, `ABSENT`.
`ExtractionStatus`: `SUBMITTED`, `SANITIZED`, `EXTRACTING`, `RECORD_EXTRACTED`, `FAILED`.

## Events (`ExtractionJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentSubmitted` | `request` | → SUBMITTED |
| `DocumentSanitized` | `sanitized` | → SANITIZED |
| `ExtractionStarted` | — | → EXTRACTING |
| `RecordExtracted` | `record` | → RECORD_EXTRACTED (terminal happy) |
| `ExtractionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ExtractionJob.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ExtractionRow` mirrors `ExtractionJob` minus `request.rawDocument` (the audit log keeps that). The UI fetches the raw document on demand via `GET /api/jobs/{id}` and reads `request.rawDocument` from the JSON.

The view declares ONE query: `getAllJobs: SELECT * AS jobs FROM extraction_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ExtractionTasks.java`)

```java
public final class ExtractionTasks {
  public static final Task<ExtractedRecord> EXTRACT_RECORD = Task
      .name("Extract record")
      .description("Read the attached sanitized document and produce an ExtractedRecord matching the target schema")
      .resultConformsTo(ExtractedRecord.class);

  private ExtractionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
