# Data model — seo-rubric-critic

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ArticleSubmission` | `title` | `String` | no | The article headline. |
| | `bodyText` | `String` | no | The full article body. |
| | `targetKeyword` | `String` | no | Primary SEO keyword. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ArticleDraft` | `title` | `String` | no | The (possibly revised) headline. |
| | `bodyText` | `String` | no | The full body text. |
| | `wordCount` | `int` | no | Word count of `bodyText`. |
| | `draftedAt` | `Instant` | no | When the agent returned the draft. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the draft. |
| | `reasonCode` | `String` | no | `OK`, `OVER_WORD_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `DimensionScore` | `dimension` | `String` | no | Name of the checklist dimension. |
| | `score` | `int` | no | 1–5 rubric score for this dimension. |
| | `note` | `String` | no | One-sentence explanation of the score. |
| `RubricFeedback` | `failingDimensions` | `List<DimensionScore>` | no | Dimensions that scored below 3; empty on PASS. |
| | `improvementSummary` | `String` | no | Two to four sentences on priority changes. |
| `RubricScore` | `passed` | `boolean` | no | Whether the article meets the acceptance threshold. |
| | `compositeScore` | `int` | no | Weighted composite 0–100. |
| | `dimensions` | `List<DimensionScore>` | no | Full per-dimension breakdown. |
| | `feedback` | `RubricFeedback` | no | Structured feedback for the optimizer. |
| | `scoredAt` | `Instant` | no | When the rubric agent returned. |
| `AuditRound` | `roundNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked revisions). |
| | `draft` | `ArticleDraft` | no | The draft scored or revised in this round. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `rubricScore` | `Optional<RubricScore>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Article` (entity state) | `articleId` | `String` | no | Unique id. |
| | `title` | `String` | no | Original headline. |
| | `targetKeyword` | `String` | no | Original target keyword. |
| | `wordCountCeiling` | `int` | no | Per-article word-count cap. |
| | `maxRounds` | `int` | no | Per-article retry ceiling (default 4). |
| | `acceptanceThreshold` | `int` | no | Minimum composite score to approve (default 75). |
| | `status` | `ArticleStatus` | no | See enum. |
| | `rounds` | `List<AuditRound>` | no | Bounded at `maxRounds`; starts empty. |
| | `approvedAtRound` | `Optional<Integer>` | yes | Populated on `ArticleApproved`. |
| | `approvedBodyText` | `Optional<String>` | yes | Populated on `ArticleApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `ArticleFailedFinal`. |
| | `createdAt` | `Instant` | no | When `ArticleCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the article reached a terminal state. |

## Enums

`ArticleStatus`: `SCORING`, `REVISING`, `APPROVED`, `FAILED_FINAL`.

`RubricVerdict`: `PASS`, `FAIL`.

## Events (`ArticleEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ArticleCreated` | `title, targetKeyword, wordCountCeiling, maxRounds, acceptanceThreshold, createdAt` | Workflow `startStep` | → `SCORING` |
| `RoundDraftRecorded` | `roundNumber, draft: ArticleDraft` | After `reviseStep` or initial draft submission | (no status change; appends to `rounds[]`) |
| `RoundGuardrailVerdictRecorded` | `roundNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → next scoring; `passed=false` → `REVISING` (re-revise) |
| `RoundScoreRecorded` | `roundNumber, rubricScore: RubricScore` | After `scoreStep` returns | `passed=true` → `APPROVED` path; `passed=false` → `REVISING` or halt path |
| `ArticleApproved` | `roundNumber, approvedBodyText, compositeScore` | `RubricScore.passed = true` | → `APPROVED`, `finishedAt = now` |
| `ArticleFailedFinal` | `bestRoundNumber, bestBodyText, bestCompositeScore, rejectionReason` | `rounds.size() == maxRounds` AND last score `passed = false`; OR `defaultStepRecovery` failover | → `FAILED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `roundNumber, verdict, compositeScore, wordCeilingExceeded, recordedAt` | `EvalSampler` per scored round; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `ArticleSubmitted` | `articleId, title, bodyText, targetKeyword, submittedBy, submittedAt` |

## View row

`ArticleRow` is structurally identical to `Article` — the `rounds` list is bounded at `maxRounds` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `ArticleEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllArticles` returning the full list. Callers filter by `status` client-side.
