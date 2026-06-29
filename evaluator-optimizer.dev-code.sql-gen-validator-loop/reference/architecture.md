# Architecture — sql-gen-validator-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SqlEndpoint` is the entry point. It writes a `QuestionSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SqlWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a SQL query, `ValidatorAgent` checks it against the schema — with a deterministic mutation guardrail between them. Each step emits an event on `QueryRequestEntity`. `QueryRequestView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips a sample natural-language question every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `ValidationEvalRecorded` event for any validated attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first generated query fails validation and the second is accepted. The GeneratorAgent → guardrail → ValidatorAgent alternation is explicit. Each attempt's events are written to `QueryRequestEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The request moves between two transient states (`GENERATING`, `VALIDATING`) and two terminal states (`ACCEPTED`, `FAILED_FINAL`). The self-loop on `GENERATING` represents the mutation-guardrail-blocked path: a query containing a forbidden keyword sends the workflow back to `generateStep` with structured feedback. The `VALIDATING → GENERATING` transition is the `INVALID` path — taken when the validator returns `INVALID` and the attempt count is still below the ceiling. `FAILED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every validation note on the entity.

## Entity model

`QueryRequestEntity` is the system's source of truth; every transition writes one of seven event types. `RequestQueue` is the audit log of submissions. `QueryRequestView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `validateStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard can be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_FINAL`, not in a hung workflow.
- Idempotency: `SqlEndpoint.submit` deduplicates on `(question, schemaName, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(requestId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `QueryRequest.attempts` with its own generated query, guardrail verdict, and (once produced) validation result. The `attemptNumber` is monotonic across the whole loop, including mutation-blocked attempts that never reached the validator.
