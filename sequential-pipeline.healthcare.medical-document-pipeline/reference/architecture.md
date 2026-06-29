# Architecture — medical-document-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs two LLM tasks per document — EXTRACT and WRITE_SUMMARY — with a human review gate between them. `DocumentEndpoint` accepts a `{documentType, rawText}` POST, writes `DocumentReceived` onto `DocumentEntity`, and starts `DocumentPipelineWorkflow` keyed by `"pipeline-" + documentId`. The workflow's first step (`sanitizeStep`) runs deterministic PHI/PII masking in-process — no LLM call — and writes `DocumentSanitized{maskedDocument}` onto the entity, setting `sanitized = true`. The second step (`extractStep`) emits `ExtractionStarted`, then calls `MedicalDocumentAgent` with `TaskDef.taskType(EXTRACT_FIELDS)` and `phase = EXTRACT` metadata. The agent invokes `ExtractTools` methods on the masked text; every call passes through `SanitizationGuardrail` first, which checks both the `sanitized` flag and the phase order. Once the agent returns `ExtractedFields`, the workflow writes `FieldsExtracted` onto the entity and advances to `reviewGateStep`.

`reviewGateStep` parks the workflow. The clinician sees the extracted fields in the App UI via the SSE stream and submits a decision via `POST /api/documents/{id}/review`. That call writes `ReviewSubmitted{validationResult}` onto the entity, which wakes the workflow. The workflow then runs `summarizeStep` — same pattern as `extractStep`, but with `phase = SUMMARIZE` and the validated fields as instruction context. After `SummaryWritten` lands, `evalStep` runs `ClinicalAccuracyScorer` over the recorded triple — no LLM call — and writes `EvaluationScored`. `DocumentView` projects every event into a read-model row; `DocumentEndpoint` serves the read model over REST and SSE.

The graph has exactly one LLM-calling component. `ClinicalAccuracyScorer` and the PHI/PII masking logic are both deterministic and rule-based; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **Sanitization is a prerequisite, not a phase.** `sanitizeStep` runs before the agent is ever invoked. `SanitizationGuardrail` enforces this at runtime: even if a tool call arrives for an EXTRACT-phase tool, the guardrail will reject it if `sanitized` is `false`. The workflow ordering is the happy-path guarantee; the guardrail is the defence-in-depth check.
2. **The task boundary IS the dependency contract.** Between `extractStep` and `summarizeStep`, the workflow writes `FieldsExtracted` and then pauses for the human review. `summarizeStep` builds its instruction context from the clinician-approved `ValidationResult`, not the raw extraction. The agent never sees unapproved field values during summarization.
3. **Every tool call is filtered through `SanitizationGuardrail`.** The guardrail reads the in-flight task's `phase` metadata and the current `DocumentEntity.status`, applies the per-phase accept matrix, and also checks `sanitized`. A SUMMARIZE-phase tool called before `ReviewSubmitted` is rejected before the tool body executes.

## State machine

Eleven states. The interesting paths:

- The happy path walks `RECEIVED → SANITIZING → SANITIZED → EXTRACTING → EXTRACTED → AWAITING_REVIEW → REVIEW_SUBMITTED → SUMMARIZING → SUMMARIZED → EVALUATED`.
- Four failure transitions land in `FAILED`: errors during `SANITIZING`, `EXTRACTING`, `AWAITING_REVIEW` (review timeout), and `SUMMARIZING`. A failed document's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget, a step timeout, or a review timeout transitions to `FAILED`.

There is no `APPROVED` or `PUBLISHED` state. The summary is a clinical aid; the clinician submits it to the chart outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`DocumentEntity` is the source of truth. It emits twelve event types — two sanitization events, two extraction events, two review events, two summarization events, the evaluation, the guardrail audit, the failure, and the initial reception. `DocumentView` projects every event into a row used by the UI. `DocumentPipelineWorkflow` both reads (`getDocument`) and writes (eight command methods) on the entity. The relationship between `MedicalDocumentAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload. The clinician's review decision also writes onto the entity via the `POST /review` endpoint, making the human decision a first-class event in the audit trail.

## Defence-in-depth governance flow

For any document that lands in the entity log, the content passed through:

1. **PHI sanitizer (S1)** — health data fields masked in-process before any LLM call. `SanitizationGuardrail` verifies the `sanitized` flag at runtime.
2. **PII sanitizer (S2)** — patient identifiers tokenised in the same `sanitizeStep`. Tokens are not passed to the agent.
3. **SanitizationGuardrail (phase order + sanitization check)** — every tool call is filtered. A SUMMARIZE-phase tool called during EXTRACT is rejected; an EXTRACT-phase tool called before `DocumentSanitized` is rejected regardless of phase.
4. **Human-in-the-loop gate (H1)** — workflow pauses after extraction; clinician approves or corrects fields; summarization only runs against approved values.
5. **MedicalDocumentAgent (2 task runs)** — two model calls, two structured outputs (EXTRACT and WRITE_SUMMARY). Each task's typed result is the dependency handoff to the next phase.
6. **On-decision evaluator (E1)** — every emitted summary receives a 1–5 clinical accuracy score. Field coverage, value provenance, section completeness, and section word count are each worth one point on a base of 1.

Each layer is independent. Removing one opens an explicit gap the others do not silently cover.
