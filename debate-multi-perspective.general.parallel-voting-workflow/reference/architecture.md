# Architecture — parallel-voting-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`TaskEndpoint` is the entry point. It writes a `TaskReceived` event to `TaskQueue` (event-sourced for audit). A Consumer subscribes to that queue and starts a `VotingWorkflow` per submission. The workflow calls four agents — `TaskDispatcher` to brief, then `FeasibilityVoter`, `RiskVoter`, and `AlignmentVoter` in parallel, then `TaskDispatcher` again to aggregate. Each transition emits an event on `TaskEntity`. `TaskView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `TaskSimulator` drips sample tasks for demonstration, and `EvalSampler` ticks every 5 minutes to score one decided task for aggregation alignment, calling `AggregationJudge`.

## Interaction sequence

The sequence diagram traces the happy path (J1). Two notes matter. First, the brief step runs before any voter is dispatched — `TaskDispatcher` tells each voter exactly what angle to evaluate so the three perspectives stay distinct. Second, the `par` block: the three voters run **concurrently**, not in a chain. The workflow awaits all three via a CompletionStage zip with a 60-second per-step timeout, which is the heart of the debate-multi-perspective pattern.

## State machine

The task has five states. `SUBMITTED` is the initial state when `TaskCreated` fires. `VOTING` is reached once briefing completes and voters are dispatched. The task then lands in one of three terminal states: `DECIDED` (all votes returned and quorum met), `INCONCLUSIVE` (all votes returned but no quorum — fewer than two votes agree), or `PARTIAL` (a voter timed out; aggregation ran on the remaining votes). An `AlignmentScored` event may land on a `DECIDED` or `PARTIAL` task without changing its status.

## Entity model

`TaskEntity` is the source of truth; every transition writes one of nine event types. `TaskQueue` is the audit log of submissions. `TaskView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for each voter call and for the aggregation call. The default 5-second workflow step timeout is far too short for LLM calls (Lesson 4).
- Partial path: a voter timeout transitions to aggregation-from-partial rather than failing the whole workflow; `failureReason` names the missing voter.
- Quorum check: `quorumStep` counts matching ballots after aggregation. Fewer than two matches → `INCONCLUSIVE`; two or more → `DECIDED`.
- Idempotency: `(title, submittedBy)` over a 10 s window deduplicates `POST /api/tasks`.
- View indexing: `getAllTasks` has no `WHERE status` clause — the `TaskStatus` enum cannot be auto-indexed (Lesson 2); callers filter client-side.
- Eval sampler: one task per 5-minute tick; the oldest `DECIDED` or `PARTIAL` task without an `alignmentScore` wins.
