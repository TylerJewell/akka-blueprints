# Architecture — autonomous-agent-pattern

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call, split across two task invocations on the same `CodingAgent` — one for planning (read-only), one for execution (read-write). `AgentTaskEndpoint` accepts a GitHub issue URL, mints a `taskId`, writes `TaskSubmitted` onto `AgentTaskEntity`, and starts `AgentTaskWorkflow`. The workflow's `fetchIssueStep` calls the GitHub REST API to retrieve the issue body, then hands it to `planStep` which calls `CodingAgent` with a PLAN-phase task definition. The plan is written back to the entity and the workflow waits at `checkpointStep` until a human approves or rejects the plan via the endpoint. On approval the workflow's `executeStep` calls `CodingAgent` again with the issue and approved plan as attachments. Every tool call during execution fires `ToolCallGuardrail` before any side effect. `RuntimeMonitor` subscribes to `ToolCallExecuted` events and runs anomaly rules without blocking the agent. When `CodingAgent` returns a `TaskOutcome`, `outcomeStep` writes `OutcomeRecorded` and the entity transitions to `PATCH_READY`. `AgentTaskView` projects every entity event into a read-model row; `AgentTaskEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second LLM-backed agent. `ToolCallGuardrail` is a rule-based validator, `HaltChecker` is a boolean helper, and `RuntimeMonitor` runs pattern-matching logic — none of these make LLM calls. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note four distinct pause points:

1. The `fetchIssueStep` network call to the GitHub API — typically sub-second on a warm connection.
2. The `planStep` agent call — one LLM iteration to produce an `AgentPlan` from the issue text.
3. The `checkpointStep` polling loop — waits up to 305 s for a human to approve or reject the plan. This is the most variable delay; a reviewer may take seconds or minutes.
4. The `executeStep` agent call — multi-iteration tool loop bounded by `maxIterationsPerTask(12)` and the 300 s step timeout.

The `RuntimeMonitor` events appear after the sequence's main flow: they are asynchronous and do not delay any step.

## State machine

Eight states. The key paths:

- **Happy path**: `SUBMITTED → ISSUE_FETCHED → PLAN_READY → CHECKPOINT_PENDING → EXECUTING → PATCH_READY`.
- **Rejection path**: `CHECKPOINT_PENDING → FAILED` (human rejected the plan).
- **Halt path**: `EXECUTING → HALTED` (operator pulled the kill switch).
- **Error paths**: `SUBMITTED → FAILED` (GitHub API failure), `ISSUE_FETCHED → FAILED` (plan agent error), `EXECUTING → FAILED` (execution agent error or guardrail exhaustion).

All terminal states (`PATCH_READY`, `HALTED`, `FAILED`) preserve the full event history and tool trace on the entity for post-mortem review.

## Entity model

`AgentTaskEntity` is the source of truth for one task. It emits thirteen event types covering the full lifecycle including individual `ToolCallExecuted` events (one per tool call, whether allowed, blocked, succeeded, or failed) and `MonitorAlertRaised` events. `AgentTaskView` projects all events into a row for the UI. `RuntimeMonitor` subscribes only to `ToolCallExecuted` events — the subscription is narrow by design to minimize processing load on the monitor. `AgentTaskWorkflow` both reads (`getTask`) and writes (`recordIssueFetched`, `recordPlan`, `markCheckpointPending`, `markExecuting`, `recordOutcome`, `fail`) on the entity.

## Defence-in-depth governance flow

For any `TaskOutcome` that lands in the entity log, the execution passed through:

1. **Before-tool-call guardrail** — every tool invocation was validated before running. Destructive calls were blocked and the block reason was returned to the agent.
2. **Human checkpoint** — the proposed `AgentPlan` was read and approved by a human before any `writeFile` or `runShell` call could occur.
3. **Halt mechanism** — an operator could have terminated the loop at any point between iterations.
4. **Runtime monitor** — the tool trace was watched for rate spikes, repeated failures, and suspicious shell patterns throughout execution.

Each layer is independent. Removing any one opens a gap the others do not silently cover: the guardrail cannot stop a runaway loop; the checkpoint cannot block a tool call in real time; the halt cannot prevent a plan from running if no operator is watching; the monitor cannot prevent a single destructive call from slipping through if the guardrail is misconfigured.
