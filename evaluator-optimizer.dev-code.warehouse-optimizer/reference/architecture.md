# Architecture — warehouse-optimizer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`OptimizerEndpoint` is the entry point. It writes a `RequestSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `OptimizationWorkflow` per submission. The workflow alternates two agents — `OptimizerAgent` produces a proposal, `EvaluatorAgent` scores it — with a deterministic guardrail check and a conditional DBA approval pause between them. Each step emits an event on `OptimizationRequestEntity`. `RequestsView` projects those events into the read model the UI streams via SSE.

Three TimedActions sit alongside: `RequestSimulator` drips a sample optimization request every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any evaluated attempt that has not yet been sampled; `DBAApprovalGate` ticks every 15 seconds and logs a reminder for any request that has been waiting for DBA approval for more than 5 minutes.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first proposal is sent back for revision and the second proposal is approved. The Optimizer → guardrail → Evaluator alternation is explicit, and both proposals are non-DDL so the DBA gate is not activated. Each attempt's events are written to `OptimizationRequestEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The request moves through one transient holding state (`AWAITING_DBA`), two transient active states (`PROPOSING`, `EVALUATING`), and two terminal states (`APPROVED`, `REJECTED_FINAL`). The path through `AWAITING_DBA` is taken only when the DDL guardrail fires; a DBA rejection from that state transitions directly to `REJECTED_FINAL` without calling the evaluator. The `EVALUATING → PROPOSING` transition is the `REVISE` path — taken when the evaluator returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is also reached when the ceiling is hit or when an irrecoverable agent failure triggers the workflow's default step recovery.

## Entity model

`OptimizationRequestEntity` is the system's source of truth; every transition writes one of eight event types. `RequestQueue` is the audit log of submissions. `RequestsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeout: 60 s for `proposeStep` and `evaluateStep`; the guardrail step is in-process and effectively instant; `dbaGateStep` has no step timeout — it waits for an external signal.
- Workflow-wide deadline: the workflow bounds itself with `maxAttempts` (default 4). A DBA gate that never receives a decision would hold the workflow in `AWAITING_DBA` indefinitely; the `DBAApprovalGate` TimedAction provides a nudge mechanism but does not auto-resolve.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `OptimizerEndpoint.submit` deduplicates on `(originalSql, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(requestId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `OptimizationRequest.attempts` with its own proposal, guardrail verdict, optional DBA decision, and (once produced) evaluation. The `attemptNumber` is monotonic across the whole loop.
