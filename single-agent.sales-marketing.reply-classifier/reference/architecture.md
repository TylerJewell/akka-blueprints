# Architecture — reply-classifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ReplyEndpoint` accepts a submission, writes a `ReplyReceived` event onto `ReplyEntity`, and starts a `ClassificationWorkflow` instance. The workflow's `classifyStep` calls `ReplyClassifierAgent` — the single AutonomousAgent — with deal context as `TaskDef.instructions(...)` and the raw reply body as a `TaskDef.attachment(...)`. The agent proposes an `updateDealStage` tool call; `CrmMutationGuardrail` intercepts every proposal via the `before-tool-call` hook, checking allowed transitions and intent constraints before passing the call through. Once the classification lands, the workflow writes `ReplyClassified` and proceeds to `crmUpdateStep`, which calls `PipedriveClient` and writes either `CrmUpdateSucceeded` or `CrmUpdateSkipped`. `ReplyView` projects every entity event into a read-model row; `ReplyEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The CRM update is a deterministic workflow step calling a thin client interface — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the guardrail interception inside the agent loop: `CrmMutationGuardrail` fires on the proposed `updateDealStage` call before the tool executes. In the happy path it allows immediately. In J2 it rejects, the agent retries, and on the corrected call (or `SKIP_CRM_UPDATE`) the loop completes within the 3-iteration budget.

The CRM update is separated from the agent call: the agent only *proposes* the action; `ClassificationWorkflow.crmUpdateStep` makes the actual external call. This separation means the CRM mutation can be retried independently of the LLM call, and audit records for the agent response and the CRM outcome are stored as distinct events on `ReplyEntity`.

## State machine

Six states. The interesting paths:

- The happy path ending in a CRM update: `RECEIVED → CLASSIFYING → CLASSIFIED → CRM_UPDATED`.
- The skip path (UNSUBSCRIBE, NOT_INTERESTED, or guardrail-exhaustion): `RECEIVED → CLASSIFYING → CLASSIFIED → CRM_SKIPPED`.
- Two failure transitions land in `FAILED`: a workflow start error during `RECEIVED`, and an agent error or guardrail-exhaustion during `CLASSIFYING`. A `FAILED` reply's prior data is preserved on the entity — the UI shows the partial state for the sales rep.
- There is no `APPROVED` or `ACTIONED` state. The classification is the terminal output; the rep reads the outcome and any follow-up (e.g., manual unsubscribe, scheduling a call) happens outside the system.

## Entity model

`ReplyEntity` is the source of truth. It emits six event types. `ReplyView` projects every event into a row used by the UI. `ClassificationWorkflow` both reads (implicitly via the workflow's start data) and writes (`markClassifying`, `recordClassification`, `recordCrmUpdate`, `skipCrmUpdate`, `fail`) on the entity. The relationship between `ReplyClassifierAgent` and `ReplyClassification` is "returns" — the agent's task result is the classification record. `CrmMutationGuardrail` is wired as a hook on the agent, not as a standalone component; its relationship to the agent is "guards".

## Defence-in-depth governance flow

For any classification that reaches `CRM_UPDATED`, the proposed mutation passed through:

1. **ReplyClassifierAgent** — one model call, one structured output with intent + CRM action.
2. **before-tool-call guardrail** — invalid stage transitions, wrong deal ids, and forbidden-intent mutations are caught before the tool call executes.
3. **ClassificationWorkflow.crmUpdateStep** — the actual Pipedrive call, isolated in its own workflow step so it can be retried independently.

Each step is independent. Removing the guardrail opens a direct path from LLM intent to CRM mutation without a business-rule check.
