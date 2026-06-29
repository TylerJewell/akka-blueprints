# Data model — story-refiner

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `StoryBrief` | `featureDescription` | `String` | no | The user-submitted feature description. |
| | `targetTeam` | `String` | no | Team that will implement the story. |
| | `storyTypeHint` | `Optional<String>` | yes | Optional hint (e.g., "epic", "task"). |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `StoryDraft` | `role` | `String` | no | The "As a …" persona clause. |
| | `goal` | `String` | no | The "I want to …" single-intent clause. |
| | `benefit` | `String` | no | The "so that …" outcome clause. |
| | `acceptanceCriteria` | `List<String>` | no | 2–5 independently testable conditions. |
| | `fieldCount` | `int` | no | Count of non-empty fields (always 4 when all present). |
| | `draftedAt` | `Instant` | no | When the Drafter returned the draft. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–4 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Review` | `verdict` | `ReviewVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric minimum across five dimensions. |
| | `reviewedAt` | `Instant` | no | When the Reviewer returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `draft` | `StoryDraft` | no | The Drafter's output for this attempt. |
| | `review` | `Optional<Review>` | yes | Empty while reviewing; populated once the Reviewer returns. |
| `Story` (entity state) | `storyId` | `String` | no | Unique id. |
| | `featureDescription` | `String` | no | Original feature description. |
| | `targetTeam` | `String` | no | Implementing team. |
| | `maxAttempts` | `int` | no | Per-story retry ceiling (default 4). |
| | `status` | `StoryStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `StoryApproved`. |
| | `approvedDraft` | `Optional<StoryDraft>` | yes | Populated on `StoryApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `StoryRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `StoryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the story reached a terminal state. |

## Enums

`StoryStatus`: `DRAFTING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewVerdict`: `APPROVE`, `REVISE`.

## Events (`StoryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `StoryCreated` | `storyId, featureDescription, targetTeam, maxAttempts, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `AttemptDrafted` | `attemptNumber, draft: StoryDraft` | After `draftStep` returns | → `REVIEWING` (status change on first draft of each cycle) |
| `AttemptReviewed` | `attemptNumber, review: Review` | After `reviewStep` returns | `APPROVE` → `APPROVED`; `REVISE` + attempts < max → `DRAFTING`; `REVISE` + attempts = max → `REJECTED_FINAL` |
| `StoryApproved` | `attemptNumber, approvedDraft: StoryDraft` | `Review.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `StoryRejectedFinal` | `bestAttemptNumber, bestDraft: StoryDraft, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, fieldScores: Map<String,Integer>, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; contributes to `evalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `storyId, featureDescription, targetTeam, storyTypeHint, requestedBy, submittedAt` |

## View row

`StoryRow` is structurally identical to `Story` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `StoryEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllStories` returning the full list. Callers filter by `status` client-side.
