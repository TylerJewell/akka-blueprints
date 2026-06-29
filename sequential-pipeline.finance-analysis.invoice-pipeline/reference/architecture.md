# Architecture — invoice-processing

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `InvoiceEndpoint` accepts a `{rawText}` POST, writes `InvoiceCreated` onto `InvoiceEntity`, and starts `InvoicePipelineWorkflow` keyed by `"pipeline-" + invoiceId`. The workflow's first step (`extractStep`) emits `ExtractionStarted`, then calls `InvoiceAgent` with `TaskDef.taskType(EXTRACT_INVOICE)` and a `phase = EXTRACT` metadata tag. The agent invokes `ExtractTools.parseHeader` and `ExtractTools.parseLineItems`; every call passes through `PhaseGuardrail` first. Once the agent returns an `ExtractedInvoice`, the workflow hands it to `VendorPiiSanitizer`, which strips `vendorEmail` and `vendorPhone` and returns a `SanitizationReport`. The workflow writes `SanitizationApplied` and `ExtractionCompleted` onto the entity — in that order — ensuring the sanitized data is what lands in the event log. The `validateStep` runs next: it reads the sanitized extract from the entity, builds the VALIDATE task's context, and calls the agent again. After `ValidationCompleted` lands, `approvalStep` checks the invoice amount against the configured threshold and either continues immediately or suspends for a reviewer decision. `postStep` runs the POST task, and `evalStep` runs `PostingQualityScorer` deterministically. `InvoiceView` projects every event into a read-model row; `InvoiceEndpoint` serves it to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `VendorPiiSanitizer` and `PostingQualityScorer` are both deterministic; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1, below-threshold invoice). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `extractStep` and `validateStep`, the workflow writes `SanitizationApplied + ExtractionCompleted` onto the entity. The next step reads the sanitized `ExtractedInvoice` from the entity to build the VALIDATE task's instruction context. The agent never sees extract-phase context inside the validate task's conversation; the typed handoff through the entity is the only path information travels.
2. Every tool call is filtered through `PhaseGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `InvoiceEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.

Agent steps are bounded by per-step timeouts (60 s on extract / validate / post). `approvalStep` carries a 48-hour timeout matching the reviewer's expected decision window. `evalStep` is synchronous and finishes in milliseconds.

## State machine

Twelve states. The interesting paths:

- The happy-path below threshold: `CREATED → EXTRACTING → EXTRACTED → VALIDATING → VALIDATED → POSTING → POSTED → EVALUATED`.
- The high-value path: `VALIDATED → APPROVAL_REQUESTED → APPROVED → POSTING → POSTED → EVALUATED`.
- The denial path: `APPROVAL_REQUESTED → DENIED` (terminal, no posting).
- Four failure transitions from `EXTRACTING`, `VALIDATING`, `POSTING`, and `APPROVAL_REQUESTED` (timeout) → `FAILED`.

`PhaseGuardrailRejected` is a side-event recorded for audit; it does not change the status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `REVERSED` state. The pipeline stops at `EVALUATED`; reversal requires a human-initiated correcting entry outside the system.

## Entity model

`InvoiceEntity` is the source of truth. It emits fourteen event types across the lifecycle. `SanitizationApplied` always precedes `ExtractionCompleted` in the log — this ordering is the auditable proof that PII was redacted before the extract data was committed. `InvoiceView` projects every event into a row used by the UI. `InvoicePipelineWorkflow` both reads (`getInvoice`) and writes (`startExtraction`, `recordExtraction`, `recordSanitization`, `startValidation`, `recordValidation`, `requestApproval`, `approve`, `deny`, `startPosting`, `recordPosting`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity.

## Defence-in-depth governance flow

For any invoice that lands in the entity log, the data passed through:

1. **Vendor PII sanitizer** — runs unconditionally after EXTRACT, before the first entity write. `vendorEmail` and `vendorPhone` are replaced with `"[REDACTED]"` in every event, projection, and SSE payload.
2. **InvoiceAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **HITL approval gate** — invoices above the configured threshold pause until a reviewer explicitly approves or denies. Automated posting cannot proceed without human sign-off on high-value amounts.
4. **On-decision evaluator** — every posted invoice receives a 1–5 quality score. GL account coverage, journal balance, line parity, and confirmation presence are each worth one point. A score ≤ 2 flags the card for the finance reviewer.

Each mechanism is independent. The sanitizer does not check balance; the HITL gate does not check PII; the evaluator does not check phase order. Removing one opens an explicit gap the others do not silently cover.
