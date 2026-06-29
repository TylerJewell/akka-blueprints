# Architecture — finance-document-triage

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`DocumentSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundDocumentReceived` events into `DocumentQueue` (event-sourced for audit). Two Consumers handle the dual-pass sanitization pipeline before any LLM call: `PiiSanitizer` subscribes to `DocumentQueue`, redacts personal identifiers, and calls `DocumentEntity.attachPiiSanitized`. `SectorSanitizer` subscribes to the resulting `DocumentPiiSanitized` entity event, applies the second redaction pass for AML flags, credit scores, fund codes, and regulatory amounts, then starts a `DocumentWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClassifierAgent` to classify the sanitized document, then branches: `INVOICE` invokes `InvoiceProcessor`, `LOAN_APPLICATION` invokes `LoanApplicationProcessor`, `COMPLIANCE_REVIEW` invokes `ComplianceReviewProcessor`. The chosen processor owns the `PROCESS` task end-to-end and returns a typed `ProcessingResult`. The workflow then calls `OutputGuardrail` to check the draft against the policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the document waits until an operator unblocks via `POST /api/documents/{id}/unblock`.

A third Consumer, `ClassificationEvalScorer`, runs independently of the workflow. It subscribes to `DocumentEntity` events; on every `DocumentClassified` it invokes `ClassificationJudge` and writes a `ClassificationScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the invoice happy path). Two distinct concurrent flows merge into `DocumentEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `invoiceStep` → `guardrailStep` → `publishStep`.
2. The eval path: `ClassificationEvalScorer` observes `DocumentClassified` and writes `ClassificationScored` in parallel.

Both write to the same `DocumentEntity`; the entity's commands are idempotent on `documentId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a classification score appear within ~10 s of the routing decision regardless of which path completes first.

The two-Consumer sanitization chain (steps 1–6 in the sequence) is a distinctive feature of this blueprint. It ensures that both PII and sector-regulated fields are redacted before any step in the workflow executes.

## State machine

Eleven states. The notable paths:

- The document passes through `RECEIVED` → `PII_SANITIZED` → `SECTOR_SANITIZED` before classification begins. These two transitions are sequential: `SectorSanitizer` waits for the `DocumentPiiSanitized` event before running its pass.
- After `CLASSIFIED`, the `docType` value branches the flow. `INVOICE`, `LOAN_APPLICATION`, and `COMPLIANCE_REVIEW` each enter their respective `ROUTED_*` state, then converge at `RESULT_DRAFTED` once the processor returns.
- After `RESULT_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `PROCESSED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `PROCESSED`. There is no auto-timeout — blocked documents wait indefinitely.

`ClassificationScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`DocumentEntity` is the source of truth and emits eleven event types covering the full lifecycle: two sanitization stages, classification, routing, draft, guardrail verdict, publish, block, escalate, and the eval score. `DocumentQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `DocumentView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any processing result that is published, the document passed through:

1. **PII sanitizer** — personal identifiers (names, account numbers, tax ids, national ids) are redacted. No LLM ever sees them.
2. **Sector sanitizer** — AML flags, credit scores, fund codes, and regulatory amounts are redacted. No LLM ever sees them.
3. **ClassifierAgent** — typed classifier with a default-to-`COMPLIANCE_REVIEW` policy under ambiguity, ensuring that borderline documents get human-assisted disposition rather than being silently routed to the wrong processor.
4. **Processor agent** — owns the result end-to-end with a tightly-scoped prompt (no approval amount invention, no credit conclusions for the loan intake processor, mandatory forwarding for the compliance processor).
5. **OutputGuardrail** — typed before-agent-response check against the policy rubric. Blocks invented approval amounts, fabricated credit conclusions, autonomous regulatory conclusions, invented reference ids, and `[REDACTED]` echoes.
6. **ClassificationEvalScorer** — out-of-band scoring of every routing decision.

Steps 1–2 are preventive (the models can't misuse what they never see). Step 3 is conservative (ambiguity routes toward human review). Step 5 is detective (drafts that violate are held). Step 6 is continuous (sustained low scores would signal a classifier regression).
