# Architecture — `recruitment-team`

Narrative for the four diagrams in [`PLAN.md`](../PLAN.md). The generated UI renders the same diagrams on the Architecture tab.

## Component graph

An application enters through `ScreeningEndpoint` (a recruiter POST) or `ApplicationSimulator` (a timed drip from a canned file). Both land on `InboundApplicationQueue`, whose events drive `ApplicationConsumer`. The consumer starts one `ScreeningWorkflow` per application. The workflow is the supervisor: it redacts the resume, delegates matching and screening to the two worker agents, asks `SupervisorAgent` to aggregate, then writes lifecycle events to `CandidateEntity`. `CandidatesView` projects those events for the endpoint to query and stream. Two timed monitors read the view: `FairnessDriftMonitor` flags skew across recent decisions, `StuckDecisionMonitor` escalates decisions left unresolved.

## Interaction sequence

The primary journey runs from enqueue to human gate. The sanitizer runs before any agent call, so protected attributes never reach a model. Match, screen, and supervise run in order; the workflow then records `DecisionRequested` and parks the candidate in `AWAITING_DECISION`. The `Note over` block marks the human gate — the workflow self-resumes on a short timer until an approve or reject command arrives.

## State machine

`CandidateEntity` walks `RECEIVED → SANITIZED → MATCHED → SCREENED → AWAITING_DECISION`, then to one terminal state: `APPROVED`, `REJECTED`, or `ESCALATED`. `driftFlagged` is an orthogonal flag set by the fairness watch; it does not change the lifecycle state.

## Entity model

`CandidateEntity` and `InboundApplicationQueue` are the two event-sourced entities. `CandidateEntity` projects into `CandidatesView`, the single read model the UI and monitors consume. Nullable lifecycle fields on the `Candidate` row record are `Optional<T>` so the view materializer accepts the record before those events fire (Lesson 6).
