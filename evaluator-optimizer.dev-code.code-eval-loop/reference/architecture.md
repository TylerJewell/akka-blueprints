# Architecture — code-eval-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`CodeEndpoint` is the entry point. It writes a `ProblemSubmitted` event to `ProblemQueue` (event-sourced for audit). A Consumer subscribes to that queue and starts a `SolutionWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a code attempt, `VerifierAgent` evaluates it — with a deterministic sandbox check between them. Each step emits an event on `SolutionEntity`. `SolutionsView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `ProblemSimulator` drips a sample problem every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any verified attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first attempt fails verification and the second attempt passes all tests. The Generator → sandbox check → Verifier alternation is explicit. Each attempt's events are written to `SolutionEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The solution moves between two transient states (`GENERATING`, `VERIFYING`) and two terminal states (`PASSED`, `EXHAUSTED`). The self-loop on `GENERATING` represents the sandbox-blocked path: code containing a forbidden import or unsafe syscall sends the workflow back to `generateStep` with structured feedback. The `VERIFYING → GENERATING` transition is the `FAIL` path — taken when the verifier returns `FAIL` and the attempt count is still below the budget ceiling. `EXHAUSTED` is reached only when the budget is hit; both terminal states preserve every attempt and every diagnostic on the entity.

## Entity model

`SolutionEntity` is the system's source of truth; every transition writes one of seven event types. `ProblemQueue` is the audit log of submissions. `SolutionsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for `generateStep` and `verifyStep`; the sandbox step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 5). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `EXHAUSTED`, not in a hung workflow.
- Idempotency: `CodeEndpoint.submit` deduplicates on `(statement, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(solutionId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Solution.attempts` with its own code, sandbox verdict, and (once produced) verification. The `attemptNumber` is monotonic across the whole loop, including sandbox-blocked attempts that never reached the verifier.
- CI gate: after the VerifierAgent returns `PASS`, `passStep` re-checks `verification.score() == 100` deterministically. Only a perfect score advances to `SolutionPassed`; any lower score re-enters the loop as `FAIL`. This guards against partial-pass hallucinations without running code outside the sandboxed verifier context.
