# Data model — basic-cv-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WorkHistoryEntry` | `title` | `String` | no | Job title. |
| | `company` | `String` | no | Employer name. |
| | `startDate` | `String` | no | "YYYY-MM" format. |
| | `endDate` | `Optional<String>` | yes | Absent = current role. |
| | `bullets` | `List<String>` | no | Achievement bullets (may be empty list). |
| `EducationEntry` | `degree` | `String` | no | Degree or qualification name. |
| | `institution` | `String` | no | School or university name. |
| | `graduationYear` | `String` | no | Four-digit year. |
| `CvRequest` | `requestId` | `String` | no | UUID minted by `CvEndpoint`. |
| | `candidateFullName` | `String` | no | Display name (not redacted). |
| | `contactEmail` | `String` | no | Raw email. Audit-only; redacted in view. |
| | `targetRole` | `String` | no | User-supplied target job title. |
| | `workHistory` | `List<WorkHistoryEntry>` | no | May be empty list. |
| | `education` | `List<EducationEntry>` | no | May be empty list. |
| | `skills` | `List<String>` | no | Free-form skill tags. |
| | `additionalNotes` | `Optional<String>` | yes | Free-text; PII often present here. |
| | `outputMode` | `OutputMode` | no | PROSE or STRUCTURED. |
| | `submittedBy` | `String` | no | Recruiter identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedProfile` | `redactedContactEmail` | `String` | no | Always `[REDACTED-EMAIL]` after sanitization. |
| | `redactedAdditionalNotes` | `Optional<String>` | yes | Absent if `additionalNotes` was absent. PII markers substituted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","nino","phone"]`. |
| `CvSection` | `sectionTitle` | `String` | no | e.g. `"Work History"`, `"Skills"`. |
| | `content` | `String` | no | Markdown prose (PROSE) or JSON string (STRUCTURED). |
| `GeneratedCv` | `headline` | `String` | no | One-line role title + value proposition (max 12 words). |
| | `sections` | `List<CvSection>` | no | At minimum 4 entries. |
| | `keywords` | `List<String>` | no | 8–15 ATS keywords. |
| | `outputMode` | `OutputMode` | no | Mirrors the request's outputMode. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `CvRequestState` (entity state) | `requestId` | `String` | no | — |
| | `request` | `Optional<CvRequest>` | yes | Populated after `ProfileSubmitted`. |
| | `sanitized` | `Optional<SanitizedProfile>` | yes | Populated after `ProfileSanitized`. |
| | `generatedCv` | `Optional<GeneratedCv>` | yes | Populated after `CvGenerated`. |
| | `status` | `CvStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ProfileSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `CvRequestState` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OutputMode`: `PROSE`, `STRUCTURED`.
`CvStatus`: `SUBMITTED`, `SANITIZED`, `GENERATING`, `CV_GENERATED`, `FAILED`.

## Events (`CvEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ProfileSubmitted` | `request` | → SUBMITTED |
| `ProfileSanitized` | `sanitized` | → SANITIZED |
| `GenerationStarted` | — | → GENERATING |
| `CvGenerated` | `generatedCv` | → CV_GENERATED (terminal happy) |
| `GenerationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CvRequestState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CvRow` mirrors `CvRequestState` minus `request.contactEmail` (raw) and `request.additionalNotes` (raw). The view holds `sanitized.redactedContactEmail` and `sanitized.redactedAdditionalNotes` for the UI. Recruiters who need the raw fields fetch `GET /api/cv-requests/{id}` and read them from the JSON.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM cv_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CvTasks.java`)

```java
public final class CvTasks {
  public static final Task<GeneratedCv> GENERATE_CV = Task
      .name("Generate CV")
      .description("Read the attached sanitized candidate profile and produce a GeneratedCv with headline, named sections, and a keywords list")
      .resultConformsTo(GeneratedCv.class);

  private CvTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
