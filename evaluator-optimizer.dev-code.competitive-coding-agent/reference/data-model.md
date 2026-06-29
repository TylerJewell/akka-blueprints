# Data model — competitive-coding-agent

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Problem` | `title` | `String` | no | Short identifier for the problem. |
| | `statementText` | `String` | no | Full problem statement as submitted. |
| | `sampleCases` | `List<TestCase>` | no | Input/output pairs used for sandbox execution. |
| | `timeLimitMs` | `int` | no | Per-attempt wall time ceiling in milliseconds. |
| | `memoryLimitMb` | `int` | no | Per-attempt memory ceiling in megabytes. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `TestCase` | `inputData` | `String` | no | Raw stdin content for this case. |
| | `expectedOutput` | `String` | no | Expected stdout, trimmed for comparison. |
| `GeneratedSolution` | `sourceCode` | `String` | no | Complete, compilable Java program. |
| | `language` | `String` | no | Always `"java"` for this blueprint. |
| | `lineCount` | `int` | no | Line count of `sourceCode`. |
| | `generatedAt` | `Instant` | no | When the SolverAgent returned the solution. |
| `SandboxReport` | `resourcesOk` | `boolean` | no | Whether the run stayed within time and memory limits. |
| | `reasonCode` | `String` | no | `OK`, `TLE`, `MLE`, `CE`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `resourcesOk=true`. |
| | `caseResults` | `List<CaseResult>` | no | Per-case outcomes. |
| | `peakMemoryMb` | `int` | no | Highest memory observed across all cases. |
| | `wallTimeMs` | `long` | no | Total wall time across all cases. |
| `CaseResult` | `caseIndex` | `int` | no | 0-indexed position in `sampleCases`. |
| | `actualOutput` | `String` | no | Actual stdout from the execution (trimmed). |
| | `passed` | `boolean` | no | Whether `actualOutput` matches `expectedOutput`. |
| | `verdict` | `String` | no | `AC`, `WA`, `TLE`, `MLE`, `RE` per case. |
| `FailureNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `JudgeVerdict` | `outcome` | `JudgeOutcome` | no | `PASS` or `FAIL`. |
| | `notes` | `FailureNotes` | no | See above. |
| | `passedCases` | `int` | no | Count of cases that produced correct output. |
| | `totalCases` | `int` | no | Total cases in the sandbox report. |
| | `evaluatedAt` | `Instant` | no | When the JudgeAgent returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes sandbox-blocked attempts). |
| | `solution` | `GeneratedSolution` | no | The SolverAgent's output for this attempt. |
| | `sandbox` | `SandboxReport` | no | The sandbox execution result. |
| | `verdict` | `Optional<JudgeVerdict>` | yes | Empty when the sandbox blocked; populated otherwise. |
| `Submission` (entity state) | `submissionId` | `String` | no | Unique id. |
| | `problemTitle` | `String` | no | Original problem title. |
| | `statementText` | `String` | no | Original problem statement. |
| | `timeLimitMs` | `int` | no | Per-attempt time ceiling. |
| | `memoryLimitMb` | `int` | no | Per-attempt memory ceiling. |
| | `maxAttempts` | `int` | no | Retry ceiling (default 5). |
| | `status` | `SubmissionStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `acceptedAttemptNumber` | `Optional<Integer>` | yes | Populated on `SubmissionAccepted`. |
| | `acceptedSourceCode` | `Optional<String>` | yes | Populated on `SubmissionAccepted`; also set to best-of on `SubmissionRejectedFinal`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `SubmissionRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `SubmissionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the submission reached a terminal state. |

## Enums

`SubmissionStatus`: `GENERATING`, `JUDGING`, `ACCEPTED`, `REJECTED_FINAL`.

`JudgeOutcome`: `PASS`, `FAIL`.

## Events (`SubmissionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SubmissionCreated` | `submissionId, problemTitle, statementText, timeLimitMs, memoryLimitMb, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, solution: GeneratedSolution` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptSandboxReportRecorded` | `attemptNumber, sandbox: SandboxReport` | After `sandboxStep` completes | `resourcesOk=true` → `JUDGING`; `resourcesOk=false` → `GENERATING` (re-generate) |
| `AttemptJudged` | `attemptNumber, verdict: JudgeVerdict` | After `judgeStep` returns | (no status change; populates `attempts[n].verdict`) |
| `SubmissionAccepted` | `attemptNumber, acceptedSourceCode` | `JudgeVerdict.outcome = PASS` | → `ACCEPTED`, `finishedAt = now` |
| `SubmissionRejectedFinal` | `bestAttemptNumber, bestSourceCode, rejectionReason` | `attempts.size() == maxAttempts` AND last verdict is `FAIL`; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, outcome, passedCases, totalCases, sandboxOk, recordedAt` | `EvalSampler` per judged attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`ProblemQueue`)

| Event | Payload |
|---|---|
| `ProblemSubmitted` | `submissionId, title, statementText, sampleCases, timeLimitMs, memoryLimitMb, submittedBy, submittedAt` |

## View row

`SubmissionRow` is structurally identical to `Submission` — the `attempts` list is bounded at `maxAttempts` (default 5) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SubmissionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSubmissions` returning the full list. Callers filter by `status` client-side.
