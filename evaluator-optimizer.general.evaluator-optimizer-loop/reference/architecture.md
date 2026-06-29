# Architecture — evaluator-optimizer-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`OptimizationEndpoint` is the entry point. It writes a `ProblemSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `OptimizationWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a candidate, `EvaluatorAgent` scores it — with a deterministic guardrail check between them. Each step emits an event on `JobEntity`. `JobsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ProblemSimulator` drips a sample problem every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any evaluated attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first candidate is sent back for revision and the second is accepted. The Generator → guardrail → Evaluator alternation is explicit. Each attempt's events are written to `JobEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The job moves between two transient states (`GENERATING`, `EVALUATING`) and two terminal states (`ACCEPTED`, `REJECTED_FINAL`). The self-loop on `GENERATING` represents the guardrail-blocked path: an over-ceiling candidate sends the workflow back to `generateStep` with structured feedback. The `EVALUATING → GENERATING` transition is the `REVISE` path — taken when the evaluator returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every evaluation on the entity.

## Entity model

`JobEntity` is the system's source of truth; every transition writes one of seven event types. `SubmissionQueue` is the audit log of submissions. `JobsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `evaluateStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; this is out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `OptimizationEndpoint.submit` deduplicates on `(description, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(jobId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Job.attempts` with its own candidate, guardrail verdict, and (once produced) evaluation. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the evaluator.
