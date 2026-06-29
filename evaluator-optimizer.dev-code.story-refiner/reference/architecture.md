# Architecture — story-refiner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`StoryEndpoint` is the entry point. It writes a `BriefSubmitted` event to `RequestQueue` (event-sourced for audit replay). A `Consumer` subscribes to that queue and starts a `RefinementWorkflow` per submission. The workflow alternates two agents — `StoryDrafterAgent` produces a structured story draft, `ReviewerAgent` scores it — feeding reviewer notes back into the next draft on each revision cycle. Each step emits an event on `StoryEntity`. `StoriesView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `RequestSimulator` drips a sample brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first draft is sent back for revision and the second draft is approved. The Drafter → Reviewer alternation is explicit. Each attempt's events are written to `StoryEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly. The 202 response returns immediately after `BriefSubmitted` is appended; the rest of the loop is asynchronous and delivered via SSE.

## State machine

The story moves between two transient states (`DRAFTING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The `REVIEWING → DRAFTING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is exhausted with no approval; both terminal states preserve every attempt and every review on the entity.

Unlike content-generation loops that carry a separate pre-reviewer guardrail step, story refinement has no deterministic format ceiling to enforce. The reviewer's rubric covers all quality dimensions directly, so the state machine has no self-loop on `DRAFTING`.

## Entity model

`StoryEntity` is the system's source of truth; every transition writes one of six event types. `RequestQueue` is the audit log of submissions — separate from `StoryEntity` so submission replay and story lifecycle remain independently queryable. `StoriesView` is the only read-side projection; the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `draftStep` and `reviewStep`; both call agents whose latency depends on the model provider and story length.
- Workflow-wide deadline: the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `StoryEndpoint.submit` deduplicates on `(featureDescription, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(storyId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Story.attempts` with its own draft and (once produced) review. The `attemptNumber` is monotonic across the whole loop.
