# Data model — hr-shortlister

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Profile` | `applicantName` | `String` | no | Full name extracted from resume. |
| | `email` | `String` | yes | Email address extracted from resume. `null` if absent. |
| | `resumeText` | `String` | no | The original resume text submitted by the recruiter. |
| | `skills` | `List<String>` | no | Deduplicated skill names stated or clearly implied in the resume. |
| | `experienceYears` | `int` | no | Total years of professional experience. 0 if not determinable. |
| | `highestEducation` | `String` | yes | Normalized highest education level. `null` if absent. |
| | `currentTitle` | `String` | yes | Most recent job title. `null` if absent. |
| | `age` | `String` | yes | `"[REDACTED]"` after sanitization; original value if PARSE phase, before sanitizer fires. |
| | `gender` | `String` | yes | `"[REDACTED]"` after sanitization; `null` if not present in resume. |
| | `nationality` | `String` | yes | `"[REDACTED]"` after sanitization; `null` if not present in resume. |
| | `disabilityIndicator` | `String` | yes | `"[REDACTED]"` after sanitization; `null` if not present in resume. |
| `CriterionScore` | `criterionId` | `String` | no | Stable id from the job's criteria config. |
| | `criterionName` | `String` | no | Human-readable criterion name. |
| | `score` | `int` | no | 0–10. |
| | `justification` | `String` | no | 1–2 sentences. No protected-attribute references. |
| `CandidateScore` | `criterionScores` | `List<CriterionScore>` | no | One entry per criterion in the job config. |
| | `overallScore` | `int` | no | Weighted average, 0–100. |
| | `scoredAt` | `Instant` | no | When the SCORE task returned. |
| `ShortlistDecision` | `decision` | `Decision` | no | `SHORTLIST` / `HOLD` / `REJECT`. |
| | `rationale` | `String` | no | 1–2 sentences citing scoring highlights. |
| | `confidenceScore` | `int` | no | 0–100. Agent's estimate of how reliably the profile mapped to the criteria. |
| | `decidedAt` | `Instant` | no | When the DECIDE task returned. |
| `RecruiterDecision` | `recruiterId` | `String` | no | Identifier of the recruiter who acted. |
| | `decision` | `Decision` | no | Final decision (may differ from agent decision on override). |
| | `overrideRationale` | `String` | yes | `null` if approving; non-empty if overriding. |
| | `decidedAt` | `Instant` | no | When the recruiter acted. |
| `RedactionRecord` | `fieldName` | `String` | no | `age` / `gender` / `nationality` / `disabilityIndicator`. |
| | `originalPresence` | `String` | no | `"present"` (field had a non-null value that was redacted). |
| | `redactedAt` | `Instant` | no | When `SpecialCategoryGuardrail` fired. |
| `ApplicationRecord` (entity state) | `applicationId` | `String` | no | — |
| | `jobId` | `Optional<String>` | yes | Populated after `ResumeReceived`. |
| | `profile` | `Optional<Profile>` | yes | Populated after `ProfileParsed`. Contains `[REDACTED]` values for sanitized fields. |
| | `score` | `Optional<CandidateScore>` | yes | Populated after `ScoreAssigned`. |
| | `agentDecision` | `Optional<ShortlistDecision>` | yes | Populated after `ShortlistDecided`. |
| | `recruiterDecision` | `Optional<RecruiterDecision>` | yes | Populated after `RecruiterApproved` or `RecruiterOverridden`. |
| | `status` | `ApplicationStatus` | no | See enum. |
| | `redactions` | `List<RedactionRecord>` | no | Appended on every `SpecialCategoryRedacted` event; empty if no protected fields were present. |
| | `receivedAt` | `Instant` | no | When `ResumeReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp (`WRITTEN` or `FAILED`). |
| `DriftReport` | `cohortDimension` | `String` | no | `"gender"` or `"nationality"`. |
| | `cohortA` | `String` | no | First cohort value (e.g. `"M"`). |
| | `cohortB` | `String` | no | Second cohort value (e.g. `"F"`). |
| | `cohortARate` | `double` | no | Shortlisting rate for cohortA (0.0–1.0). |
| | `cohortBRate` | `double` | no | Shortlisting rate for cohortB (0.0–1.0). |
| | `ratio` | `double` | no | `max(cohortARate, cohortBRate) / min(...)`. |
| | `threshold` | `double` | no | Default 1.20. Configurable in `application.conf`. |
| | `breached` | `boolean` | no | `ratio > threshold`. |
| | `computedAt` | `Instant` | no | When `FairnessScorer` ran. |

