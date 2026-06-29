# Data model — score-aggregator

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SkillMatch` | `skill` | `String` | no | Name of the required skill from the role requirements. |
| | `matched` | `boolean` | no | Whether the candidate's resume demonstrates this skill. |
| | `evidence` | `String` | no | Quoted passage from the resume or requirements supporting the match decision. |
| `ScreenResult` | `candidateId` | `String` | no | Slug derived from candidateName (e.g. `alice-chen`). |
| | `role` | `String` | no | Target role slug (e.g. `staff-software-engineer`). |
| | `passedInitialScreen` | `boolean` | no | `true` only when candidate meets stated minimum requirements. |
| | `skillMatches` | `List<SkillMatch>` | no | One entry per required skill from the role. |
| | `fitSummary` | `String` | no | 1–2 sentence assessment of overall fit. |
| | `screenedAt` | `Instant` | no | When `ScreeningAgent` completed the SCREEN_RESUME task. |
| `DimensionScore` | `dimension` | `String` | no | One of: `skills-match`, `experience-years`, `role-seniority`, `location-preference`, `screen-flag`. |
| | `score` | `int` | no | Score for this dimension (range varies by dimension; see `ScoreAggregator`). |
| | `rationale` | `String` | no | One sentence explaining the score. |
| `CandidateScore` | `totalScore` | `int` | no | Sum of dimension scores, capped at 100. Range: [0, 100]. |
| | `dimensionScores` | `List<DimensionScore>` | no | Exactly five entries, one per dimension. |
| | `scoredAt` | `Instant` | no | When `ScoreAggregator.score(...)` completed. |
| `Recommendation` | `decision` | `Decision` | no | `ADVANCE`, `HOLD`, or `REJECT`. |
| | `rationale` | `String` | no | One sentence explaining the primary factor. |
| | `recommendedAt` | `Instant` | no | When `ScreeningAgent` completed the GENERATE_RECOMMENDATION task. |
| `AggregatedScore` | `score` | `CandidateScore` | no | The `CandidateScore` produced by `ScoreAggregator`. |
| | `recommendation` | `Recommendation` | no | The `Recommendation` produced by `ScreeningAgent`. |
| `StatusUpdate` | `applicationId` | `String` | no | Matches `ApplicationRecord.applicationId`. |
| | `decision` | `Decision` | no | MUST equal `AggregatedScore.recommendation.decision`. |
| | `score` | `int` | no | MUST equal `AggregatedScore.score.totalScore`. |
| | `notificationPayload` | `String` | no | JSON-formatted string suitable for downstream HR system consumption. Non-empty. |
| | `notifiedAt` | `Instant` | no | When `StatusNotifier.notify(...)` completed. |
| `GateResult` | `pass` | `boolean` | no | `true` when all four QualityGate checks pass. |
| | `reason` | `String` | no | `"all checks passed"` or a sentence naming the first failing check. |
| | `evaluatedAt` | `Instant` | no | When `QualityGate.evaluate(...)` completed. |
| `RoleRequirements` | `role` | `String` | no | Role slug. |
| | `requiredSkills` | `List<String>` | no | Canonical skill list for this role. |
| | `experienceFloorYears` | `int` | no | Minimum years of experience required. |
| | `seniorityLevel` | `String` | no | e.g. `staff`, `senior`, `mid`. |
| | `locationPreference` | `String` | no | e.g. `remote`, `hybrid`, `onsite`. |
| `SkillMatchResult` | `matchedRatio` | `double` | no | `matchedSkills.size() / requiredSkills.size()`. |
| | `matches` | `List<SkillMatch>` | no | Per-skill match evidence. |
| `CandidateHistory` | `candidateId` | `String` | no | Slug. |
| | `priorRoles` | `List<String>` | no | Previous job titles. May be empty. |
| | `yearsExperience` | `int` | no | Total years of professional experience. |
| `MarketBenchmark` | `role` | `String` | no | Role slug. |
| | `demandLevel` | `String` | no | `high`, `medium`, or `low`. |
| | `supplyLevel` | `String` | no | `high`, `medium`, or `low`. |
| `ApplicationRecord` (entity state) | `applicationId` | `String` | no | — |
| | `candidateName` | `Optional<String>` | yes | Populated after `ApplicationCreated`. |
| | `resumeText` | `Optional<String>` | yes | Populated after `ApplicationCreated`. |
| | `targetRole` | `Optional<String>` | yes | Populated after `ApplicationCreated`. |
| | `screenResult` | `Optional<ScreenResult>` | yes | Populated after `ScreeningCompleted`. |
| | `candidateScore` | `Optional<CandidateScore>` | yes | Populated after `ScoringCompleted`. |
| | `recommendation` | `Optional<Recommendation>` | yes | Populated after `RecommendationCompleted`. |
| | `aggregatedScore` | `Optional<AggregatedScore>` | yes | Populated after `RecommendationCompleted`. |
| | `statusUpdate` | `Optional<StatusUpdate>` | yes | Populated after `NotificationRecorded`. |
| | `gateResult` | `Optional<GateResult>` | yes | Populated after `GateResultRecorded`. |
| | `status` | `ApplicationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ApplicationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `ApplicationRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ApplicationStatus`: `CREATED`, `SCREENING`, `SCREENED`, `SCORING`, `SCORED`, `RECOMMENDING`, `RECOMMENDED`, `NOTIFYING`, `NOTIFIED`, `FAILED`.

