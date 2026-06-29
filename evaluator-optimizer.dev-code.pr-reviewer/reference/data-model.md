# Data model — pr-reviewer

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PrSubmission` | `diffText` | `String` | no | The unified diff of the pull request. |
| | `description` | `String` | no | PR author's description; empty string when not provided. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ReviewComment` | `filePath` | `String` | no | Relative path from the diff header. |
| | `lineNumber` | `int` | no | Line number from the diff; 0 for PR-level comments. |
| | `severity` | `CommentSeverity` | no | `BLOCKER`, `SUGGESTION`, or `NITPICK`. |
| | `body` | `String` | no | Impersonal comment text; 1–3 sentences. |
| `FeedbackDraft` | `comments` | `List<ReviewComment>` | no | The structured feedback comments. |
| | `commentCount` | `int` | no | Count of items in `comments`. |
| | `draftedAt` | `Instant` | no | When the Reviewer returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK` or `PERSONAL_CRITIQUE`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `AlignmentNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `AlignmentCheck` | `verdict` | `AlignmentVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `AlignmentNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `checkedAt` | `Instant` | no | When the Alignment agent returned. |
| `ReviewAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `draft` | `FeedbackDraft` | no | The Reviewer's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `alignmentCheck` | `Optional<AlignmentCheck>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Review` (entity state) | `reviewId` | `String` | no | Unique id. |
| | `diffText` | `String` | no | Original PR diff. |
| | `description` | `String` | no | PR description. |
| | `maxAttempts` | `int` | no | Per-review retry ceiling (default 4). |
| | `status` | `ReviewStatus` | no | See enum. |
| | `attempts` | `List<ReviewAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `ReviewApproved`. |
| | `approvedComments` | `Optional<List<ReviewComment>>` | yes | Populated on `ReviewApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `ReviewRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `ReviewCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the review reached a terminal state. |

## Enums

`ReviewStatus`: `REVIEWING`, `CHECKING_ALIGNMENT`, `APPROVED`, `REJECTED_FINAL`.

`AlignmentVerdict`: `APPROVE`, `REVISE`.

`CommentSeverity`: `BLOCKER`, `SUGGESTION`, `NITPICK`.

## Events (`ReviewEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ReviewCreated` | `reviewId, diffText, description, maxAttempts, createdAt` | Workflow `startStep` | → `REVIEWING` |
| `FeedbackDrafted` | `attemptNumber, draft: FeedbackDraft` | After `reviewStep` returns | (no status change; appends to `attempts[]`) |
| `FeedbackGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `CHECKING_ALIGNMENT`; `passed=false` → `REVIEWING` (re-review) |
| `FeedbackAlignmentChecked` | `attemptNumber, alignmentCheck: AlignmentCheck` | After `alignmentStep` returns | (no status change; populates `attempts[n].alignmentCheck`) |
| `ReviewApproved` | `attemptNumber, approvedComments` | `AlignmentCheck.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `ReviewRejectedFinal` | `bestAttemptNumber, bestComments, rejectionReason` | `attempts.size() == maxAttempts` AND last alignment check is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, personalCritiqueBlocked, recordedAt` | `EvalSampler` per alignment-checked attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`PrQueue`)

| Event | Payload |
|---|---|
| `PrSubmitted` | `reviewId, diffText, description, submittedBy, submittedAt` |

## View row

`ReviewRow` is structurally identical to `Review` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `ReviewEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllReviews` returning the full list. Callers filter by `status` client-side.
