# Data model — brand-presentation-builder

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PresentationBrief` | `topic` | `String` | no | The user-submitted topic. |
| | `targetAudience` | `String` | no | Intended audience for the deck. |
| | `slideCount` | `int` | no | Number of slides to produce. |
| | `wordsPerSlide` | `int` | no | Per-slide hard word cap. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `Slide` | `slideNumber` | `int` | no | 1-indexed position in the deck. |
| | `title` | `String` | no | Short noun-phrase heading for the slide. |
| | `body` | `String` | no | Prose content of the slide. |
| | `wordCount` | `int` | no | Word count of `body`. |
| `SlideSet` | `slides` | `List<Slide>` | no | Ordered list of slides; bounded at `slideCount`. |
| | `totalWordCount` | `int` | no | Sum of all `slide.wordCount` values. |
| | `builtAt` | `Instant` | no | When the Builder returned the slide set. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted all slides. |
| | `reasonCode` | `String` | no | `OK`, `OVER_WORD_LIMIT`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail naming offending slides; empty when `passed=true`. |
| `BrandFeedback` | `bullets` | `List<String>` | no | 0–5 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `BrandReview` | `verdict` | `ReviewVerdict` | no | `APPROVE` or `REVISE`. |
| | `feedback` | `BrandFeedback` | no | See above. |
| | `score` | `int` | no | 1–5 rubric minimum across five dimensions. |
| | `reviewedAt` | `Instant` | no | When the Reviewer returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `slideSet` | `SlideSet` | no | The Builder's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `brandReview` | `Optional<BrandReview>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Presentation` (entity state) | `presentationId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original brief. |
| | `targetAudience` | `String` | no | Original audience. |
| | `slideCount` | `int` | no | Target slide count. |
| | `wordsPerSlide` | `int` | no | Per-slide word cap. |
| | `maxAttempts` | `int` | no | Per-presentation retry ceiling (default 4). |
| | `status` | `PresentationStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `PresentationApproved`. |
| | `approvedSlideSet` | `Optional<SlideSet>` | yes | Populated on `PresentationApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `PresentationRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `PresentationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the presentation reached a terminal state. |

## Enums

`PresentationStatus`: `BUILDING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewVerdict`: `APPROVE`, `REVISE`.

## Events (`PresentationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `PresentationCreated` | `topic, targetAudience, slideCount, wordsPerSlide, maxAttempts, createdAt` | Workflow `startStep` | → `BUILDING` |
| `AttemptBuilt` | `attemptNumber, slideSet: SlideSet` | After `buildStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `REVIEWING`; `passed=false` → `BUILDING` (re-build) |
| `AttemptBrandReviewed` | `attemptNumber, brandReview: BrandReview` | After `reviewStep` returns | (no status change; populates `attempts[n].brandReview`) |
| `PresentationApproved` | `attemptNumber, approvedSlideSet: SlideSet` | `BrandReview.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `PresentationRejectedFinal` | `bestAttemptNumber, bestSlideSet: SlideSet, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `BrandEvalRecorded` | `attemptNumber, verdict, score, guardedSlideNumbers, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `brandEvalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `presentationId, topic, targetAudience, slideCount, wordsPerSlide, requestedBy, submittedAt` |

## View row

`PresentationRow` is structurally identical to `Presentation` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `PresentationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllPresentations` returning the full list. Callers filter by `status` client-side.
