# Architecture — blackboard-swe-coordination

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`BoardEndpoint` is the entry point. A submitted ticket is logged as a `TicketSubmitted` event on `TicketQueue` (event-sourced for audit). `TicketConsumer` subscribes to that queue, creates a `TicketEntity`, and starts a `ControllerWorkflow`. The workflow is the blackboard controller: it reads the `BlackboardEntity` stage, determines which specialist has work to do, invokes that agent, and writes the result back to `BlackboardEntity`. Every stage transition projects into `BlackboardView` — the shared read model the UI streams.

Nine `AutonomousAgent` classes each own one stage on the board: `PlannerAgent`, `ArchitectAgent`, `BackendDevAgent`, `FrontendDevAgent`, `ReviewerAgent`, `SecurityAnalystAgent`, `TesterAgent`, `DocWriterAgent`, `IntegrationPlannerAgent`. No specialist talks to another directly; all knowledge flows through `BlackboardEntity`. When all stages are complete the controller starts `SignoffWorkflow`, which pauses for a lead-engineer HITL decision via `BoardEndpoint`. Two TimedActions run alongside: `TicketSimulator` drips sample tickets; `StuckBoardMonitor` marks boards that are idle too long.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → architect → code (backend and frontend in parallel) → review + security (parallel) → test + docs (parallel) → integrate → signoff pause → approval → `MERGED`. The `Note over BackendDevAgent` marks where `CodeValidator` vets the artifact before `BlackboardEntity` persists the event. All specialists are invoked by `ControllerWorkflow`; the SSE stream on `BlackboardView` keeps the UI current throughout.

## State machine

`BlackboardEntity` is the heart of the pattern. `INTAKE` is the initial state. Each stage transition is a named command on the entity. The two parallel stages (`CODED` after both backend and frontend, `REVIEWED` after both reviewer and security) require both results before advancing. `AWAITING_SIGNOFF` pauses indefinitely for a human decision: approval goes to `MERGED`; rejection goes to `IN_REVIEW`, which lets the controller re-run the review cycle without restarting planning or architecture.

## Entity model

`BlackboardEntity` is the source of truth for all specialist contributions. Every stage write emits a distinct event so the audit log is granular. `BlackboardView` is the only read-side projection — the controller reads the board via the view, not via direct entity commands. `TicketEntity` tracks the external ticket lifecycle. `SignoffEntity` records the HITL decision and the actor who made it. `TicketQueue` is the audit log of submissions.

## Governance architecture

- **G1 — before-state-write guardrail.** `CodeValidator` runs inside `BlackboardEntity`'s `writeBackendCode` and `writeFrontendCode` command handlers. A failing validation returns an error reply; the event is never persisted. The controller retries once with the validation errors injected into the agent prompt.
- **H1 — HITL application signoff.** `SignoffWorkflow` pauses at `waitStep` after all specialists have contributed. The workflow resumes only when `BoardEndpoint` receives a POST from the lead engineer. Approval advances the board to `MERGED`; rejection returns to `IN_REVIEW` for a reviewer re-run. No merge proceeds without an explicit human decision.

## Concurrency notes

- `BlackboardEntity` is a single writer. All specialist writes are serialised through it — no two parallel agents can corrupt the board.
- Parallel agent invocations in `codeStep`, `reviewStep`, and `verifyStep` wait for both results before advancing the step. The workflow persists intermediate results so a crash mid-parallel-step does not lose work.
- `SignoffWorkflow.waitStep` is a durable pause — no polling loop, no timeout. The workflow is safe to restart; it re-reads the `SignoffEntity` to determine whether a decision was already recorded.
- `StuckBoardMonitor` uses wall-clock comparison against `createdAt`. It does not force transitions — it marks `stuckAlert` so an operator can investigate without the system making autonomous recovery decisions.
