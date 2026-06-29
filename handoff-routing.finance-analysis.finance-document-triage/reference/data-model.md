# Data model — finance-document-triage

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingDocument` | `documentId` | `String` | no | Server-assigned uuid. |
| | `sourceId` | `String` | no | Opaque source-system reference (SFTP drop, S3 key, webhook id). |
| | `docFormat` | `String` | no | `"pdf"` / `"xml"` / `"csv"` / `"json"`. |
| | `rawContent` | `String` | no | Raw text or structured payload as string; *not yet sanitized*. |
| | `subject` | `String` | no | Human-readable document description; raw. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedDocument` | `redactedContent` | `String` | no | Both PII and sector fields redacted. |
| | `redactedSubject` | `String` | no | Subject after PII redaction. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["name","account-number","tax-id","national-id"]`. |
| | `sectorFieldsRedacted` | `List<String>` | no | e.g. `["aml-flag","credit-score","fund-code","regulatory-amount"]`. |
| `ClassificationDecision` | `docType` | `DocumentType` | no | `INVOICE` / `LOAN_APPLICATION` / `COMPLIANCE_REVIEW`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `ProcessingResult` | `resultSummary` | `String` | no | One sentence ≤ 100 characters. |
| | `resultDetail` | `String` | no | 2–4 paragraphs. |
| | `action` | `ProcessingAction` | no | `APPROVED` / `FLAGGED_FOR_REVIEW` / `INFORMATION_REQUESTED` / `REJECTED` / `FORWARDED_TO_COMPLIANCE` / `ESCALATED`. |
| | `processorTag` | `String` | no | `"invoice"` / `"loan"` / `"compliance"`. |
| | `referenceId` | `Optional<String>` | yes | Downstream reference id if issued. |
| | `processedAt` | `Instant` | no | When the processor finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `ClassificationScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Document` (entity state) | `documentId` | `String` | no | — |
| | `incoming` | `IncomingDocument` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedDocument>` | yes | Populated after `DocumentSectorSanitized`. |
| | `classification` | `Optional<ClassificationDecision>` | yes | Populated after `DocumentClassified`. |
| | `result` | `Optional<ProcessingResult>` | yes | Populated after `ResultDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `classificationScore` | `Optional<ClassificationScore>` | yes | Populated after `ClassificationScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `DocumentEscalated`. |
| | `status` | `DocumentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DocumentReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Document` record is reused as the `DocumentView` row type.

## Enums

`DocumentType`: `INVOICE`, `LOAN_APPLICATION`, `COMPLIANCE_REVIEW`.

`ProcessingAction`: `APPROVED`, `FLAGGED_FOR_REVIEW`, `INFORMATION_REQUESTED`, `REJECTED`, `FORWARDED_TO_COMPLIANCE`, `ESCALATED`.

`DocumentStatus`: `RECEIVED`, `PII_SANITIZED`, `SECTOR_SANITIZED`, `CLASSIFIED`, `ROUTED_INVOICE`, `ROUTED_LOAN`, `ROUTED_COMPLIANCE`, `RESULT_DRAFTED`, `BLOCKED`, `PROCESSED`, `ESCALATED`.

## Events (`DocumentEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `DocumentReceived` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `DocumentPiiSanitized` | intermediate redacted payload | `PiiSanitizer.attachPiiSanitized` | → `PII_SANITIZED` |
| `DocumentSectorSanitized` | `sanitized` (final `SanitizedDocument`) | `SectorSanitizer.attachSectorSanitized` | → `SECTOR_SANITIZED` |
| `DocumentClassified` | `classification` | `DocumentWorkflow.classifyStep` | → `CLASSIFIED` |
| `DocumentRouted` | `docType` | `DocumentWorkflow.routeStep` | → `ROUTED_INVOICE` or `ROUTED_LOAN` or `ROUTED_COMPLIANCE` |
| `ResultDrafted` | `result` | `DocumentWorkflow.invoiceStep` / `loanStep` / `complianceStep` | → `RESULT_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `DocumentWorkflow.guardrailStep` | (no status change) |
| `ResultPublished` | — | `DocumentWorkflow.publishStep` (guardrail allowed) | → `PROCESSED` (terminal) |
| `ResultBlocked` | `violations` | `DocumentWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `DocumentEscalated` | `escalationReason` | `DocumentWorkflow.error` (unrecoverable) | → `ESCALATED` (terminal) |
| `ClassificationScored` | `score, rationale, scoredAt` | `ClassificationEvalScorer` Consumer | (no status change; populates `classificationScore`) |

A `BLOCKED → PROCESSED` transition is emitted via `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResultPublished` in the same command.

## Events (`DocumentQueue`)

| Event | Payload |
|---|---|
| `InboundDocumentReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`DocumentRow` is `Document` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS documents FROM document_view` with no `WHERE docType` or `WHERE status` filter — callers (`DocumentEndpoint`) apply those client-side.
