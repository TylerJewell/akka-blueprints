# Architecture — lead-qualifier

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `LeadEndpoint` accepts a `{rawInquiry}` POST, writes `LeadCreated` onto `LeadEntity`, and starts `LeadPipelineWorkflow` keyed by `"pipeline-" + leadId`. The workflow's first step (`captureStep`) emits `CaptureStarted`, then calls `InquiryAgent` with `TaskDef.taskType(CAPTURE_INQUIRY)` and a `phase = CAPTURE` metadata tag. The agent invokes `CaptureTools.parseContact` and `CaptureTools.detectIntent`; every call passes through `CrmWriteGuardrail` first. Once the agent returns an `InquiryForm`, the workflow writes `InquiryCaptured` onto the entity and advances to `qualifyStep` — same pattern, the QUALIFY task carries `phase = QUALIFY`. Then `enrichStep` runs with `phase = ENRICH`; on ENRICH-phase tool calls `CrmWriteGuardrail` additionally validates CRM schema and delegates to `PiiSanitizer` before the tool body runs. After `CrmEntryWritten` lands, `evalStep` runs `QualityScorer` over the recorded `(CrmEntry, LeadScore, InquiryForm)` triple — no LLM call — and writes `EvaluationScored`. `LeadView` projects every event into a read-model row; `LeadEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` and `PiiSanitizer` are deterministic rule-based classes; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `captureStep` and `qualifyStep`, the workflow writes `InquiryCaptured` onto the entity. The next step then reads `form` from the entity to build the QUALIFY task's instruction context. The agent never sees capture-phase context inside the qualify task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `CrmWriteGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `LeadEntity.status`, applies the per-phase accept matrix, and — for ENRICH-phase tools — validates CRM schema before proceeding. A misordered or schema-invalid call is rejected before the tool body executes.
3. `PiiSanitizer` runs inside the guardrail on every accepted ENRICH-phase call. The tool body receives a masked payload; raw email, phone, and full name never reach the CRM tools, the tool logs, or the SSE event stream.

The agent calls themselves are bounded by per-step timeouts (60 s on capture / qualify / enrich). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → CAPTURING → CAPTURED → QUALIFYING → QUALIFIED → ENRICHING → ENRICHED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `CAPTURING`, `QUALIFYING`, or `ENRICHING`. A `FAILED` lead's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `ACTIVE` state. The CRM entry is advisory; the human SDR reviews it and acts in their CRM directly. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`LeadEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `LeadView` projects every event into a row used by the UI. `LeadPipelineWorkflow` both reads (`getLead`) and writes (`startCapture`, `recordForm`, `startQualify`, `recordScore`, `startEnrich`, `recordCrmEntry`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `InquiryAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any lead that lands in the entity log, the raw inquiry passed through:

1. **Phase-gate + CRM schema guardrail** — every tool call is filtered. An ENRICH-phase tool called during CAPTURE is rejected before the tool body runs; a stage value outside the allowed enum is rejected before the CRM write executes. A `GuardrailRejected` event records every violation for audit.
2. **InquiryAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **PII sanitizer** — every accepted ENRICH-phase tool call passes through `PiiSanitizer` before the tool body runs. Raw contact fields never appear in outbound payloads.
4. **On-decision evaluator** — every emitted `CrmEntry` gets a 1–5 data-quality score. Stage validity, owner assignment, notes presence, and fit-score consistency are each worth one point on a base of 1.

Each layer is independent. The guardrail does not check data quality; the evaluator does not check phase order; the sanitizer does not check schema. Removing one layer opens an explicit gap the others do not silently cover.
