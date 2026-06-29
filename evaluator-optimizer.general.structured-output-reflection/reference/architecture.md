# Architecture — structured-output-reflection

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`GenerationEndpoint` is the entry point. It writes a `RequestSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `SubmissionConsumer` subscribes to that queue and starts a `ReflectionWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a JSON document, `CriticAgent` scores it — with a deterministic schema-validation guardrail between them. Each step emits an event on `GenerationEntity`. `GenerationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips a sample generation request every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any validated attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first output is sent back for revision and the second output passes. The Generator → guardrail → Critic alternation is explicit. Each attempt's events are written to `GenerationEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The generation moves between two transient states (`GENERATING`, `VALIDATING`) and two terminal states (`PASSED`, `FAILED_FINAL`). The self-loop on `GENERATING` represents the guardrail-blocked path: a schema-invalid document sends the workflow back to `generateStep` with structured feedback. The `VALIDATING → GENERATING` transition is the `REVISE` path — taken when the critic returns `REVISE` and the attempt count is still below the ceiling. `FAILED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every validation report on the entity.

## Entity model

`GenerationEntity` is the system's source of truth; every transition writes one of seven event types. `SubmissionQueue` is the audit log of submissions. `GenerationsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `validateStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_FINAL`, not in a hung workflow.
- Idempotency: `GenerationEndpoint.submit` deduplicates on `(schemaName, prompt, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(generationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Generation.attempts` with its own output, guardrail verdict, and (once produced) validation report. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the critic.
