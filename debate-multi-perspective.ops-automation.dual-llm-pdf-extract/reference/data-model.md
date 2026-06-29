# Data model — dual-llm-pdf-extract

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DocumentSubmission` | `filename` | `String` | no | The document's filename. |
| | `extractedText` | `String` | no | The raw text extracted from the PDF (redacted before persistence; never stored). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ExtractedField` | `name` | `String` | no | Field name in snake_case (e.g., `total_amount`). |
| | `value` | `String` | no | The extracted string value. |
| | `confidence` | `double` | no | Extraction confidence 0.0–1.0. |
| `RawExtraction` | `modelFamily` | `String` | no | `CLAUDE` or `GEMINI`. |
| | `fields` | `List<ExtractedField>` | no | All fields extracted by this model. |
| | `extractorNotes` | `String` | no | One or two sentences from the extractor about uncertainty or document structure. |
| | `extractedAt` | `Instant` | no | When the extractor completed. |
| `FieldAgreement` | `fieldName` | `String` | no | The field where the models disagreed. |
| | `agreed` | `boolean` | no | `false` for every entry in the disagreementFields list. |
| | `claudeValue` | `String` | no | Value produced by ClaudeExtractor. |
| | `geminiValue` | `String` | no | Value produced by GeminiExtractor. |
| `MergedExtraction` | `fields` | `List<ExtractedField>` | no | Authoritative merged field list. |
| | `confidenceMap` | `Map<String,Double>` | no | Per-field merged confidence. |
| | `disagreementFields` | `List<FieldAgreement>` | no | Fields where the two models differed. |
| | `reconciliationNotes` | `String` | no | Two or three sentence reconciliation summary. |
| | `disagreementCount` | `int` | no | Count of disagreement fields. |
| | `reconciledAt` | `Instant` | no | When the reconciler completed. |
| `AgreementVerdict` | `score` | `int` | no | Cross-model agreement score 1–5. |
| | `rationale` | `String` | no | One-sentence reason for the score. |
| | `highDisagreementFields` | `List<String>` | no | Field names with high-confidence disagreements. |
| `RedactionResult` | `redactedText` | `String` | no | Text after PII redaction. |
| | `redactionCount` | `int` | no | Number of substitutions made. |
| `Extraction` (entity state, View row source) | `extractionId` | `String` | no | Unique id. |
| | `filename` | `String` | no | Document filename. |
| | `status` | `ExtractionStatus` | no | See enum. |
| | `redactedText` | `Optional<String>` | yes | Populated after DocumentSanitized. |
| | `redactionCount` | `Optional<Integer>` | yes | Populated after DocumentSanitized. |
| | `claudeExtraction` | `Optional<RawExtraction>` | yes | Populated after ClaudeExtractionAttached. |
| | `geminiExtraction` | `Optional<RawExtraction>` | yes | Populated after GeminiExtractionAttached. |
| | `mergedExtraction` | `Optional<MergedExtraction>` | yes | Populated after ExtractionReconciled. |
| | `failureReason` | `Optional<String>` | yes | Populated on ExtractionDegraded. |
| | `agreementScore` | `Optional<Integer>` | yes | Populated on AgreementScored. |
| | `agreementRationale` | `Optional<String>` | yes | Populated on AgreementScored. |
| | `createdAt` | `Instant` | no | When ExtractionCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the extraction reached a terminal state. |

Every nullable lifecycle field is `Optional<T>` because the View materializer rejects a row record with a non-optional `null` field (Lesson 6).

## Enums

`ExtractionStatus`: `INTAKE`, `EXTRACTING`, `RECONCILED`, `DEGRADED`.

## Events (`ExtractionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ExtractionCreated` | `extractionId, filename, createdAt` | Workflow `createStep` | → INTAKE |
| `DocumentSanitized` | `redactedText, redactionCount` | Workflow `sanitizeStep`, after `PiiSanitizer.redact` | → EXTRACTING |
| `ClaudeExtractionAttached` | `rawExtraction` | `ClaudeExtractor` returned | (no status change; populates `claudeExtraction`) |
| `GeminiExtractionAttached` | `rawExtraction` | `GeminiExtractor` returned | (no status change; populates `geminiExtraction`) |
| `ExtractionReconciled` | `mergedExtraction` | Reconciler completed; both extractors (or one, degraded) returned | → RECONCILED, `finishedAt = now` |
| `ExtractionDegraded` | `mergedExtraction(partial), failureReason` | An extractor timed out | → DEGRADED, `finishedAt = now` |
| `AgreementScored` | `score, rationale, highDisagreementFields` | `AgreementSampler` → `EvalJudge` | (no status change; populates `agreementScore` + `agreementRationale`) |

## Events (`DocumentQueue`)

| Event | Payload |
|---|---|
| `DocumentReceived` | `extractionId, filename, extractedText, submittedBy, submittedAt` |

The `extractedText` carried on `DocumentReceived` is the raw text; it is handed to the workflow's start command and redacted in `sanitizeStep` before it touches `ExtractionEntity`. The queue event is the only place raw text is held, and it exists for replay/audit; deployers who must avoid even that retention can drop the `extractedText` from the queue event and re-fetch from source.

## View row

`ExtractionRow` mirrors `Extraction` minus the heavy `redactedText` (the row keeps `redactionCount`, the field counts, the merged field list summary, and the agreement score; it truncates `claudeExtraction.fields` and `geminiExtraction.fields` to counts for the list view). The UI fetches the full extraction by id on expand.
