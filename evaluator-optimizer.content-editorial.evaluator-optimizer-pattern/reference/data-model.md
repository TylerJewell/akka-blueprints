# Data model — evaluator-optimizer-workflow

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ArticleBrief` | `summary` | `String` | no | The user-submitted article summary. |
| | `wordCeiling` | `int` | no | Per-headline hard cap on word count. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `HeadlineDraft` | `text` | `String` | no | The headline itself. |
| | `wordCount` | `int` | no | Number of words in `text`. |
| | `draftedAt` | `Instant` | no | When the Writer returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK`, `OVER_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Review` | `verdict` | `ReviewerVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `reviewedAt` | `Instant` | no | When the Reviewer returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `draft` | `HeadlineDraft` | no | The Writer's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `review` | `Optional<Review>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Headline` (entity state) | `headlineId` | `String` | no | Unique id. |
| | `summary` | `String` | no | Original article summary. |
| | `wordCeiling` | `int` | no | Per-headline cap. |
| | `maxAttempts` | `int` | no | Per-headline retry ceiling (default 4). |
| | `status` | `HeadlineStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `HeadlineApproved`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `HeadlineApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `HeadlineRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `HeadlineCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the headline reached a terminal state. |

## Enums

`HeadlineStatus`: `DRAFTING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewerVerdict`: `APPROVE`, `REVISE`.

## Events (`HeadlineEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `HeadlineCreated` | `summary, wordCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `AttemptDrafted` | `attemptNumber, draft: HeadlineDraft` | After `draftStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `REVIEWING`; `passed=false` → `DRAFTING` (re-draft) |
| `AttemptReviewed` | `attemptNumber, review: Review` | After `reviewStep` returns | (no status change; populates `attempts[n].review`) |
| `HeadlineApproved` | `attemptNumber, approvedText` | `Review.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `HeadlineRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, ceilingExceeded, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `ArticleSubmitted` | `headlineId, summary, wordCeiling, requestedBy, submittedAt` |

## View row

`HeadlineRow` is structurally identical to `Headline` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `HeadlineEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllHeadlines` returning the full list. Callers filter by `status` client-side.
