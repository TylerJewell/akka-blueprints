# Data model — Game Builder Team

Authoritative records, events, and the status enum the generated system defines. Lifecycle fields that are null before their transition are declared `Optional<T>` (Lesson 6).

## `Build` record — BuildEntity state and BuildsView row

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Build id (UUID) |
| `brief` | `Optional<String>` | yes | The one-line game brief |
| `status` | `BuildStatus` | no | Lifecycle status |
| `engineeredAt` | `Optional<Instant>` | yes | When the engineer produced code |
| `filename` | `Optional<String>` | yes | Generated `.py` filename |
| `sourceCode` | `Optional<String>` | yes | Generated program text |
| `qaAt` | `Optional<Instant>` | yes | When QA completed |
| `qaPassed` | `Optional<Boolean>` | yes | Whether tests passed |
| `qaScore` | `Optional<Integer>` | yes | QA quality score 0–100 |
| `qaNotes` | `Optional<String>` | yes | One-line QA note |
| `chiefReviewedAt` | `Optional<Instant>` | yes | When the chief reviewed |
| `shipped` | `Optional<Boolean>` | yes | Final ship decision |
| `shipSummary` | `Optional<String>` | yes | One-line ship rationale |
| `reworkReason` | `Optional<String>` | yes | Why the build was rejected |

`emptyState()` returns `Build.initial("")` (status `QUEUED`, all `Optional` fields empty) and never references `commandContext()` (Lesson 3).

## `BuildStatus` enum

`QUEUED`, `ENGINEERED`, `QA`, `SHIPPED`, `REJECTED`.

## Typed agent results

| Record | Fields | Returned by |
|---|---|---|
| `GameCode` | `String filename`, `String source` | EngineerAgent |
| `QaReport` | `boolean passed`, `int score`, `String notes` | QaAgent |
| `ShipDecision` | `boolean shipped`, `String summary` | ChiefQaAgent |

These are the `resultConformsTo` types declared as `Task<R>` constants in `BuildTasks.java`: `ENGINEER` → `GameCode`, `QA` → `QaReport`, `CHIEF` → `ShipDecision` (Lesson 7).

## Events

### BuildEntity

| Event | Trigger | Effect on state |
|---|---|---|
| `GameEngineered` | `recordEngineered` after engineerStep | sets `engineeredAt`, `filename`, `sourceCode`; status → `ENGINEERED` |
| `QaCompleted` | `recordQa` after qaStep | sets `qaAt`, `qaPassed`, `qaScore`, `qaNotes`; status → `QA` |
| `BuildShipped` | `ship` when QA passed and chief ships | sets `chiefReviewedAt`, `shipped: true`, `shipSummary`; status → `SHIPPED` |
| `BuildRejected` | `reject` when QA failed or chief reworks | sets `chiefReviewedAt`, `shipped: false`, `reworkReason`; status → `REJECTED` |

### InboundBriefQueue

| Event | Trigger | Effect |
|---|---|---|
| `BriefQueued` | `enqueueBrief(brief)` | records the inbound brief; `BriefConsumer` reacts and starts a workflow |

## View row type

`BuildsView` uses `Build` as its row type. Single query `getAllBuilds` (`SELECT * AS builds FROM builds_view`) with no `WHERE status` clause; status filtering happens client-side (Lesson 2). Event-applier wraps lifecycle values with `Optional.of(...)` so the materializer accepts the row (Lesson 6).
