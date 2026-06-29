# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## Document (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `documentId` | `String` | no | UUID, also the workflow id |
| `batchId` | `String` | no | Parent batch UUID |
| `rawContent` | `String` | no | Raw document text as submitted; omitted from view row |
| `status` | `DocumentStatus` | no | Lifecycle state |
| `fields` | `Optional<ExtractedFields>` | yes | Extractor output; null until `FieldsExtracted` |
| `classification` | `Optional<DocumentClassification>` | yes | Classifier output; null until `DocumentClassified` |
| `processed` | `Optional<ProcessedDocument>` | yes | Merged + sanitized record; null until `DocumentProcessed` |
| `failureReason` | `Optional<String>` | yes | Set on `DocumentDegraded` |
| `qualityScore` | `Optional<Integer>` | yes | 1–5; null until `QualityScored` |
| `qualityRationale` | `Optional<String>` | yes | One-line quality justification |
| `createdAt` | `Instant` | no | Document creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `DocumentView` row type mirrors this with `Optional<T>` on every nullable field. `rawContent` is excluded from the view row to avoid bloating the read model.

## Batch (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `batchId` | `String` | no | UUID |
| `submittedBy` | `String` | no | Submitter identifier |
| `status` | `BatchStatus` | no | Aggregate batch lifecycle state |
| `totalDocuments` | `int` | no | Count of documents in the submission |
| `processedCount` | `int` | no | Count of documents that reached `PROCESSED` |
| `degradedCount` | `int` | no | Count of documents that reached `DEGRADED` |
| `createdAt` | `Instant` | no | Batch creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set when `processedCount + degradedCount == totalDocuments` |

## Supporting records

```java
record BatchSubmission(List<String> rawDocuments, String submittedBy) {}

record WorkPartition(String extractionInstruction, String classificationInstruction) {}

record ExtractedFields(
    Optional<String> documentDate,
    Optional<String> author,
    Optional<String> referenceNumber,
    String bodyText,
    Instant extractedAt
) {}

record DocumentClassification(
    RiskCategory riskCategory,
    SensitivityLevel sensitivityLevel,
    List<String> flaggedTopics,
    Instant classifiedAt
) {}

record PiiRedaction(String fieldName, String piiType, String replacement) {}

record ProcessedDocument(
    ExtractedFields fields,
    DocumentClassification classification,
    String sanitizedText,
    List<PiiRedaction> piiRedactions,
    Instant processedAt
) {}
```

## Status enums

```java
enum DocumentStatus  { QUEUED, PROCESSING, PROCESSED, DEGRADED }
enum BatchStatus     { PENDING, IN_PROGRESS, COMPLETE, PARTIAL }
enum RiskCategory    { LOW, MEDIUM, HIGH, CRITICAL }
enum SensitivityLevel { PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED }
```

## Events

### DocumentEntity

| Event | Trigger |
|---|---|
| `DocumentQueued` | Workflow calls `queueDocument` |
| `DocumentProcessingStarted` | Workflow calls `startProcessing` after partition |
| `FieldsExtracted` | Extractor returns an `ExtractedFields` |
| `DocumentClassified` | Classifier returns a `DocumentClassification` |
| `DocumentProcessed` | Coordinator merge + sanitize passes; workflow calls `markProcessed` |
| `DocumentDegraded` | A worker timed out; merge ran on partial input |
| `QualityScored` | `QualitySampler` recorded a 1–5 score |

### BatchEntity

| Event | Trigger |
|---|---|
| `BatchCreated` | `BatchRequestConsumer` calls `createBatch` |
| `BatchStarted` | First document workflow starts; status → `IN_PROGRESS` |
| `BatchDocumentCompleted` | Each document resolves (PROCESSED or DEGRADED); `documentDone` command |
| `BatchComplete` | `degradedCount == 0` and all documents resolved |
| `BatchPartiallyComplete` | `degradedCount > 0` and all documents resolved |

### BatchQueue

| Event | Trigger |
|---|---|
| `BatchSubmitted` | `submitBatch(rawDocuments, submittedBy)` from endpoint or simulator |

Fields: `{ batchId, submittedBy, documentCount, submittedAt }`.
