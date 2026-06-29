# Architecture — brand-aligner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`BrandEndpoint` is the entry point. It writes a `BriefSubmitted` event to `BriefQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `AlignmentWorkflow` per submission. The workflow alternates two agents — `CopywriterAgent` produces a variant, `BrandReviewerAgent` scores it — with a deterministic compliance check between them. Each step emits an event on `MaterialEntity`. `MaterialsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `CampaignSimulator` drips a sample brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `BrandEvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first variant is sent back for revision and the second variant is approved. The Copywriter → compliance check → Reviewer alternation is explicit. Each attempt's events are written to `MaterialEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The material moves between two transient states (`DRAFTING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `DRAFTING` represents the compliance-blocked path: an over-word-ceiling variant sends the workflow back to `generateStep` with structured feedback. The `REVIEWING → DRAFTING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every variant and every review on the entity.

## Entity model

`MaterialEntity` is the system's source of truth; every transition writes one of seven event types. `BriefQueue` is the audit log of submissions. `MaterialsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `reviewStep`; the compliance step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `BrandEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(materialId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Material.attempts` with its own variant, compliance verdict, and (once produced) review. The `attemptNumber` is monotonic across the whole loop, including compliance-blocked attempts that never reached the reviewer.
