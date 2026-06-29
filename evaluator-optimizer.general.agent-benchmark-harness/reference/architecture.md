# Architecture — agent-benchmark-harness

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`BenchmarkEndpoint` is the primary entry point. A POST request (or a `RunScheduler` tick) starts a `BenchmarkRunWorkflow`. The workflow first loads the registered task set from `TaskRegistry` (bootstrapped at startup from the bundled `benchmark-tasks.jsonl`). It then fans out across the task set, calling `RunnerAgent` per task to elicit a response and `ScorerAgent` per response to grade it. Each step emits events on `RunEntity`. `RunsView` projects those events into the read model the UI streams via SSE.

`TaskResultConsumer` subscribes to `RunEntity` events and maintains running pass/fail counters, making partial progress visible through the SSE stream before the run completes. `AccuracySampler` ticks every 15 minutes and records a durable accuracy snapshot for completed runs. `RunScheduler` fires every 24 hours to kick off an unattended benchmark cycle.

## Interaction sequence

The sequence diagram traces a J1-style passing run with two tasks. `RunnerAgent` and `ScorerAgent` alternate per task; the workflow waits for each scoring result before proceeding to the next task. This sequential ordering keeps the workflow within a single execution thread while still producing a full per-task audit trail. The aggregate step is a pure function; it emits the terminal `RunSummaryEvent` once all tasks are scored.

## State machine

The run moves through one setup state (`PENDING`), one active state (`RUNNING`), and two terminal states (`PASSED`, `FAILED`). The self-loop on `RUNNING` represents the per-task events (`TaskResponseRecorded`, `TaskScoredEvent`) that accumulate as the workflow processes each task. `FAILED` is reached either when `aggregateStep` computes a pass rate below the threshold or when `defaultStepRecovery` triggers the failover path after exhausting retries on an agent step.

## Entity model

`RunEntity` is the system's source of truth; every transition writes one of six event types. `TaskRegistry` is an independent entity that stores the task set; it is read by the workflow but never written after the bootstrap phase. `RunsView` is the only read-side projection — the UI never queries the entity directly. `TaskResultConsumer` subscribes to `RunEntity` events and drives counter updates back into the same entity, forming a controlled intra-system feedback loop.

## Concurrency & timeouts

- Per-step timeout: 90 s for `runStep` and `scoreStep`; `aggregateStep` is in-process and effectively instant.
- Fan-out ordering: tasks are processed sequentially within a single workflow thread. Adding parallelism requires splitting the fan-out into child workflows, which is a named extension point in the spec but out of scope for this blueprint.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any agent failure that exhausts retries transitions the run to `FAILED` with a structured `failureReason`, not to a hung or stuck state.
- AccuracySampler idempotency: snapshot recording is keyed on `runId`; a double-fire of the sampler produces one event, not two.
- CI gate conservatism: `GET /api/ci-gate` returns `gated=true` when no completed run exists in `RunsView`. A pipeline that has never been benchmarked cannot silently pass through the gate.
- Threshold immutability per run: `benchmark.passing-threshold` is captured at workflow start and stored on `RunEntity` state, so a configuration change during a long-running benchmark does not retroactively change the pass/fail outcome.
