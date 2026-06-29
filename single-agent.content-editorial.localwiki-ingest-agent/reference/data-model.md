# Data model — localwiki-ingest-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SourceSubmission` | `ingestId` | `String` | no | UUID minted by `IngestEndpoint`. |
| | `sourceType` | `SourceType` | no | Enum: URL, IMAGE, PDF. |
| | `sourceAddress` | `String` | no | URL, image path, or PDF path submitted by the user. Audit-only after sanitization. |
| | `targetNamespace` | `String` | no | Deployer-declared namespace: `/docs`, `/blog`, `/reference`, or custom. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedContent` | `redactedText` | `String` | no | PII redacted; this is what the agent sees. |
| | `mimeType` | `String` | no | Detected MIME type of the source (e.g., `text/html`, `image/png`, `application/pdf`). |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email", "phone", "ssn", "payment-card-number", "address", "person-name"]`. |
| `WikiPage` | `title` | `String` | no | Descriptive title in title case; 5–10 words. |
| | `slug` | `String` | no | Kebab-case slug derived from the title. |
| | `summary` | `String` | no | 1–2 sentence plain-prose summary. |
| | `body` | `String` | no | Markdown content with `##` headings; 100–400 words. |
| | `categoryTags` | `List<String>` | no | 2–4 lowercase topic tags. |
| | `targetPath` | `String` | no | Full path: `<namespace>/<slug>`, e.g. `/docs/introduction-to-akka`. |
| | `filedAt` | `Instant` | no | When the agent returned the page. |
| `Ingest` (entity state) | `ingestId` | `String` | no | — |
| | `submission` | `Optional<SourceSubmission>` | yes | Populated after `SourceSubmitted`. |
| | `sanitized` | `Optional<SanitizedContent>` | yes | Populated after `ContentSanitized`. |
| | `page` | `Optional<WikiPage>` | yes | Populated after `PageFiled`. |
| | `status` | `IngestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SourceSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `failureReason` | `Optional<String>` | yes | Populated after `IngestFailed`. |

Every nullable field on `Ingest` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SourceType`: `URL`, `IMAGE`, `PDF`.
`IngestStatus`: `SUBMITTED`, `SANITIZED`, `INGESTING`, `PAGE_FILED`, `FAILED`.

## Events (`IngestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SourceSubmitted` | `submission` | → SUBMITTED |
| `ContentSanitized` | `sanitized` | → SANITIZED |
| `IngestStarted` | — | → INGESTING |
| `PageFiled` | `page` | → PAGE_FILED (terminal happy) |
| `IngestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Ingest.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`IngestRow` mirrors `Ingest` minus `submission.sourceAddress` raw bytes — the audit log keeps that. The UI fetches the raw submission on demand via `GET /api/ingests/{id}` and reads `submission.sourceAddress` from the JSON.

The view declares ONE query: `getAllIngests: SELECT * AS ingests FROM wiki_page_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`IngestTasks.java`)

```java
public final class IngestTasks {
  public static final Task<WikiPage> INGEST_SOURCE = Task
      .name("Ingest source")
      .description("Read the attached source content and produce a WikiPage filed to the target namespace")
      .resultConformsTo(WikiPage.class);

  private IngestTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
