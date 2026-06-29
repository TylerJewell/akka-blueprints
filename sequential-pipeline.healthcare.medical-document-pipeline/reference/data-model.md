# Data model — medical-document-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PiiToken` | `token` | `String` | no | Token string, e.g. `[PATIENT_NAME]`. |
| | `category` | `String` | no | `PATIENT_NAME` / `DOB` / `MRN` / `ADDRESS` / `PHONE`. |
| `MaskedDocument` | `documentId` | `String` | no | — |
| | `documentType` | `String` | no | `discharge-summary` / `radiology-report` / `referral-letter`. |
| | `maskedText` | `String` | no | Document text with PHI/PII replaced by tokens. |
| | `tokens` | `List<PiiToken>` | no | Token-to-category mapping. Original values NOT stored here. |
| | `maskedAt` | `Instant` | no | When `sanitizeStep` completed. |
| `ClinicalField` | `fieldId` | `String` | no | Stable short id (`f-<8 hex>`). |
| | `fieldName` | `String` | no | e.g. `primary_diagnosis`, `heart_rate`, `medication_name`. |
| | `extractedValue` | `String` | no | Value extracted from masked text. |
| | `confidence` | `double` | no | 0.0–1.0. |
| | `sourceSpan` | `String` | no | Character range in `maskedText` (e.g. `"88:120"`). |
| `ExtractedFields` | `demographics` | `List<ClinicalField>` | no | Possibly empty. |
| | `diagnoses` | `List<ClinicalField>` | no | Possibly empty; E1 rule 1 checks this list. |
| | `medications` | `List<ClinicalField>` | no | Possibly empty. |
| | `vitals` | `List<ClinicalField>` | no | Possibly empty. |
| | `extractedAt` | `Instant` | no | When EXTRACT task returned. |
| `ValidationFinding` | `findingId` | `String` | no | Short stable id. |
| | `fieldId` | `String` | no | References a `ClinicalField.fieldId` in `ExtractedFields`. |
| | `severity` | `FindingSeverity` | no | `OK` / `WARNING` / `ERROR`. |
| | `message` | `String` | no | Human-readable finding description. |
| `ValidationResult` | `approvedFields` | `List<ClinicalField>` | no | As accepted/corrected by the clinician. E1 rule 2 checks against this list. |
| | `findings` | `List<ValidationFinding>` | no | From tool checks; may be empty. |
| | `reviewedBy` | `String` | no | Clinician identifier token (not a real name). |
| | `reviewedAt` | `Instant` | no | When the clinician submitted the review decision. |
| `ClinicalSection` | `sectionId` | `String` | no | Short stable id. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 10–150 words (E1 rule 4). |
| | `sourceFieldIds` | `List<String>` | no | Non-empty (E1 rule 2). Each entry MUST reference a `fieldId` in `ValidationResult.approvedFields`. |
| `ClinicalSummary` | `documentType` | `String` | no | Matches the source document type. |
| | `patientContext` | `String` | no | Non-empty de-identified demographics block (E1 rule 3). |
| | `chiefComplaint` | `String` | no | Non-empty (E1 rule 3). |
| | `sections` | `List<ClinicalSection>` | no | One per primary diagnosis group; possibly empty if no diagnoses extracted. |
| | `clinicalImpression` | `String` | no | Non-empty 1–3 sentences (E1 rule 3). |
| | `recommendations` | `String` | no | Non-empty 1–3 sentences (E1 rule 3). |
| | `writtenAt` | `Instant` | no | When WRITE_SUMMARY task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `ClinicalAccuracyScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `EXTRACT` / `VALIDATE` / `SUMMARIZE`. |
| | `tool` | `String` | no | Name of the rejected tool. |
| | `reason` | `String` | no | Structured reason, e.g. `"phase-violation: ..."` or `"phi-not-sanitized"`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `DocumentRecord` (entity state) | `documentId` | `String` | no | — |
| | `documentType` | `String` | no | — |
| | `maskedDocument` | `Optional<MaskedDocument>` | yes | Populated after `DocumentSanitized`. |
| | `extractedFields` | `Optional<ExtractedFields>` | yes | Populated after `FieldsExtracted`. |
| | `validationResult` | `Optional<ValidationResult>` | yes | Populated after `ReviewSubmitted`. |
| | `clinicalSummary` | `Optional<ClinicalSummary>` | yes | Populated after `SummaryWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `DocumentStatus` | no | See enum. |
| | `sanitized` | `boolean` | no | `true` after `DocumentSanitized` lands. Checked by guardrail on every EXTRACT-phase tool call. |
| | `receivedAt` | `Instant` | no | When `DocumentReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `DocumentRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DocumentStatus`: `RECEIVED`, `SANITIZING`, `SANITIZED`, `EXTRACTING`, `EXTRACTED`, `AWAITING_REVIEW`, `REVIEW_SUBMITTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`.

`FindingSeverity` (used by `ValidateTools` and `ValidationFinding`): `OK`, `WARNING`, `ERROR`.

`DocumentPhase` (used by `@FunctionTool` annotations and `SanitizationGuardrail`): `EXTRACT`, `VALIDATE`, `SUMMARIZE`.

## Events (`DocumentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentReceived` | `documentType: String, rawText: String` | → RECEIVED |
| `SanitizationStarted` | — | → SANITIZING |
| `DocumentSanitized` | `maskedDocument: MaskedDocument` | → SANITIZED; sets `sanitized = true` |
| `ExtractionStarted` | — | → EXTRACTING |
| `FieldsExtracted` | `extractedFields: ExtractedFields` | → EXTRACTED |
| `ReviewRequested` | — | → AWAITING_REVIEW |
| `ReviewSubmitted` | `validationResult: ValidationResult` | → REVIEW_SUBMITTED |
| `SummarizationStarted` | — | → SUMMARIZING |
| `SummaryWritten` | `clinicalSummary: ClinicalSummary` | → SUMMARIZED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `DocumentFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `DocumentRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, `sanitized = false`, and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DocumentRow` mirrors `DocumentRecord` exactly. The UI fetches the full row via `GET /api/documents/{id}` and streams updates via `GET /api/documents/sse`.

