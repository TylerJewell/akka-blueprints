# Architecture — nurse-handover

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `HandoverEndpoint` accepts a submission and writes a `ReportSubmitted` event onto `HandoverEntity`. The `ReportSanitizer` Consumer subscribes, redacts PHI, and writes the sanitized report back via `attachSanitized`. The same Consumer then starts a `HandoverWorkflow` instance. The workflow's `summarizeStep` calls `HandoverAgent` — the single AutonomousAgent — with the ward checklist as `TaskDef.instructions(...)` and the sanitized report as a `TaskDef.attachment(...)`. Once the summary returns, the workflow writes `SummaryRecorded` and advances to `awaitSignoffStep`. That step polls the entity until a clinician calls `PATCH /api/handovers/{id}/signoff`, which triggers `requestSignoff` on the entity and is detected on the next poll cycle. `HandoverView` projects every entity event into a read-model row; `HandoverEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The sign-off gate is a workflow polling loop, not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1 + J2). Two distinct pauses exist:

1. The `ReportSanitizer` subscription lag between `ReportSubmitted` and `ReportSanitized` — sub-second in normal operation, bounded by the `awaitSanitizedStep` 15 s timeout.
2. The `awaitSignoffStep` polling loop — polls every 5 s up to the 4-hour step timeout, advancing as soon as `handover.signoff().isPresent()` returns true after the clinician's PATCH call.

The agent call is bounded by `summarizeStep`'s 60 s timeout. The transition from `SUMMARY_READY` to `AWAITING_SIGNOFF` happens within the same workflow step — the entity command `requestSignoff` fires immediately after `recordSummary` completes.

## State machine

Seven states. The notable paths:

- The happy path is `SUBMITTED → SANITIZED → SUMMARIZING → SUMMARY_READY → AWAITING_SIGNOFF → SIGNED_OFF`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or iteration-budget exhaustion) during `SUMMARIZING`. A `FAILED` handover's prior data is preserved on the entity — the UI shows the partial state so the clinical team knows to fall back to manual handover.
- There is no `APPROVED` or `REJECTED` state. The summary is advisory; `SIGNED_OFF` means the incoming clinician has read it and attested, not that they agree with every finding. The blueprint deliberately stops at `SIGNED_OFF`.

## Entity model

`HandoverEntity` is the source of truth. It emits seven event types. `HandoverView` projects every event into a row used by the UI. `ReportSanitizer` subscribes to entity events to compute the sanitized form. `HandoverWorkflow` both reads (`getHandover`) and writes (`markSummarizing`, `recordSummary`, `requestSignoff`, `completeSignoff`, `fail`) on the entity. `HandoverEndpoint` also writes (`requestSignoff`) when the clinician PATCHes the signoff endpoint directly from the UI.

## Governance flow

For any handover that transitions to `SIGNED_OFF`, the shift report passed through:

1. **PHI sanitizer** — the model never sees patient identifiers; the audit log retains the raw form.
2. **HandoverAgent** — one model call, one structured output.
3. **Clinical sign-off gate** — the summary sits in `AWAITING_SIGNOFF` until a named clinician attests. No automated path to `SIGNED_OFF` exists.

Each control is independent. Removing the sanitizer exposes PHI to the LLM. Removing the sign-off gate allows unsigned summaries to be sealed without clinician attestation. Neither failure is silently absorbed by the other control.
