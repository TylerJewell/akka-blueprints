# Architecture — reflection-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ReflectionEndpoint` is the entry point. It writes a `TaskSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReflectionWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a response, `ReflectorAgent` scores it — feeding each critique back into the next revision. Each step emits an event on `TaskEntity`. `TasksView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `TaskSimulator` drips a sample task every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any reflected iteration that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first response is sent back for revision and the second response is accepted. The Generator → Reflector alternation is explicit. Each iteration's events are written to `TaskEntity` before the next step begins, so the UI's per-iteration timeline reconstructs the loop exactly.

## State machine

The task moves between two transient states (`GENERATING`, `REFLECTING`) and two terminal states (`ACCEPTED`, `REJECTED_FINAL`). The `REFLECTING → GENERATING` transition is the `REVISE` path — taken when the reflector returns `REVISE` and the iteration count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every iteration and every critique on the entity.

## Entity model

`TaskEntity` is the system's source of truth; every transition writes one of six event types. `SubmissionQueue` is the audit log of submissions. `TasksView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `reflectStep`; both carry explicit `stepTimeout(Duration.ofSeconds(60))` overrides (Lesson 4).
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxIterations` (default 4).
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `ReflectionEndpoint.submit` deduplicates on `(text, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(taskId, iterationNumber)` so a tick that fires twice for the same iteration is a no-op.
- Per-iteration accounting: every iteration is appended to `Task.iterations` with its own response and (once produced) reflection. The `iterationNumber` is monotonic across the whole loop.
- No guardrail step: this general-domain variant does not include a deterministic content check between the generator and reflector. All quality judgment passes through `ReflectorAgent`.