The view declares ONE query: `getAllDocuments: SELECT * AS documents FROM document_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`DocumentTasks.java`)

```java
public final class DocumentTasks {
  public static final Task<ExtractedFields> EXTRACT_FIELDS = Task
      .name("Extract fields")
      .description("Extract structured clinical fields from the masked document text by calling "
          + "extractDemographics, extractDiagnoses, extractMedications, and extractVitals")
      .resultConformsTo(ExtractedFields.class);

  public static final Task<ValidationResult> VALIDATE_FIELDS = Task
      .name("Validate fields")
      .description("Check extracted fields for completeness and value plausibility using "
          + "checkFieldCompleteness and checkValuePlausibility")
      .resultConformsTo(ValidationResult.class);

  public static final Task<ClinicalSummary> WRITE_SUMMARY = Task
      .name("Write summary")
      .description("Compose a ClinicalSummary whose sections are traceable to approved fields")
      .resultConformsTo(ClinicalSummary.class);

  private DocumentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools and the guardrail accept matrix

Each `@FunctionTool` method on `ExtractTools`, `ValidateTools`, and `SummarizeTools` carries a `DocumentPhase` constant. `SanitizationGuardrail` reads this constant before the tool body runs and applies two checks:

1. **Sanitization check (EXTRACT phase only):** `DocumentEntity.sanitized` must be `true`. If `false`, reject with `"phi-not-sanitized"` regardless of status.
2. **Phase ordering check (all phases):**
   - `EXTRACT` tools: require `status ∈ {SANITIZED, EXTRACTING}`.
   - `VALIDATE` tools: require `status ∈ {EXTRACTED, AWAITING_REVIEW}`.
   - `SUMMARIZE` tools: require `status ∈ {REVIEW_SUBMITTED, SUMMARIZING}` AND `validationResult.isPresent()`.

On reject, `SanitizationGuardrail` returns a structured error to the agent loop and calls `DocumentEntity.recordGuardrailRejection(phase, tool, reason)` so the rejection appears in the UI's rejection-log strip and in the audit trail.
