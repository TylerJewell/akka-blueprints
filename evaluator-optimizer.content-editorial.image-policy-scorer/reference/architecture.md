# Architecture — image-policy-scorer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ImageEndpoint` is the entry point. It writes a `PromptSubmitted` event to `PromptQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ScoringWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a structured image description, `ScorerAgent` evaluates it against the policy rubric — with a deterministic safety gate check between them. Each step emits an event on `ImageEntity`. `ImagesView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `PromptSimulator` drips a sample prompt every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `PolicyEvalRecorded` event for any scored attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first description fails the policy score and the second description passes. The Generator → safety gate → Scorer alternation is explicit. Each attempt's events are written to `ImageEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The image moves between two transient states (`GENERATING`, `SCORING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `GENERATING` represents the gate-blocked path: a description that matches a prohibited signal sends the workflow back to `generateStep` with structured feedback. The `SCORING → GENERATING` transition is the `FAIL` path — taken when the scorer returns `FAIL` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every policy verdict on the entity.

## Entity model

`ImageEntity` is the system's source of truth; every transition writes one of seven event types. `PromptQueue` is the audit log of submissions. `ImagesView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `scoreStep`; the gate step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `ImageEndpoint.submit` deduplicates on `(promptText, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(imageId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Image.attempts` with its own description, gate verdict, and (once produced) policy verdict. The `attemptNumber` is monotonic across the whole loop, including gate-blocked attempts that never reached the scorer.
