# Architecture — hrsd-inquiry

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `InquiryEndpoint` accepts a submission and writes an `InquirySubmitted` event onto `InquiryEntity`. The `SpecialCategoryScreener` Consumer subscribes, redacts special-category data, and writes the screened message back via `attachScreened`. The same Consumer then starts an `InquiryWorkflow` instance. The workflow's `answerStep` calls `HrInquiryAgent` — the single AutonomousAgent — with the screened message as `TaskDef.instructions(...)` and the relevant policy catalog entries as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ResponseGuardrail`) validates each candidate response. Once a response passes, the workflow writes `InquiryAnswered` and — if the employee opted in and the agent produced a service request — calls `HrRequestSubmitter` in `submitRequestStep`. The confirmation reference lands as `ServiceRequestSubmitted`. `InquiryView` projects every entity event into a read-model row; `InquiryEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The service-request submitter (`HrRequestSubmitter`) is deterministic — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system pauses:

1. The `SpecialCategoryScreener` subscription lag between `InquirySubmitted` and `InquiryScreened` — sub-second in normal operation.
2. The `awaitScreenedStep` polling loop inside the workflow — polls `InquiryEntity` every 1 s up to its 15 s timeout, advancing as soon as `inquiry.screened().isPresent()` returns true.

The agent call itself is bounded by `answerStep`'s 60 s timeout. The `submitRequestStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The full happy path (with service request submission) is `SUBMITTED → SCREENED → ANSWERING → ANSWERED → REQUEST_SUBMITTED`.
- For inquiries without a service request (or where the employee did not opt in), the terminal happy state is `ANSWERED`.
- Two failure transitions land in `FAILED`: a screener error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ANSWERING`. A `FAILED` inquiry's prior data is preserved on the entity — the UI shows the partial state for HR review.
- There is no `APPROVED` or `COMPLETED` state. The agent's answer is advisory; a human HR representative or the employee acts outside the system. The blueprint deliberately stops at `REQUEST_SUBMITTED`.

## Entity model

`InquiryEntity` is the source of truth. It emits six event types. `InquiryView` projects every event into a row used by the UI. `SpecialCategoryScreener` subscribes to entity events to compute the screened form. `InquiryWorkflow` both reads (`getInquiry`) and writes (`markAnswering`, `recordAnswer`, `recordServiceRequest`, `fail`) on the entity. The relationship between `HrInquiryAgent` and `InquiryResponse` is "returns" — the agent's task result is the response record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the employee message passed through:

1. **Special-category data screener** — the model never sees health conditions, religious preferences, union membership, or disability disclosures; the audit log retains the raw form.
2. **HrInquiryAgent** — one model call, one structured output with mandatory policy citations.
3. **before-agent-response guardrail** — uncited answers, unrecognized request types, and malformed responses are caught before the response leaves the agent loop.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
