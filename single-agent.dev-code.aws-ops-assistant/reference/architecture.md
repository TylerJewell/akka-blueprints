# Architecture — aws-ops-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `OpsEndpoint` accepts a submission, writes a `RequestSubmitted` event onto `OpsRequestEntity`, and starts `OpsWorkflow`. The workflow's `planStep` calls `AwsOpsAgent` — the single AutonomousAgent — with the request text and context. The agent's `before-tool-call` guardrail (`ActionGuardrail`) validates each planned MCP tool call before it fires. Allowed READ calls go straight to `AwsMcpStubs`; allowed MUTATING calls pause the workflow in `confirmStep`, which writes a `ConfirmationRequested` event and waits for a `ConfirmationReceived` event triggered by the user's Approve or Decline action on `OpsEndpoint`. Once all actions are resolved, the agent returns `OperationReport` to the workflow, which writes `ReportRecorded` onto the entity. `OpsView` projects every entity event into a read-model row; `OpsEndpoint` serves the read model to the UI over REST and SSE. An operator can POST to `/halt` at any point; the workflow observes the HALTED entity status between action dispatches and stops.

The graph deliberately has no second agent. The HITL confirmation and the operator halt are wired through the workflow and entity — not through additional LLM calls. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the J2 path: a mutating request that requires one HITL confirmation. Note three distinct pauses:

1. The `confirmStep` emitting `ConfirmationRequested` and the workflow blocking until `ConfirmationReceived` arrives — bounded by a 600 s step timeout.
2. The `before-tool-call` hook running on every MCP tool call — sub-millisecond in the allow-list check path.
3. The `AwsMcpStubs` response time — deterministic in the blueprint (simulated), variable in a real AWS deployment (API latency).

The `planStep` is bounded by a 15 s timeout. The `executeStep` is bounded by 120 s, which accommodates LLM latency across multi-action plans.

## State machine

Seven states. The notable paths:

- The read-only happy path is `SUBMITTED → PLANNING → EXECUTING → COMPLETED` (no confirmation needed).
- The mutating happy path adds `AWAITING_CONFIRMATION` between `PLANNING` and `EXECUTING`, and can loop back to `AWAITING_CONFIRMATION` for each subsequent mutating action in the plan.
- The halt transition can land from `AWAITING_CONFIRMATION` or `EXECUTING`, both reaching `HALTED` (terminal).
- `FAILED` is reached only on a hard workflow error (agent exception or step timeout exhaustion after retries).
- There is no `ROLLED_BACK` state. Mutation actions that have already completed are not undone — a deployer adding real AWS credentials must handle rollback outside this blueprint.

## Entity model

`OpsRequestEntity` is the source of truth. It emits nine event types. `OpsView` projects every event into a row used by the UI. `ConfirmationConsumer` subscribes to `ConfirmationRequested` events to confirm delivery to the UI before the workflow times out. `OpsWorkflow` both reads (`getRequest`) and writes (`startPlanning`, `requestConfirmation`, `recordConfirmationDelivered`, `receiveConfirmation`, `startExecution`, `recordReport`, `halt`, `fail`) on the entity. The relationship between `AwsOpsAgent` and `OperationReport` is "returns" — the agent's task result is the report record.

## Defence-in-depth governance flow

For any mutating action that executes, the call passed through:

1. **before-tool-call guardrail** — the target service and resource ARN were on the allow-list; the API call left the process with no restriction lifted.
2. **HITL confirmation** — a human explicitly approved this specific action, with the `decidedBy` field recorded in the entity log.
3. **Operator halt check** — the workflow verified the entity was not in HALTED status before dispatching this action.

Each layer is independent. A misconfigured allow-list does not bypass the HITL; a distracted operator who approves everything does not remove the allow-list; and a halt does not retroactively undo what already executed — it stops what comes next.
