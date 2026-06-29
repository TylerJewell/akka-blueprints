# Data model — evaluator-optimizer-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ProblemStatement` | `description` | `String` | no | The user-submitted problem. |
| | `tokenCeiling` | `int` | no | Per-job hard cap on candidate token count. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `Candidate` | `text` | `String` | no | The solution text itself. |
| | `tokenCount` | `int` | no | Token count of `text`. |
| | `generatedAt` | `Instant` | no | When the Generator returned the candidate. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the candidate. |
| | `reasonCode` | `String` | no | `OK`, `OVER_CEILING`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `EvaluationNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Evaluation` | `verdict` | `EvaluatorVerdict` | no | `ACCEPT` or `REVISE`. |
| | `notes` | `EvaluationNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Evaluator returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `candidate` | `Candidate` | no | The Generator's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `evaluation` | `Optional<Evaluation>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `Job` (entity state) | `jobId` | `String` | no | Unique id. |
| | `description` | `String` | no | Original problem statement. |
| | `tokenCeiling` | `int` | no | Per-job cap. |
| | `maxAttempts` | `int` | no | Per-job attempt ceiling (default 4). |
| | `status` | `JobStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `JobAccepted`. |
| | `acceptedText` | `Optional<String>` | yes | Populated on `JobAccepted`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `JobRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `JobCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |

## Enums

`JobStatus`: `GENERATING`, `EVALUATING`, `ACCEPTED`, `REJECTED_FINAL`.

`EvaluatorVerdict`: `ACCEPT`, `REVISE`.

## Events (`JobEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `JobCreated` | `description, tokenCeiling, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, candidate: Candidate` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `EVALUATING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptEvaluated` | `attemptNumber, evaluation: Evaluation` | After `evaluateStep` returns | (no status change; populates `attempts[n].evaluation`) |
| `JobAccepted` | `attemptNumber, acceptedText` | `Evaluation.verdict = ACCEPT` | → `ACCEPTED`, `finishedAt = now` |
| `JobRejectedFinal` | `bestAttemptNumber, bestText, rejectionReason` | `attempts.size() == maxAttempts` AND last evaluation is `REVISE`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, ceilingExceeded, recordedAt` | `EvalSampler` per evaluated attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`SubmissionQueue`)

| Event | Payload |
|---|---|
| `ProblemSubmitted` | `jobId, description, tokenCeiling, submittedBy, submittedAt` |

## View row

`JobRow` is structurally identical to `Job` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `JobEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllJobs` returning the full list. Callers filter by `status` client-side.
