# Architecture — sentiment-monitor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`CommentPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `CommentReceived` events into `CommentQueue` (event-sourced for audit). A `SentimentScoringConsumer` subscribes to that queue and starts a `SentimentScoringWorkflow` per comment. The workflow calls `SentimentScoringAgent` (typed scorer), writes the `CommentScored` event to `IssueThreadEntity`, and checks whether the updated consecutive-negative count has crossed the threshold. When it has, a second workflow — `AlertDispatchWorkflow` — starts.

`AlertDispatchWorkflow` runs the before-tool-call guardrail in `validateChannelStep`, then calls `TrendAnalysisAgent` for a Slack-ready summary, then calls the `postSlackAlert` tool stub. The dispatch result is written back to `IssueThreadEntity`.

`SentimentEvalRunner` runs alongside as a second TimedAction, ticking every 60 minutes and checking recent scores against ground-truth labels via `EvalJudge`.

## Interaction sequence

The sequence traces the happy path: a comment arrives, scores negative (five consecutive times on the same thread), and triggers the alert flow. Note that `AlertDispatchWorkflow` is a separate workflow instance from `SentimentScoringWorkflow` — the scoring workflow merely signals that the threshold was crossed; the alert workflow handles its own validation, summarisation, and dispatch.

The guardrail check in `validateChannelStep` is synchronous within the workflow step — there is no LLM call at that point. The LLM is invoked only in `summariseStep`, after the channel has already been validated.

## State machine

The state machine models `IssueThreadEntity`'s observable status. The `CRITICAL_ACTIVE` state is an overlay on `ACTIVE` — the thread continues to receive new comments while the alert is in flight. A `TrendUpdated` event (triggered when consecutive-negative count drops back below the threshold after new positive scores) can return a `CRITICAL_ACTIVE` thread to `ACTIVE`. This means repeated alert dispatch is possible if the thread yo-yos — the `AlertDispatchWorkflow` uses an `issueId + alertEpochSecond` composite workflow id to prevent double dispatch within the same trend window.

The `SILENCED` state suppresses further alert dispatch without clearing the scored-comment history. `RESOLVED` is terminal.

## Entity model

`IssueThreadEntity` is the primary source of truth for each Linear issue thread. `CommentQueue` is the upstream audit log — only `SentimentScoringConsumer` subscribes to it. `SentimentEvalEntity` accumulates drift-eval results independently; it is projected into `ThreadView` as a sub-row alongside the thread data.

## Governance layering

For any alert that reaches Slack, the following checks have occurred:
1. **SentimentScoringAgent** — typed classifier scored the comment in the [−5, +5] range.
2. **IssueThreadEntity** — threshold logic applied deterministically (no ML involved).
3. **AlertDispatchWorkflow.validateChannelStep (G1)** — channel allowlist and mention-scope guardrail enforced before the tool call.
4. **TrendAnalysisAgent** — alert summary produced; agent never calls the Slack tool directly.

The periodic drift watch (E1) monitors whether step 1 remains calibrated over time. Each component is independent — the guardrail fires regardless of what the agents produced.