`Decision`: `ADVANCE`, `HOLD`, `REJECT`.

## Events (`ApplicationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ApplicationCreated` | `candidateName, resumeText, targetRole` | → CREATED |
| `ScreeningStarted` | — | → SCREENING |
| `ScreeningCompleted` | `screenResult: ScreenResult` | → SCREENED |
| `ScoringStarted` | — | → SCORING |
| `ScoringCompleted` | `candidateScore: CandidateScore` | → SCORED |
| `RecommendationStarted` | — | → RECOMMENDING |
| `RecommendationCompleted` | `recommendation: Recommendation` | → RECOMMENDED |
| `NotificationRecorded` | `statusUpdate: StatusUpdate` | → NOTIFIED |
| `GateResultRecorded` | `gateResult: GateResult` | no status change (annotation) |
| `ApplicationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ApplicationRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ApplicationRow` mirrors `ApplicationRecord` exactly. The UI fetches the full row via `GET /api/applications/{id}` and streams updates via `GET /api/applications/sse`.

The view declares ONE query: `getAllApplications: SELECT * AS applications FROM application_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`ScreeningTasks.java`)

```java
public final class ScreeningTasks {
  public static final Task<ScreenResult> SCREEN_RESUME = Task
      .name("Screen resume")
      .description("Assess the candidate resume against the target role using lookupRoleRequirements and checkSkillMatch")
      .resultConformsTo(ScreenResult.class);

  public static final Task<Recommendation> GENERATE_RECOMMENDATION = Task
      .name("Generate recommendation")
      .description("Produce a hiring recommendation (ADVANCE / HOLD / REJECT) with a one-sentence rationale, informed by the score and screening results")
      .resultConformsTo(Recommendation.class);

  private ScreeningTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## ScoreAggregator dimension weights

| Dimension key | Max score | Input |
|---|---|---|
| `skills-match` | 30 | `SkillMatchResult.matchedRatio × 30`, rounded |
| `experience-years` | 25 | `min(experienceYears / experienceFloorYears, 1.0) × 25`, rounded |
| `role-seniority` | 20 | 20 if seniority matches, 10 if one level off, 0 otherwise |
| `location-preference` | 15 | 15 if preference matches role, 7 if hybrid/partial match, 0 otherwise |
| `screen-flag` | 10 | 10 if `passedInitialScreen`, 0 otherwise |

`totalScore = min(sum, 100)`. All five dimensions are always present in `CandidateScore.dimensionScores`.
