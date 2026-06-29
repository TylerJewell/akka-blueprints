# Architecture — dev-team-task-board

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`DevTeamEndpoint` is the entry point. A submitted project is logged as a `ProjectSubmitted` event on `IntakeQueue` (event-sourced for audit). `ProjectRequestConsumer` subscribes to that queue, creates a `ProjectEntity`, and starts a `PlanningWorkflow`. The workflow runs the `TeamLead` agent to decompose the brief, then writes one `TaskEntity` per task onto the board (each `OPEN`). Every task transition projects into `TaskBoardView` — the shared list.

The developer side runs in parallel and independently. `Bootstrap` starts one `DeveloperWorkflow` per developer id (`dev-1`, `dev-2`, `dev-3`). Each developer loop polls `TaskBoardView`, claims an eligible task on its `TaskEntity`, runs the `DeveloperAgent`, runs the artifact through the test gate, and marks the task done or blocked. When a developer is blocked on coordination it posts to a peer's `PeerMailbox`. `SystemControl` (a key-value entity) holds the operator halt flag that both the poll loop and the tool guardrail read. Two TimedActions sit alongside: `ProjectSimulator` drips sample briefs; `StuckTaskMonitor` releases stranded claims.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → tasks on the board → a developer claims, codes, passes the gate, and the task goes `DONE`. The `Note over DeveloperAgent` block marks where the before-tool-call guardrail vets every tool call. The developer loops are already polling before any task exists, so the only thing the plan adds is rows on the board for them to find.

## State machine

`TaskEntity` is the heart of the pattern. `OPEN` is the initial state. `claim` is atomic — only an `OPEN` task can be claimed, and the entity's single-writer guarantee means exactly one developer wins a contested task. From `CLAIMED` the developer `start`s into `IN_PROGRESS`, submits an artifact into `IN_REVIEW`, and the test gate decides: `DONE` on pass, back to `IN_PROGRESS` on a retryable failure, `BLOCKED` once retries are exhausted. A peer request or a guardrail refusal moves the task to `BLOCKED` directly. `StuckTaskMonitor` returns a claimed-but-idle task to `OPEN` for liveness, and a peer reply (or operator reopen) returns a blocked task to `OPEN`.

## Entity model

`TaskEntity` is the source of truth for each unit of work; every transition writes one of nine event types, and `TaskBoardView` is the only read-side projection — the developer loops never query the entity directly, they read the board. `ProjectEntity` tracks the project lifecycle and owns the set of task ids. `IntakeQueue` is the audit log of submissions. `PeerMailbox` (one per developer) holds the cross-developer messages.

## Concurrency & timeouts

- Atomic claim through `TaskEntity`'s single writer is the coordination primitive — no lock, no external queue.
- Per-step timeout: 90 s on the agent-calling steps (`PlanningWorkflow.decomposeStep`, `DeveloperWorkflow.workStep`).
- Idle developers are paused workflows with a 5 s resume timer, not busy loops.
- A task is eligible only when every `dependsOn` title resolves to a `DONE` task — checked client-side because the view exposes no enum-status filter.
- `StuckTaskMonitor` releases a task claimed-but-idle for more than two minutes so a failed developer does not strand work.
- The test gate (`TestRunner`) is deterministic, so a given artifact always yields the same verdict; the Maven `test` phase enforces the same gate at build time.
