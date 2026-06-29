# Architecture — fraud-flagging-team

Narrative around the four diagrams in `PLAN.md`. The generated system renders the diagrams on the Architecture tab with the Akka theme variables plus the Lesson 24 state-label and edge-label CSS overrides.

## Component graph

A transaction enters one of two ways: `TransactionSimulator` (a TimedAction) drips canned lines from `transactions.jsonl` every 30 seconds, or `FraudEndpoint` accepts a POST. Both write to `TransactionQueue`, an event-sourced entity. `TransactionConsumer` subscribes to its events; for each queued transaction it opens a `CaseEntity` and starts a `FraudReviewWorkflow`.

The workflow is the supervisor of the delegation pattern at the orchestration level: it fans the transaction out to three worker agents — `FraudScoringWorker`, `ComplianceWorker`, `RiskAssessmentWorker` — collects their typed results, then calls `SupervisorAgent` to synthesize a verdict. The workflow writes case events; `CasesView` projects them for the API and SSE stream. `StuckCaseMonitor` reads the view and escalates flagged cases that age past the review window.

## Interaction sequence

The primary journey runs submit → delegate → synthesize → flag → human gate → action. The workflow pauses after flagging; the human analyst's confirm call writes to `CaseEntity`, and the workflow's next poll observes `CONFIRMED` and proceeds to the gated account action. The before-tool-call guardrail re-checks the status at the action step so a stale resume cannot act on a dismissed or escalated case.

## State machine

A case is `ANALYZING`, then `FLAGGED` or `CLEARED`. A flagged case is `CONFIRMED` or `DISMISSED` by the analyst, or `ESCALATED` by the monitor. A confirmed case becomes `ACTIONED` once the gated action runs. `CLEARED`, `DISMISSED`, `ESCALATED`, and `ACTIONED` are terminal.

## Entity model

`TransactionQueue` opens cases. Each `CaseEntity` emits the lifecycle events and projects into `CasesView`. The view row is the same `FraudCase` record the entity holds, with every nullable lifecycle field declared `Optional<T>` (Lesson 6).
