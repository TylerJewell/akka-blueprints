# Architecture — Contract-Net Task Auctioneer

## Component Graph

The component graph in PLAN.md §1 shows the full dependency structure. The HTTP endpoint is the only inbound surface: all external callers — task requestors, contractor agents, and UI clients — communicate through `TaskAuctioneerEndpoint`. The endpoint translates HTTP verbs into entity commands or workflow triggers; it never reads entity state directly for list responses (that is the view layer's job).

`TaskAnnouncementEntity` and `ContractorRegistryEntity` are the two authoritative state stores. Both are event sourced: every state change is recorded as an immutable event before any response is returned. This means the full history of every auction — who bid, what the evaluator decided, which guardrail checks ran, and what the performance scorer returned — is permanently queryable from the journal.

`AuctionWorkflow` is the protocol conductor. It starts when `TaskAnnounced` is emitted and does not complete until either `PerformanceScored` is written (happy path) or the renegotiation attempt counter is exhausted. Because workflows are durable, a service restart mid-auction resumes from the last committed step without re-running prior steps.

`RenegotiationWorkflow` is a short-lived child workflow started by `AuctionWorkflow` when `TaskFailed` arrives. It carries the failed contractor's ID so the guardrail step in the reopened auction can exclude them.

The two views — `ActiveAuctionsView` and `ContractorLeaderboardView` — are read-side projections. They subscribe to entity events and maintain their own queryable state. The SSE endpoints for `/auctions/active` and `/contractors/leaderboard` stream from these views, so the UI receives push updates without polling.

## Sequence — Happy Path

The sequence diagram in PLAN.md §2 traces a single auction from announcement through performance scoring. Key observations:

- The bidding loop is open-ended: any number of bids arrive during the window.
- The evaluator is invoked once, after the window closes, with the full bid list. It never sees partial bid sets.
- The guardrail runs synchronously inside the workflow between the evaluator step and the AwardTask command. If the top-ranked bid fails the guardrail, the workflow promotes the next bid and re-checks — no LLM call is repeated for this.
- The scorer is invoked after `TaskCompleted`; the workflow waits for the score before writing `PerformanceScored`. If the scorer is unavailable, the workflow retries up to three times before recording a sentinel score.

## State Machine

The state machine in PLAN.md §3 shows the full lifecycle of a `TaskAnnouncementEntity`. The key branch point is after `IN_PROGRESS`: `COMPLETED` is terminal, while `FAILED` feeds into `RENEGOTIATING`, which loops back to `BIDDING`. The renegotiation loop can repeat; `RenegotiationWorkflow` tracks the attempt counter and can halt the loop after a configurable maximum.

## Entity-Relationship Diagram

The ER diagram in PLAN.md §4 shows four persistent record types. `TASK_ANNOUNCEMENT` is the root: it holds the task spec and links to the winning bid. `BID` is child to both `TASK_ANNOUNCEMENT` (many bids per task) and `CONTRACTOR_PROFILE` (many bids per contractor over time). `PERFORMANCE_RECORD` is written once per task end and links to both the task and the contractor, giving a complete history of which contractor did what work and how it was scored. The `CONTRACTOR_PROFILE.performanceScore` field is a denormalized rolling average, recalculated from the last five `PERFORMANCE_RECORD` rows on each `UpdateScore` command.

## Governance Control Wiring

The two controls are not bolt-on: they are native steps in `AuctionWorkflow`.

- **G-001 (guardrail)** sits between the evaluator step and the AwardTask command. Nothing in the system can commit an award without passing through it, because `AwardTask` is only sent by `AuctionWorkflow` and only after the guardrail step succeeds.

- **E-001 (eval-event)** sits between the `TaskCompleted`/`TaskFailed` event and the `PerformanceScored` write. The throttle logic lives in `ContractorRegistryEntity`, which means even if the workflow were bypassed and `UpdateScore` were called directly via a backoffice route, the throttle check would still run.

This wiring ensures that the controls are observable (every invocation produces a journal event) and enforceable (neither can be skipped by alternative code paths).
