# Architecture — on-brand-genmedia

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`BrandEndpoint` is the entry point. It writes a `BriefSubmitted` event to `CampaignQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `GenerationWorkflow` per submission. The workflow alternates two agents — `BrandAgent` generates an asset, `ReviewerAgent` scores it — with a deterministic guardrail check between them. Each step emits an event on `AssetEntity`. `AssetsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `CampaignSimulator` drips a sample campaign brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `ReviewEvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first generated asset is sent back for revision and the second asset is approved. The brand agent → guardrail → reviewer alternation is explicit. Each attempt's events are written to `AssetEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The asset moves between two transient states (`GENERATING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `GENERATING` represents the guardrail-blocked path: an asset containing prohibited content sends the workflow back to `generateStep` with structured feedback. The `REVIEWING → GENERATING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every review on the entity.

## Entity model

`AssetEntity` is the system's source of truth; every transition writes one of seven event types. `CampaignQueue` is the audit log of submissions. `AssetsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `reviewStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `BrandEndpoint.submit` deduplicates on `(product, channel, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(assetId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Asset.attempts` with its own generated asset, guardrail verdict, and (once produced) review. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the reviewer.
