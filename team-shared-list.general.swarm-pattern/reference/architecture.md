# Architecture — swarm-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SwarmEndpoint` is the entry point. A submitted job is logged as a `JobSubmitted` event on `SubmissionLog` (event-sourced for audit). `JobRequestConsumer` subscribes to that log, creates a `JobEntity`, and starts a `DecompositionWorkflow`. The workflow runs the `Coordinator` agent to decompose the brief, then writes one `WorkItemEntity` per item onto the shared list (each `OPEN`). Every item transition projects into `WorkListView` — the shared list the workers poll.

The worker side runs in parallel and independently. `Bootstrap` starts one `WorkerWorkflow` per worker id (`worker-1`, `worker-2`, `worker-3`). Each worker loop polls `WorkListView`, claims an eligible item on its `WorkItemEntity`, runs the `Worker` agent, runs the result through the quality gate, and marks the item done or stalled. When a worker is blocked on coordination it posts to a peer's `RelayMailbox`. `SwarmControl` (a key-value entity) holds the operator pause flag that the poll loop reads. Two TimedActions sit alongside: `JobSimulator` drips sample briefs; `StalledItemMonitor` releases stranded claims.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → decompose → items on the list → a worker claims, produces a result, passes the gate, and the item goes `DONE`. The worker loops are already polling before any item exists, so the only thing the decomposition adds is rows on the shared list for them to find.

## State machine

`WorkItemEntity` is the heart of the pattern. `OPEN` is the initial state. `claim` is atomic — only an `OPEN` item can be claimed, and the entity's single-writer guarantee means exactly one worker wins a contested item. From `CLAIMED` the worker `start`s into `IN_PROGRESS`, submits a result into `IN_REVIEW`, and the quality gate decides: `DONE` on pass, back to `IN_PROGRESS` on a retryable failure, `STALLED` once retries are exhausted. A relay request moves the item to `STALLED` directly. `StalledItemMonitor` returns a claimed-but-idle item to `OPEN` for liveness, and a relay reply (or operator reopen) returns a stalled item to `OPEN`.

## Entity model

`WorkItemEntity` is the source of truth for each unit of work; every transition writes one of nine event types, and `WorkListView` is the only read-side projection — the worker loops never query the entity directly, they read the shared list. `JobEntity` tracks the job lifecycle and owns the set of item ids. `SubmissionLog` is the audit log of submissions. `RelayMailbox` (one per worker) holds the cross-worker coordination messages.

## Concurrency & timeouts

- Atomic claim through `WorkItemEntity`'s single writer is the coordination primitive — no lock, no external queue.
- Per-step timeout: 90 s on the agent-calling steps (`DecompositionWorkflow.decomposeStep`, `WorkerWorkflow.workStep`).
- Idle workers are paused workflows with a 5 s resume timer, not busy loops.
- An item is eligible only when every `dependsOn` title resolves to a `DONE` item — checked client-side because the view exposes no enum-status filter.
- `StalledItemMonitor` releases an item claimed-but-idle for more than two minutes so a failed worker does not strand work.
- The quality gate (`QualityChecker`) is deterministic, so a given result always yields the same verdict.
