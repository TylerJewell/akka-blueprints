# Architecture — brand-presentation-builder

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`PresentationEndpoint` is the entry point. It writes a `BriefSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `PresentationWorkflow` per submission. The workflow alternates two agents — `BuilderAgent` produces a slide set, `BrandReviewerAgent` scores it — with a deterministic per-slide word-count guardrail check between them. Each step emits an event on `PresentationEntity`. `PresentationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips a sample brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `BrandEvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first slide set is sent back for revision and the second is approved. The Builder → guardrail → Reviewer alternation is explicit. Each attempt's events are written to `PresentationEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The presentation moves between two transient states (`BUILDING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `BUILDING` represents the guardrail-blocked path: an over-limit slide set sends the workflow back to `buildStep` with structured feedback naming the offending slides. The `REVIEWING → BUILDING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every round of feedback on the entity.

## Entity model

`PresentationEntity` is the system's source of truth; every transition writes one of seven event types. `RequestQueue` is the audit log of submissions. `PresentationsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `buildStep` and `reviewStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `PresentationEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(presentationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Presentation.attempts` with its own slide set, guardrail verdict, and (once produced) brand review. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the reviewer.
