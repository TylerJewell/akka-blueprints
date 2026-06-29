# Architecture — trajectory-eval

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`TrajectoryEndpoint` is the entry point. It verifies that a reference path exists in `ReferenceStore` for the submitted scenario before creating an evaluation. A `Consumer` subscribes to evaluation-creation events and starts a `TrajectoryWorkflow` per submission. The workflow alternates two agents — `RunnerAgent` produces a trajectory, `EvaluatorAgent` compares it against the reference — with a deterministic step-count guardrail between them. Each step emits an event on `EvaluationEntity`. `EvaluationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ScenarioSimulator` drips a sample scenario every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any evaluated attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first trajectory deviates on two steps and the second trajectory matches the reference exactly. The Runner → guardrail → Evaluator alternation is explicit. Each attempt's events are written to `EvaluationEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The evaluation moves between two transient states (`RUNNING`, `EVALUATING`) and two terminal states (`PASSED`, `FAILED_FINAL`). The self-loop on `RUNNING` represents the guardrail-blocked path: an over-step-limit trajectory sends the workflow back to `executeStep` with structured feedback. The `EVALUATING → RUNNING` transition is the `FAIL` path — taken when the evaluator returns `FAIL` and the attempt count is still below the ceiling. `FAILED_FINAL` is reached only when the ceiling is exhausted; both terminal states preserve every trajectory and every deviation report on the entity.

## Entity model

`EvaluationEntity` is the system's source of truth; every transition writes one of seven event types. `ReferenceStore` is the registry of reference paths — it is an event-sourced entity keyed by scenario id, so reference updates are themselves auditable. `EvaluationsView` is the only read-side projection; the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `executeStep` and `evaluateStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A duration-based guard could be layered on top by the deployer; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_FINAL`, not in a hung workflow.
- Idempotency: `TrajectoryEndpoint.submit` deduplicates on `(scenarioId, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(evaluationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Evaluation.attempts` with its own trajectory, guardrail verdict, and (once produced) evaluator verdict. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the evaluator.
- Reference path snapshot: the workflow fetches `ReferencePath` from `ReferenceStore` at `startStep` and passes it into every subsequent `evaluateStep`. Mid-loop reference updates take effect on the next evaluation submission, not mid-run.
