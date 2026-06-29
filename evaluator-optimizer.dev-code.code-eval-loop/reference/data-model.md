# Data model — code-eval-loop

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Problem` | `statement` | `String` | no | The user-submitted problem description. |
| | `language` | `String` | no | Target programming language (e.g., "java"). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `CodeAttempt` | `code` | `String` | no | The code solution text. |
| | `language` | `String` | no | Target language, echoed from input. |
| | `lineCount` | `int` | no | Count of non-empty lines in `code`. |
| | `generatedAt` | `Instant` | no | When the Generator returned the attempt. |
| `SandboxVerdict` | `passed` | `boolean` | no | Whether the sandbox check accepted the code. |
| | `reasonCode` | `String` | no | `OK`, `FORBIDDEN_IMPORT`, `UNSAFE_SYSCALL`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `VerificationNotes` | `diagnostics` | `List<String>` | no | 0–5 diagnostic lines (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Verification` | `verdict` | `VerifierVerdict` | no | `PASS` or `FAIL`. |
| | `notes` | `VerificationNotes` | no | See above. |
| | `score` | `int` | no | 0–100 test-pass percentage; 100 required for PASS. |
| | `verifiedAt` | `Instant` | no | When the Verifier returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes sandbox-blocked attempts). |
| | `code` | `CodeAttempt` | no | The Generator's output for this attempt. |
| | `sandbox` | `SandboxVerdict` | no | The deterministic check's verdict. |
| | `verification` | `Optional<Verification>` | yes | Empty when the sandbox blocked; populated otherwise. |
| `Solution` (entity state) | `solutionId` | `String` | no | Unique id. |
| | `statement` | `String` | no | Original problem description. |
| | `language` | `String` | no | Target language. |
| | `maxAttempts` | `int` | no | Per-solution retry budget (default 5). |
| | `status` | `SolutionStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `passedAttemptNumber` | `Optional<Integer>` | yes | Populated on `SolutionPassed`. |
| | `passedCode` | `Optional<String>` | yes | Populated on `SolutionPassed`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `SolutionExhausted`. |
| | `createdAt` | `Instant` | no | When `SolutionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the solution reached a terminal state. |

## Enums

`SolutionStatus`: `GENERATING`, `VERIFYING`, `PASSED`, `EXHAUSTED`.

`VerifierVerdict`: `PASS`, `FAIL`.

## Events (`SolutionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SolutionCreated` | `statement, language, maxAttempts, createdAt` | Workflow `startStep` | → `GENERATING` |
| `AttemptGenerated` | `attemptNumber, code: CodeAttempt` | After `generateStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptSandboxVerdictRecorded` | `attemptNumber, verdict: SandboxVerdict` | After `sandboxStep` | `passed=true` → `VERIFYING`; `passed=false` → `GENERATING` (re-generate) |
| `AttemptVerified` | `attemptNumber, verification: Verification` | After `verifyStep` returns, or ci-gate synthetic rejection | (no status change; populates `attempts[n].verification`) |
| `SolutionPassed` | `attemptNumber, passedCode` | `Verification.verdict = PASS` AND `score = 100` | → `PASSED`, `finishedAt = now` |
| `SolutionExhausted` | `bestAttemptNumber, bestCode, exhaustionReason` | `attempts.size() == maxAttempts` AND last verification is `FAIL`; OR `defaultStepRecovery` failover | → `EXHAUSTED`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, sandboxBlocked, recordedAt` | `EvalSampler` per verified attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`ProblemQueue`)

| Event | Payload |
|---|---|
| `ProblemSubmitted` | `solutionId, statement, language, submittedBy, submittedAt` |

## View row

`SolutionRow` is structurally identical to `Solution` — the `attempts` list is bounded at `maxAttempts` (default 5) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SolutionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSolutions` returning the full list. Callers filter by `status` client-side.
