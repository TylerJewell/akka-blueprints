# Architecture — multi-agent-eval-harness

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EvalEndpoint` is the entry point. It writes a `SuiteRegistered` event to `ScenarioRegistry` (event-sourced so scenario definitions are durable and replayable). A `Consumer` subscribes to that registry and starts an `EvalRunWorkflow` per submitted run. The workflow fans the scenario suite to `OrchestratorAgent`, collects the aggregate result, applies a deterministic threshold check, and then hands the result to `JudgeAgent` for a rubric-based verdict. Each step emits events on `EvalRunEntity`. `EvalRunsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ScenarioLoader` submits a canned eval run every 120 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 60 seconds and records a `ScenarioEvalRecorded` event for any judged scenario that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style passing run where the first orchestration attempt produces an aggregate score above the pass threshold and the judge returns `PASS`. The Orchestrator → threshold check → Judge sequence is explicit. Each step's events are written to `EvalRunEntity` before the next step begins, so the UI's per-scenario timeline reconstructs the run exactly without a separate fetch.

## State machine

The eval run moves between two transient states (`RUNNING`, `JUDGING`) and two terminal states (`PASSED`, `FAILED`). The self-loop on `RUNNING` represents the rerun path: a below-threshold orchestration that still has retries remaining returns the workflow to `dispatchStep` with only the failing scenario IDs. The `JUDGING → RUNNING` transition is the `FAIL + retries remaining` path. `FAILED` is reached only when retries are exhausted or an unrecoverable agent error triggers `defaultStepRecovery`. Both terminal states preserve all scenario results and all judgments on the entity.

## Entity model

`EvalRunEntity` is the system's source of truth; every workflow transition writes one of eight event types. `ScenarioRegistry` is the durable store of scenario definitions, not a transient cache. `EvalRunsView` is the only read-side projection — the UI never queries the entity directly, keeping the read path fast and the event log uncontested.

## Concurrency & timeouts

- Per-step timeout: 90 s for `dispatchStep` and `judgeStep`. The orchestrator fans out across all scenarios in a suite; 90 s accommodates the aggregate latency on a suite of up to 20 scenarios. The `ciGateStep` and `thresholdStep` are in-process and effectively instant.
- Workflow-wide deadline: the workflow bounds itself with `maxRetries` (default 2) rather than a wall-clock deadline. Adding a wall-clock guard is straightforward by lifting `maxRetries` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED`, not a hung workflow.
- Idempotency: `EvalEndpoint.submit` deduplicates on `(suiteName, requestedBy)` over a 15 s window. `EvalSampler` deduplicates on `(runId, scenarioId)` so a tick that fires twice for the same scenario is a no-op.
- CI gate: `ciGateStep` runs after every terminal transition and is idempotent on `(runId)`. External CI pipelines consume the SSE stream at `GET /api/runs/sse` or poll `GET /api/runs/{id}` for the `ciGateBlocked` field.
- Rerun scope: on a `FAIL` verdict, the workflow extracts failing `scenarioId` values from `JudgmentNotes.bullets` and passes only those to `RERUN_FAILING`, reducing orchestrator latency on subsequent attempts rather than re-running the entire suite.
