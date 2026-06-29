# Architecture — email-triage-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `TriageEndpoint` accepts a submission and writes an `EmailSubmitted` event onto `EmailEntity`. The `EmailSanitizer` Consumer subscribes, strips PII from the body and all attachment text content, and writes the sanitized email back via `attachSanitized`. The same Consumer then starts a `TriageWorkflow` instance. The workflow's `triageStep` calls `EmailTriageAgent` — the single AutonomousAgent — with the email metadata as `TaskDef.instructions(...)` and the sanitized body and attachment texts as `TaskDef.attachment(...)` calls. The agent has one declared tool: `send_email`. Every invocation of that tool is intercepted by `SendGuardrail`, which reads `ConfirmationEntity` before deciding to pass or block.

The `confirmStep` pauses execution after `DraftReady` and polls `ConfirmationEntity` every 2 s. The user's Approve or Reject action in the UI POSTs to `TriageEndpoint POST /api/emails/{id}/confirm`, which writes a `DecisionRecorded` event to `ConfirmationEntity`. The workflow then advances to `sendStep` (approved) or records `EmailRejected` (rejected). `TriageView` projects every `EmailEntity` event into a read-model row; `TriageEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The `SendGuardrail` is a structural check against entity state — not an LLM. The HITL `confirmStep` is a workflow pause — not a model call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the approve path (J2). Two distinct pauses occur:

1. The `EmailSanitizer` subscription lag between `EmailSubmitted` and `EmailSanitized` — sub-second in normal operation.
2. The `confirmStep` pause — the workflow idles until a `DecisionRecorded` event lands in `ConfirmationEntity`. This can take seconds or minutes depending on how quickly the user reads and approves the draft.

The agent call is bounded by `triageStep`'s 90 s timeout. `confirmStep` has a 300 s timeout — five minutes for the user to decide.

## State machine

Eight states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → REVIEWING → DRAFT_READY → SENDING → SENT`.
- The reject path branches at `DRAFT_READY`: `DRAFT_READY → REJECTED`. No outbound call is ever attempted on the reject path.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or timeout) during `REVIEWING`. A `FAILED` email's prior data is preserved on the entity — the UI shows the partial state for the operator.
- There is no automatic re-draft state. If a user rejects, they submit a new email; the blueprint deliberately stops at `REJECTED` without a loopback.

## Entity model

`EmailEntity` is the primary source of truth and emits eight event types. `ConfirmationEntity` is a separate entity that isolates the user's approve/reject decision from the email lifecycle, preventing write contention between the workflow's step transitions and the user's UI action. `TriageView` projects `EmailEntity` events into the read-model row. `EmailSanitizer` subscribes to `EmailEntity` events to compute the sanitized form. `TriageWorkflow` both reads (`getEmail`) and writes (`markReviewing`, `recordDraft`, `recordConfirmation`, `recordSent`, `recordRejected`, `fail`) on `EmailEntity`. `SendGuardrail` reads `ConfirmationEntity` synchronously on every `send_email` tool invocation.

## Defence-in-depth governance flow

For any email that reaches the `SENT` state, the content passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **EmailTriageAgent** — one model call, one structured `TriageResult`.
3. **Human-in-the-loop confirmation** — the user reads the draft and clicks Approve; no automated path bypasses this gate.
4. **before-tool-call guardrail** — even after the workflow advances to `sendStep`, the `send_email` tool call is individually checked against `ConfirmationEntity` before execution.

Each layer is independent. A deployer removing the HITL `confirmStep` would still face the guardrail; a deployer removing the guardrail would still face the HITL step. Neither alone is sufficient; both together make the send path explicit and auditable.
