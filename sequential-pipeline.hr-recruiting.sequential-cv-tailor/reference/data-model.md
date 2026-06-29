# Data model — sequential-cv-tailor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RawExperience` | `company` | `String` | no | Employer name. |
| | `title` | `String` | no | Job title. |
| | `period` | `String` | no | Date range string (e.g. `"2021–present"`). |
| | `description` | `String` | no | Free-text experience description; source for bullet extraction. |
| `CandidateProfile` | `profileId` | `String` | no | Fixture identifier. |
| | `candidateName` | `String` | no | Full name — PII; stripped by `PiiSanitizer` before model calls. |
| | `email` | `String` | no | Email address — PII; stripped. |
| | `phone` | `String` | no | Phone number — PII; stripped. |
| | `address` | `Optional<String>` | yes | Home address — PII; stripped if present. |
| | `dateOfBirth` | `Optional<String>` | yes | Date of birth — PII; stripped if present. |
| | `rawSummary` | `String` | no | Unstructured career summary. |
| | `rawSkills` | `List<String>` | no | Unprocessed skill names. |
| | `rawExperience` | `List<RawExperience>` | no | Unstructured experience entries. |
| `Skill` | `name` | `String` | no | Skill name. |
| | `level` | `String` | no | `beginner` / `intermediate` / `expert`. |
| `ExperienceEntry` | `company` | `String` | no | Employer name. |
| | `title` | `String` | no | Job title. |
| | `period` | `String` | no | Date range string. |
| | `bulletPoints` | `String` | no | Newline-separated bullet lines; 2–4 per entry. |
| `BaseCv` | `generatedSummary` | `String` | no | 3–6 sentence professional summary. No PII. |
| | `skills` | `List<Skill>` | no | Structured skills with inferred levels. |
| | `experience` | `List<ExperienceEntry>` | no | Structured experience; possibly empty (J5). |
| | `generatedAt` | `Instant` | no | When `CvGeneratorAgent` returned. |
| `JobPosting` | `postingId` | `String` | no | Fixture identifier. |
| | `title` | `String` | no | Job title. |
| | `company` | `String` | no | Hiring company name. |
| | `description` | `String` | no | Full job description text. |
| | `requiredKeywords` | `List<String>` | no | Keywords the `AlignmentScorer` checks for; must all be present for max score. |
| | `preferredKeywords` | `List<String>` | no | Keywords whose 50%+ coverage adds one point. |
| `KeywordMatch` | `keyword` | `String` | no | The keyword checked. |
| | `required` | `boolean` | no | `true` if from `requiredKeywords`. |
| | `sourceSection` | `Optional<String>` | yes | `"summary"` / `"skills"` / `"experience"` / `null` (not found). |
| `TailoredCv` | `tailoredSummary` | `String` | no | Rewritten summary surfacing required keywords. ≥ 50 chars. |
| | `skills` | `List<Skill>` | no | Carried forward from `BaseCv`, possibly lightly renamed. |
| | `experience` | `List<ExperienceEntry>` | no | Adjusted entries; same count as `BaseCv.experience`. |
| | `keywordMatches` | `List<KeywordMatch>` | no | One entry per required + preferred keyword. |
| | `tailoredAt` | `Instant` | no | When `CvTailorAgent` returned. |
| `AlignmentResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `keywordsCovered` | `List<String>` | no | Required + preferred keywords found in the tailored CV. |
| | `keywordsMissed` | `List<String>` | no | Required + preferred keywords not found. |
| | `scoredAt` | `Instant` | no | When `AlignmentScorer` finished. |
| `CvRequestRecord` (entity state) | `requestId` | `String` | no | — |
| | `candidateProfileId` | `Optional<String>` | yes | Populated after `CvRequestCreated`. |
| | `jobPostingId` | `Optional<String>` | yes | Populated after `CvRequestCreated`. |
| | `baseCv` | `Optional<BaseCv>` | yes | Populated after `BaseCvReady`. |
| | `tailoredCv` | `Optional<TailoredCv>` | yes | Populated after `TailoredCvReady`. |
| | `alignmentResult` | `Optional<AlignmentResult>` | yes | Populated after `QualityScored`. |
| | `sanitizationCount` | `int` | no | Incremented on every `SanitizationApplied` event; 0 initially. |
| | `status` | `CvRequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CvRequestCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `CvRequestRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`CvRequestStatus`: `CREATED`, `GENERATING`, `GENERATED`, `TAILORING`, `TAILORED`, `SCORED`, `FAILED`.

## Events (`CvRequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CvRequestCreated` | `candidateProfileId: String, jobPostingId: String` | → CREATED |
| `GenerationStarted` | — | → GENERATING |
| `BaseCvReady` | `baseCv: BaseCv` | → GENERATED |
| `TailoringStarted` | — | → TAILORING |
| `TailoredCvReady` | `tailoredCv: TailoredCv` | → TAILORED |
| `QualityScored` | `alignmentResult: AlignmentResult` | → SCORED (terminal happy) |
| `SanitizationApplied` | `agent: String, removedCount: int, appliedAt: Instant` | no status change (audit-only; increments `sanitizationCount`) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `CvRequestRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `sanitizationCount = 0`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CvRequestRow` mirrors `CvRequestRecord` exactly. The UI fetches the full row via `GET /api/cv-requests/{id}` and streams updates via `GET /api/cv-requests/sse`.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM cv_request_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`CvTasks.java`)

```java
public final class CvTasks {
  public static final Task<BaseCv> GENERATE_BASE_CV = Task
      .name("Generate base CV")
      .description("Expand and structure a CandidateProfile into a BaseCv")
      .resultConformsTo(BaseCv.class);

  public static final Task<TailoredCv> TAILOR_CV = Task
      .name("Tailor CV")
      .description("Adapt a BaseCv for a specific JobPosting by surfacing required and preferred keywords")
      .resultConformsTo(TailoredCv.class);

  private CvTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7). Both constants live in the same file; `CvGeneratorAgent` references `GENERATE_BASE_CV`, `CvTailorAgent` references `TAILOR_CV`.

## PII fields registry

The fields stripped by `PiiSanitizer` are checked by name (case-insensitive) at every nesting level of the payload:

| Field name | Record it appears in | Category |
|---|---|---|
| `candidateName` | `CandidateProfile` | identity |
| `email` | `CandidateProfile` | contact |
| `phone` | `CandidateProfile` | contact |
| `address` | `CandidateProfile` | location |
| `dateOfBirth` | `CandidateProfile` | demographic |

Any future field added to `CandidateProfile` or `BaseCv` that falls into these categories must be added to the registry — the sanitizer reads the registry at startup, not the record schema at runtime.
