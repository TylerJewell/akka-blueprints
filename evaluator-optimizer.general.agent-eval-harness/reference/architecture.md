# Architecture — agent-eval-harness

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EvalEndpoint` is the primary entry point. It writes a `RunTriggered` event to `SuiteRegistry` (event-sourced for audit). A `Consumer` subscribes to that registry and starts an `EvalRunWorkflow` per triggered run. The workflow iterates over each test case: `EvaluatorAgent` invokes the target agent, `JudgeAgent` scores the response, and each result is written to `EvalRunEntity` before the next case begins. After all cases, a pure-function aggregate step computes accuracy and checks it against the configured threshold. `EvalRunsView` projects `EvalRunEntity` events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RunScheduler` triggers a periodic run every 5 minutes on the default suite so the App UI is not empty when first loaded; `AccuracySampler` ticks every 60 seconds and records an `AccuracySnapshot` event for any completed run that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style passing run with three test cases. The evaluate → judge alternation is explicit for each case. Each `CaseEvaluated` event is written to `EvalRunEntity` before the next case begins so the UI's per-case timeline reflects incremental progress in real time. The aggregate step runs after the last case, checks the accuracy gate, and emits `RunPassed`.

## State machine

The run moves through one setup state (`PENDING`), one active state (`RUNNING`), and two terminal states (`PASSED`, `FAILED`). The self-loop on `RUNNING` represents incremental case recording — each `CaseEvaluated` event does not change the run status; it appends to the `results` list. `PASSED` is reached only when `passRate >= passingThreshold`; `FAILED` is reached when the accuracy gate fires or when `defaultStepRecovery` triggers a failover from an unrecoverable agent error.

## Entity model

`EvalRunEntity` is the system's source of truth; every transition writes one of seven event types. `SuiteRegistry` is the audit log of suite registrations and run triggers. `EvalRunsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `invokeStep` and `judgeStep`; the aggregate step is in-process and effectively instant.
- Workflow-wide deadline: the workflow bounds itself with the suite's case count plus `stepTimeout` per case. For a 20-case suite, the maximum wall-clock time before `defaultStepRecovery` triggers is approximately `20 × (60 + 60) = 40 minutes`. A tighter wall-clock guard can be added by reducing `maxCasesPerRun`.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure in any case ends the run in `FAILED`, not in a hung workflow. The `failureReason` records which step triggered the failover.
- Idempotency: `AccuracySampler` deduplicates on `runId` so a tick that fires twice for the same completed run is a no-op on the entity side.
- Suite isolation: each run is keyed by `runId`; multiple concurrent runs on different suites do not share workflow or entity state. The `SuiteRegistry` entity is keyed by `suiteName`; concurrent suite registrations on different names do not conflict.
- Sequential case processing: the workflow does not fan out cases in parallel; it processes them one at a time. This avoids concurrent writes to the same `EvalRunEntity` and keeps the per-attempt timeline in `createdAt` order.
