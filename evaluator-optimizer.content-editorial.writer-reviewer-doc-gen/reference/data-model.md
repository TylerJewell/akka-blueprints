# Data model — writer-reviewer-doc-gen

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TopicRequest` | `topic` | `String` | no | The user-submitted topic. |
| | `wordCeiling` | `int` | no | Per-document hard cap on draft word count. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `DocumentDraft` | `text` | `String` | no | The document body itself. |
| | `wordCount` | `int` | no | Whitespace-delimited token count of `text`. |
| | `draftedAt` | `Instant` | no | When the Writer returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK`, `OVER_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `ReviewNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Review` | `verdict` | `ReviewVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `ReviewNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Reviewer returned. |
| `DraftAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `draft` | `DocumentDraft` | no | The Writer's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `review` | `Optional<Review>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Document` (entity state) | `documentId` | `String` | no | Unique id. |
| | `topic` | `String` | no | Original topic. |
| | `wordCeiling` | `int` | no | Per-document cap. |
| | `maxAttempts` | `int` | no | Per-document retry ceiling (default 4). |
| | `status` | `DocumentStatus` | no | See enum. |
| | `attempts` | `List<DraftAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `DocumentApproved`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `DocumentApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `DocumentRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `DocumentCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the document reached a terminal state. |

## Enums

`DocumentStatus`: `DRAFTING`, `REVIEWING`, `APPROVED`, `REJECTED_FINAL`.

`ReviewVerdict`: `APPROVE`, `REVISE`.

## Events (`DocumentEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `DocumentCreated` | `topic, wordCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `DRAFTING` |
| `DraftProduced` | `attemptNumber, draft: DocumentDraft` | After `writeStep` returns | (no status change; appends to `attempts[]`) |
| `DraftGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `REVIEWING`; `passed=false` → `DRAFTING` (re-draft) |
| `DraftReviewed` | `attemptNumber, review: Review` | After `reviewStep` returns | (no status change; populates `attempts[n].review`) |
| `DocumentApproved` | `attemptNumber, approvedText` | `Review.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `DocumentRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last review is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, ceilingExceeded, recordedAt` | `EvalSampler` per reviewed attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `TopicSubmitted` | `documentId, topic, wordCeiling, requestedBy, submittedAt` |

## View row

`DocumentRow` is structurally identical to `Document` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `DocumentEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllDocuments` returning the full list. Callers filter by `status` client-side.
