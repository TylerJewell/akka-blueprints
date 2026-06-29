# Data model — ab-model-eval

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskPrompt` | `text` | `String` | no | The task prompt submitted by the user. |
| | `preferredCandidate` | `String` | no | Label of the operator's currently preferred model (`"A"` or `"B"`). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `CandidateResponse` | `candidateId` | `String` | no | `"A"` or `"B"`. |
| | `text` | `String` | no | The candidate's answer to the task prompt. |
| | `tokensUsed` | `int` | no | Approximate token count for this response. |
| | `respondedAt` | `Instant` | no | When the agent returned the response. |
| `RubricScores` | `accuracy` | `int` | no | 1–5 score for factual accuracy. |
| | `relevance` | `int` | no | 1–5 score for on-topic relevance. |
| | `clarity` | `int` | no | 1–5 score for structural clarity. |
| | `conciseness` | `int` | no | 1–5 score for absence of padding. |
| `JudgementRecord` | `winner` | `String` | no | `"A"`, `"B"`, or `"TIE"`. |
| | `scoresA` | `RubricScores` | no | Per-dimension scores for candidate A. |
| | `scoresB` | `RubricScores` | no | Per-dimension scores for candidate B. |
| | `totalA` | `int` | no | Sum of `scoresA` (range 4–20). |
| | `totalB` | `int` | no | Sum of `scoresB` (range 4–20). |
| | `rationale` | `String` | no | One-sentence explanation of the winner decision. |
| | `judgedAt` | `Instant` | no | When the judge agent returned. |
| `Trial` (entity state) | `trialId` | `String` | no | Unique id. |
| | `taskText` | `String` | no | Original task prompt. |
| | `preferredCandidate` | `String` | no | Operator's preferred candidate label at submission time. |
| | `timeoutSeconds` | `int` | no | Per-step timeout (default 60). |
| | `status` | `TrialStatus` | no | See enum. |
| | `responseA` | `Optional<CandidateResponse>` | yes | Populated after `CandidateAResponded`. |
| | `responseB` | `Optional<CandidateResponse>` | yes | Populated after `CandidateBResponded`. |
| | `judgement` | `Optional<JudgementRecord>` | yes | Populated after `TrialJudged`. |
| | `recertificationRequired` | `boolean` | no | Set by `RecertificationFlagged`; starts `false`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `TrialTimedOut`. |
| | `createdAt` | `Instant` | no | When `TrialCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the trial reached a terminal state. |

## Enums

`TrialStatus`: `RUNNING`, `JUDGED`, `TIMED_OUT`.

`CandidateWinner`: `A`, `B`, `TIE`. (Used in `EvalRecorded` and `JudgementRecord.winner` string field; the enum type is for internal dispatch.)

## Events (`TrialEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TrialCreated` | `trialId, taskText, preferredCandidate, timeoutSeconds, createdAt` | Workflow `startStep` | → `RUNNING` |
| `CandidateAResponded` | `trialId, response: CandidateResponse` | After `candidateAStep` returns | (no status change; populates `responseA`) |
| `CandidateBResponded` | `trialId, response: CandidateResponse` | After `candidateBStep` returns | (no status change; populates `responseB`) |
| `TrialJudged` | `trialId, judgement: JudgementRecord` | After `judgeStep` returns | → `JUDGED`, `finishedAt = now` |
| `TrialTimedOut` | `trialId, failureReason: String` | `defaultStepRecovery` failover fires | → `TIMED_OUT`, `finishedAt = now` |
| `RecertificationFlagged` | `trialId, required: boolean` | `recertStep` detects win-rate below threshold; or `EvalEndpoint.recertify` clears | (no status change; sets `recertificationRequired`) |
| `EvalRecorded` | `trialId, winner, totalA, totalB, preferredCandidateWon, recertificationRequired, recordedAt` | `AccuracySampler` per judged trial; workflow on `TrialJudged` terminal transition | (no status change; appends to `evalEvents[]` view-side projection) |

## Events (`TaskQueueEntity`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `trialId, text, preferredCandidate, submittedBy, submittedAt` |

## View row

`TrialRow` is structurally identical to `Trial` plus two computed fields: `winRateA` (double) and `winRateB` (double), computed over all `JUDGED` trials in the view. The `TableUpdater` consumes every `TrialEntity` event and rewrites the row atomically, recomputing the win-rate aggregates after each `TrialJudged` event.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllTrials` returning the full list. Callers filter by `status` client-side.
