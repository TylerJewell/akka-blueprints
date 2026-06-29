# Data model — pokemon-advisor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RosterSlot` | `slotNumber` | `int` | no | Position in the team (1–6). |
| | `species` | `String` | no | Pokemon species name (case-preserved, trimmed). |
| | `nickname` | `String` | no | Trainer-given name; empty string if none. |
| `RosterSubmission` | `advisoryId` | `String` | no | UUID minted by `AdvisoryEndpoint`. |
| | `trainerName` | `String` | no | User-supplied trainer identifier. |
| | `slots` | `List<RosterSlot>` | no | Submitted roster (1–6 slots). |
| | `format` | `BattleFormat` | no | Enum value. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ValidatedRoster` | `slots` | `List<RosterSlot>` | no | Slots after legality check (same content, confirmed valid). |
| | `format` | `BattleFormat` | no | Matches submission. |
| | `warningsFound` | `List<String>` | no | Non-fatal warnings (e.g., legendary usage note). Empty list if none. |
| `SlotRecommendation` | `slotNumber` | `int` | no | MUST equal a slot number in the submitted roster. |
| | `species` | `String` | no | MUST equal the species in that slot. |
| | `role` | `Role` | no | Enum value. |
| | `suggestedMoves` | `List<String>` | no | 1–4 move names. |
| | `rationale` | `String` | no | One-sentence role justification. |
| `CoverageGap` | `type` | `String` | no | Damage type name (e.g., `"Water"`). |
| | `severity` | `GapSeverity` | no | Enum value. |
| | `suggestion` | `String` | no | Concrete recommendation to address the gap. |
| `TeamRecommendation` | `verdict` | `Verdict` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `slots` | `List<SlotRecommendation>` | no | One entry per submitted slot. |
| | `gaps` | `List<CoverageGap>` | no | Zero or more coverage holes. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `CoverageScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `scoredAt` | `Instant` | no | When `CoverageScorer` finished. |
| `Advisory` (entity state) | `advisoryId` | `String` | no | — |
| | `submission` | `Optional<RosterSubmission>` | yes | Populated after `RosterSubmitted`. |
| | `validated` | `Optional<ValidatedRoster>` | yes | Populated after `RosterValidated`. |
| | `recommendation` | `Optional<TeamRecommendation>` | yes | Populated after `RecommendationRecorded`. |
| | `coverageScore` | `Optional<CoverageScore>` | yes | Populated after `CoverageScored`. |
| | `status` | `AdvisoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RosterSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Advisory` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BattleFormat`: `VGC`, `SINGLES`, `DOUBLE_BATTLE`, `CASUAL`.
`Role`: `PHYSICAL_SWEEPER`, `SPECIAL_SWEEPER`, `WALL`, `SUPPORT`, `LEAD`, `UTILITY`.
`GapSeverity`: `MINOR`, `MODERATE`, `SIGNIFICANT`.
`Verdict`: `STRONG`, `VIABLE`, `NEEDS_ADJUSTMENT`.
`AdvisoryStatus`: `SUBMITTED`, `VALIDATED`, `ADVISING`, `RECOMMENDATION_RECORDED`, `SCORED`, `FAILED`.

## Events (`RosterEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RosterSubmitted` | `submission` | → SUBMITTED |
| `RosterValidated` | `validated` | → VALIDATED |
| `AdvisingStarted` | — | → ADVISING |
| `RecommendationRecorded` | `recommendation` | → RECOMMENDATION_RECORDED |
| `CoverageScored` | `coverageScore` | → SCORED (terminal happy) |
| `AdvisoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Advisory.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AdvisoryRow` mirrors `Advisory`. No fields are excluded — the roster submission contains no audit-sensitive raw data (unlike the docreview pattern where `rawDocument` is excluded from the view row).

The view declares ONE query: `getAllAdvisories: SELECT * AS advisories FROM advisory_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AdvisoryTasks.java`)

```java
public final class AdvisoryTasks {
  public static final Task<TeamRecommendation> ADVISE_TEAM = Task
      .name("Advise team")
      .description("Read the attached roster and produce a TeamRecommendation with slot rationale and coverage gaps")
      .resultConformsTo(TeamRecommendation.class);

  private AdvisoryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
