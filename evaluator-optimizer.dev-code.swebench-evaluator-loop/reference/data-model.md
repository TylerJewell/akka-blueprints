# Data model — swebench-evaluator-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IssueContext` | `repoName` | `String` | no | Repository name (e.g., `example-org/myrepo`). |
| | `issueNumber` | `int` | no | GitHub issue number. |
| | `description` | `String` | no | Issue text describing the bug or missing behaviour. |
| | `fileContext` | `String` | no | Source file(s) relevant to the fix; empty string if not provided. |
| `PatchAttempt` | `unifiedDiff` | `String` | no | The unified diff produced by `PatchAgent`. |
| | `lineCount` | `int` | no | Total line count of the diff. |
| | `patchedAt` | `Instant` | no | When `PatchAgent` returned the diff. |
| `CiGateVerdict` | `passed` | `boolean` | no | Whether the gate accepted the diff. |
| | `reasonCode` | `String` | no | `OK`, `MALFORMED_DIFF`, `EXCEEDS_LINE_BUDGET`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `TestFailureSummary` | `failingTests` | `List<String>` | no | Test function names that failed; empty on `PASS`. |
| | `errorSnippets` | `List<String>` | no | Short error excerpts, one per failing test; empty on `PASS`. |
| | `overallSummary` | `String` | no | One-sentence outcome summary; required either way. |
| `TestResult` | `verdict` | `EvalVerdict` | no | `PASS` or `FAIL`. |
| | `failures` | `TestFailureSummary` | no | Structured failure details (empty lists on `PASS`). |
| | `passCount` | `int` | no | Number of tests that passed. |
| | `failCount` | `int` | no | Number of tests that failed; 0 on `PASS`. |
| | `evaluatedAt` | `Instant` | no | When `TestEvaluatorAgent` returned. |
| `Iteration` | `iterationNumber` | `int` | no | 1-indexed; monotonic across the loop (includes gate-blocked iterations). |
| | `patch` | `PatchAttempt` | no | The `PatchAgent`'s output for this iteration. |
| | `gate` | `CiGateVerdict` | no | The deterministic structural check's verdict. |
| | `testResult` | `Optional<TestResult>` | yes | Empty when the gate blocked; populated otherwise. |
| `Issue` (entity state) | `issueId` | `String` | no | Unique id. |
| | `repoName` | `String` | no | Repository name. |
| | `issueNumber` | `int` | no | Issue number. |
| | `description` | `String` | no | Original issue description. |
| | `maxIterations` | `int` | no | Per-issue iteration ceiling (default 5). |
| | `tokenBudget` | `int` | no | Per-issue cumulative token ceiling (default 50000). |
| | `status` | `IssueStatus` | no | See enum. |
| | `iterations` | `List<Iteration>` | no | Bounded at `maxIterations`; starts empty. |
| | `solvedAtIteration` | `Optional<Integer>` | yes | Populated on `IssueSolved`. |
| | `acceptedPatch` | `Optional<String>` | yes | The accepted diff text; populated on `IssueSolved`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `IssueExhausted`. |
| | `createdAt` | `Instant` | no | When `IssueCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the issue reached a terminal state. |

## Enums

`IssueStatus`: `PATCHING`, `EVALUATING`, `SOLVED`, `EXHAUSTED`.

`EvalVerdict`: `PASS`, `FAIL`.

## Events (`IssueEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `IssueCreated` | `repoName, issueNumber, description, maxIterations, tokenBudget, createdAt` | Workflow `startStep` | → `PATCHING` |
| `IterationPatched` | `iterationNumber, patch: PatchAttempt` | After `patchStep` returns | (no status change; appends to `iterations[]`) |
| `IterationGateVerdictRecorded` | `iterationNumber, verdict: CiGateVerdict` | After `gateStep` | `passed=true` → `EVALUATING`; `passed=false` → `PATCHING` (re-patch) |
| `IterationEvaluated` | `iterationNumber, testResult: TestResult` | After `evaluateStep` returns | (no status change; populates `iterations[n].testResult`) |
| `IssueSolved` | `iterationNumber, acceptedPatch` | `TestResult.verdict = PASS` | → `SOLVED`, `finishedAt = now` |
| `IssueExhausted` | `bestIterationNumber, bestPatch, exhaustionReason` | `iterationCount == maxIterations` AND last result is `FAIL`; OR `tokenUsage >= tokenBudget`; OR `defaultStepRecovery` failover | → `EXHAUSTED`, `finishedAt = now` |
| `EvalRecorded` | `iterationNumber, verdict, passCount, failCount, gateBlocked, recordedAt` | `EvalSampler` per evaluated iteration; workflow on terminal transition | (no status change; appended to an internal `evalEvents[]` view-side projection) |

## Events (`IssueQueue`)

| Event | Payload |
|---|---|
| `IssueSubmitted` | `issueId, repoName, issueNumber, description, fileContext, submittedAt` |

## View row

`IssueRow` is structurally identical to `Issue` — the `iterations` list is bounded at `maxIterations` (default 5) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `IssueEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllIssues` returning the full list. Callers filter by `status` client-side.
