# Architecture — competitive-coding-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SolverEndpoint` is the entry point. It writes a `ProblemSubmitted` event to `ProblemQueue` (event-sourced for audit and replay). A `Consumer` subscribes to that queue and starts a `SolvingWorkflow` per submission. The workflow alternates two agents — `SolverAgent` generates a Java solution, `JudgeAgent` scores the sandbox execution report — with a deterministic resource check between them. Each step emits an event on `SubmissionEntity`. `SubmissionsView` projects those events into the read model the UI streams via SSE.

The `E2B Sandbox` external service sits between the solver and the judge: the workflow's `sandboxStep` POSTs the generated source, compiles it, and runs it against all sample test cases inside a resource-bounded micro-VM. The sandbox result is a `SandboxReport`; only reports with `resourcesOk = true` proceed to the judge.

Two TimedActions sit alongside: `ProblemSimulator` drips a sample USACO problem every 90 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 45 seconds and records an `EvalRecorded` event for any judged attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first solution fails two of three test cases and the second solution passes all three. The Solver → sandbox → Judge alternation is explicit. Each attempt's events are written to `SubmissionEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The submission moves between two transient states (`GENERATING`, `JUDGING`) and two terminal states (`ACCEPTED`, `REJECTED_FINAL`). The self-loop on `GENERATING` represents the sandbox-blocked path: a solution that exceeds the time or memory limit sends the workflow back to `generateStep` with structured feedback. The `JUDGING → GENERATING` transition is the `FAIL` path — taken when the judge returns `FAIL` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit without a `PASS`; both terminal states preserve every solution and every judge report on the entity.

## Entity model

`SubmissionEntity` is the system's source of truth; every transition writes one of seven event types. `ProblemQueue` is the audit log of problem submissions. `SubmissionsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeout: 120 s for `generateStep` and `sandboxStep`; 60 s for `judgeStep`; the sandbox compile-and-run is included in the 120 s sandboxStep budget.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 5). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent or sandbox failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `SolverEndpoint.submit` deduplicates on `(title, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(submissionId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Submission.attempts` with its own solution, sandbox report, and (once produced) judge verdict. The `attemptNumber` is monotonic across the whole loop, including sandbox-blocked attempts that never reached the judge.
- Best-of selection on rejection: `rejectStep` selects the attempt with the highest `passedCases / totalCases` ratio; ties broken by later `attemptNumber`. The source code of that attempt is stored as `acceptedSourceCode` so an operator can inspect and fix it manually.