Every nullable lifecycle field on `ApplicationRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ApplicationStatus`: `RECEIVED`, `PARSING`, `PARSED`, `SCORING`, `SCORED`, `DECIDING`, `DECIDED`, `AWAITING_APPROVAL`, `APPROVED`, `WRITTEN`, `FAILED`.

`Decision`: `SHORTLIST`, `HOLD`, `REJECT`.

`Phase` (used by `@FunctionTool` annotations and `SpecialCategoryGuardrail`): `PARSE`, `SCORE`, `DECIDE`.

## Events (`ApplicationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ResumeReceived` | `jobId: String, resumeText: String` | → RECEIVED |
| `ParseStarted` | — | → PARSING |
| `ProfileParsed` | `profile: Profile` | → PARSED |
| `ScoreStarted` | — | → SCORING |
| `ScoreAssigned` | `score: CandidateScore` | → SCORED |
| `DecisionStarted` | — | → DECIDING |
| `ShortlistDecided` | `agentDecision: ShortlistDecision` | → DECIDED |
| `ApprovalRequested` | — | → AWAITING_APPROVAL |
| `RecruiterApproved` | `recruiterDecision: RecruiterDecision` | → APPROVED |
| `RecruiterOverridden` | `recruiterDecision: RecruiterDecision` | → APPROVED |
| `WrittenToErpNext` | `erpNextId: String` | → WRITTEN (terminal happy) |
| `SpecialCategoryRedacted` | `fieldName, redactedAt` | no status change (audit-only) |
| `ApplicationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ApplicationRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `redactions = List.of()`, and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ApplicationRow` mirrors `ApplicationRecord` exactly. The UI fetches the full row via `GET /api/applications/{id}` and streams updates via `GET /api/applications/sse`.

The view declares two queries:
- `getAllApplications: SELECT * AS applications FROM application_view` — no `WHERE status` filter (Lesson 2); caller filters client-side.
- `getApplicationsInWindow(from Instant, to Instant): SELECT * AS applications FROM application_view WHERE receivedAt >= :from AND receivedAt <= :to` — used by `FairnessDriftWorkflow` only.

## Task definitions (`ShortlistTasks.java`)

```java
public final class ShortlistTasks {
  public static final Task<Profile> PARSE_RESUME = Task
      .name("Parse resume")
      .description("Extract a structured Profile from the resume text using extractProfile and normalizeEducation")
      .resultConformsTo(Profile.class);

  public static final Task<CandidateScore> SCORE_PROFILE = Task
      .name("Score profile")
      .description("Evaluate the sanitized Profile against job criteria using evaluateCriteria and computeOverallScore")
      .resultConformsTo(CandidateScore.class);

  public static final Task<ShortlistDecision> DECIDE_SHORTLIST = Task
      .name("Decide shortlist")
      .description("Classify the application as SHORTLIST, HOLD, or REJECT using classifyDecision and generateRationale")
      .resultConformsTo(ShortlistDecision.class);

  private ShortlistTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ParseTools`, `ScoreTools`, and `ShortlistTools` carries a `Phase` constant. `SpecialCategoryGuardrail` reads this constant; it fires only for `Phase.SCORE` calls and always passes (sanitize-then-accept). The tool registry is built once at startup.
