# SPEC — medical-document-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Medical Document Processing Assistant.
**One-line pitch:** A clinician uploads a medical document; one `MedicalDocumentAgent` walks it through three task phases — **EXTRACT** structured clinical fields, **VALIDATE** them for plausibility and completeness, **SUMMARIZE** into a clinician-ready `ClinicalSummary` — with each phase gated on the prior phase's recorded output, PHI/PII sanitization enforced before any extract tool runs, and a clinician approval gate between extraction and summarization.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a healthcare document processing domain. One `MedicalDocumentAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the EXTRACT task's typed output becomes the VALIDATE task's instruction context; the VALIDATE task's typed output becomes the SUMMARIZE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Four governance mechanisms are wired around the pipeline:

- A **special-category sanitizer** ensures that health data fields in the raw document are masked before any extract tool is called. The sanitizer runs at document ingestion time and stamps a `sanitized: true` flag on the `DocumentEntity`. The `before-tool-call` guardrail checks this flag before allowing any EXTRACT-phase tool to proceed.
- A **PII sanitizer** runs alongside the special-category sanitizer. Patient identifiers — name, date of birth, medical record number, contact details — are replaced with tokenised placeholders before the document enters the pipeline.
- A **human-in-the-loop gate** at `validateStep`. After `FieldsExtracted` is recorded, the workflow pauses and presents the extracted fields to the reviewing clinician via the App UI. The clinician either approves (marks fields as accepted) or annotates corrections. The workflow resumes only after the clinician submits the validation decision. A configurable timeout (default 48 h) auto-escalates to a senior reviewer if no decision is received.
- An **`on-decision-eval`** runs immediately after `SummaryWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `ClinicalAccuracyScorer` (no LLM call) checks that every mandatory clinical field in the extracted set has a matching entry in the summary, that every clinical value in the summary is present in the validated `ValidationResult`, that no value is cited without a traceable source field, and that the word count of each summary section falls within the configured range.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the per-phase guardrails.

## 3. User-facing flows

The clinician opens the App UI tab.

1. The clinician selects a **document** from the upload panel (or picks one of three seeded documents — `discharge-summary-cardiology.txt`, `radiology-report-chest-ct.txt`, `referral-letter-orthopedics.txt`).
2. The clinician clicks **Process document**. The UI POSTs to `/api/documents` and receives a `documentId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `SANITIZING` — the service has started PHI/PII masking. Within ~2 s it reaches `SANITIZED`.
4. Within ~10–20 s the card reaches `EXTRACTING`, then `EXTRACTED` — the `ExtractedFields` record is visible in the card detail (a table of field names, extracted values, and confidence scores).
5. The card transitions to `AWAITING_REVIEW`. The clinician sees the extracted fields in the right pane and either clicks **Approve** or edits individual field values and clicks **Submit corrections**.
6. After approval the card transitions to `SUMMARIZING`, then `SUMMARIZED`, then within ~1 s to `EVALUATED`. The right pane now shows the full `ClinicalSummary` — patient context block (de-identified), chief complaint, key findings, clinical impression, recommendations — plus an eval score chip (1–5) and a one-line rationale.
7. The clinician can upload another document; the live list keeps all processed documents visible with their status.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DocumentEndpoint` | `HttpEndpoint` | `/api/documents/*` — submit, list, get, SSE, clinician-review decision; serves `/api/metadata/*`. | — | `DocumentEntity`, `DocumentView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DocumentEntity` | `EventSourcedEntity` | Per-document lifecycle: received → sanitizing → sanitized → extracting → extracted → awaiting-review → review-submitted → summarizing → summarized → evaluated. Source of truth for document state and all typed payloads. | `DocumentEndpoint`, `DocumentPipelineWorkflow` | `DocumentView` |
| `DocumentPipelineWorkflow` | `Workflow` | One workflow per document. Steps: `sanitizeStep` → `extractStep` → `reviewGateStep` → `summarizeStep` → `evalStep`. `sanitizeStep` runs deterministic PHI/PII masking; `extractStep` and `summarizeStep` call the agent; `reviewGateStep` pauses until the clinician submits a decision; `evalStep` runs the scorer. | started by `DocumentEndpoint` after `RECEIVED` | `MedicalDocumentAgent`, `DocumentEntity` |
| `MedicalDocumentAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `DocumentTasks.java`: `EXTRACT_FIELDS` → `ExtractedFields`, `VALIDATE_FIELDS` → `ValidationResult`, `WRITE_SUMMARY` → `ClinicalSummary`. Each task is registered with the phase-appropriate function tools. | invoked by `DocumentPipelineWorkflow` | returns typed results |
| `ExtractTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `extractDemographics(maskedText)`, `extractDiagnoses(maskedText)`, `extractMedications(maskedText)`, `extractVitals(maskedText)`. Reads masked document text from `DocumentEntity`. | called from EXTRACT task | returns typed field groups |
| `ValidateTools` | function-tools class | Implements `checkFieldCompleteness(fields)` and `checkValuePlausibility(fields)`. Pure in-memory rule checks against a bundled reference table (`src/main/resources/reference/clinical-ranges.json`). | called from VALIDATE task | returns `List<ValidationFinding>` |
| `SummarizeTools` | function-tools class | Implements `buildPatientContext(fields)`, `buildClinicalImpression(findings, fields)`, and `buildRecommendations(findings)`. | called from SUMMARIZE task | returns `ClinicalSummary` sections |
| `SanitizationGuardrail` | `before-tool-call` guardrail (registered on `MedicalDocumentAgent`) | Checks that the `DocumentEntity.sanitized` flag is `true` before any EXTRACT-phase tool runs. Also enforces phase ordering: VALIDATE tools require `status ∈ {EXTRACTED, AWAITING_REVIEW, REVIEW_SUBMITTED}`; SUMMARIZE tools require `status ∈ {REVIEW_SUBMITTED, SUMMARIZING}` AND `validationResult.isPresent()`. On reject, returns a structured error and records `GuardrailRejected` on the entity. | every tool call on every task | accept / structured-reject |
| `ClinicalAccuracyScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `ClinicalSummary`, `ValidationResult`, `ExtractedFields`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `DocumentView` | `View` | Read model: one row per document for the UI. | `DocumentEntity` events | `DocumentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MaskedDocument(
    String documentId,
    String documentType,     // "discharge-summary" | "radiology-report" | "referral-letter"
    String maskedText,       // PHI/PII replaced with tokens like [PATIENT_NAME], [DOB], [MRN]
    List<PiiToken> tokens,   // mapping from token to original value (stored separately, access-controlled)
    Instant maskedAt
) {}

record PiiToken(String token, String category) {} // category: PATIENT_NAME | DOB | MRN | ADDRESS | PHONE

record ClinicalField(
    String fieldId,
    String fieldName,        // e.g. "primary_diagnosis", "heart_rate", "medication_name"
    String extractedValue,
    double confidence,       // 0.0–1.0
    String sourceSpan        // character range in maskedText this value was drawn from
) {}

record ExtractedFields(
    List<ClinicalField> demographics,
    List<ClinicalField> diagnoses,
    List<ClinicalField> medications,
    List<ClinicalField> vitals,
    Instant extractedAt
) {}

record ValidationFinding(
    String findingId,
    String fieldId,
    FindingSeverity severity,  // OK | WARNING | ERROR
    String message
) {}

enum FindingSeverity { OK, WARNING, ERROR }

record ValidationResult(
    List<ClinicalField> approvedFields,   // as accepted/corrected by clinician
    List<ValidationFinding> findings,
    String reviewedBy,                    // clinician identifier (de-identified token)
    Instant reviewedAt
) {}

record ClinicalSection(
    String sectionId,
    String heading,
    String body,
    List<String> sourceFieldIds  // MUST reference fieldIds from approvedFields
) {}

record ClinicalSummary(
    String documentType,
    String patientContext,       // de-identified demographics block
    String chiefComplaint,
    List<ClinicalSection> sections,
    String clinicalImpression,
    String recommendations,
    Instant writtenAt
) {}

record EvalResult(
    int score,             // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record DocumentRecord(
    String documentId,
    String documentType,
    Optional<MaskedDocument> maskedDocument,
    Optional<ExtractedFields> extractedFields,
    Optional<ValidationResult> validationResult,
    Optional<ClinicalSummary> clinicalSummary,
    Optional<EvalResult> eval,
    DocumentStatus status,
    boolean sanitized,
    Instant receivedAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum DocumentStatus {
    RECEIVED, SANITIZING, SANITIZED, EXTRACTING, EXTRACTED,
    AWAITING_REVIEW, REVIEW_SUBMITTED, SUMMARIZING, SUMMARIZED,
    EVALUATED, FAILED
}
```

Events on `DocumentEntity`: `DocumentReceived`, `SanitizationStarted`, `DocumentSanitized`, `ExtractionStarted`, `FieldsExtracted`, `ReviewRequested`, `ReviewSubmitted`, `SummarizationStarted`, `SummaryWritten`, `EvaluationScored`, `GuardrailRejected`, `DocumentFailed`.

Every nullable lifecycle field on the `DocumentRecord` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/documents` — body `{ documentType, rawText }` → `{ documentId }`.
- `GET /api/documents` — list all documents, newest-first.
- `GET /api/documents/{id}` — one document record.
- `GET /api/documents/sse` — Server-Sent Events; one event per state transition.
- `POST /api/documents/{id}/review` — body `{ approvedFields, corrections, reviewedBy }` → `204`. Clinician submits their validation decision.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Medical Document Processing Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of documents (status pill + document type + age) and a right pane with the selected document's detail — masked text preview, extracted fields table, validation findings, clinician review form (when in `AWAITING_REVIEW`), clinical summary sections, eval score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — special-category sanitizer (PHI)**: runs at document ingestion in `sanitizeStep`. Health data fields — diagnoses, medications, vitals, procedure notes — are identified using a bundled regex/rule set (`src/main/resources/reference/phi-patterns.json`) and masked before any LLM call. `DocumentEntity` records `DocumentSanitized{maskedDocument}` which sets `sanitized = true`. The `SanitizationGuardrail` checks this flag on every EXTRACT-phase tool call; if `sanitized` is `false` the tool is rejected with `"phi-not-sanitized"` regardless of phase.
- **S2 — PII sanitizer (patient identifiers)**: runs in the same `sanitizeStep`. Patient name, date of birth, MRN, address, and phone number are replaced with stable tokens (`[PATIENT_NAME]`, `[DOB]`, `[MRN]`, `[ADDRESS]`, `[PHONE]`). Tokens are stored in `DocumentEntity.maskedDocument.tokens` and are not passed to the agent.
- **H1 — human-in-the-loop (clinician approval)**: after `FieldsExtracted`, `reviewGateStep` transitions the document to `AWAITING_REVIEW` and parks the workflow. The clinician sees the extracted fields in the App UI, can edit individual field values, and submits a decision via `POST /api/documents/{id}/review`. The workflow resumes on `ReviewSubmitted`. If no decision arrives within the configured timeout, the workflow escalates to a `FAILED` state with `reason = "review-timeout"`.
- **E1 — on-decision eval (clinical accuracy)**: runs immediately after `SummaryWritten` lands, as `evalStep` inside the workflow. `ClinicalAccuracyScorer` is deterministic and rule-based (no LLM call). Four checks, one point per check satisfied on a base of 1: (1) mandatory field coverage — every field in `ExtractedFields.diagnoses` has a matching `sourceFieldId` in a summary section; (2) value provenance — every `ClinicalSection.sourceFieldIds[i]` references a `fieldId` in `ValidationResult.approvedFields`; (3) section completeness — `patientContext`, `chiefComplaint`, `clinicalImpression`, and `recommendations` are all non-empty strings; (4) section word count — no summary section body exceeds 150 words or is shorter than 10 words. Emits `EvaluationScored{score:1..5, rationale}`. Score ≤ 2 flags the card for clinician follow-up.

## 9. Agent prompts

- `MedicalDocumentAgent` → `prompts/medical-document-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, never invent clinical values, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Clinician uploads `discharge-summary-cardiology.txt`; within 60 s (including review) the document reaches `EVALUATED` with non-empty extracted fields across all four field groups, a validation result with no `ERROR`-severity findings, ≥ 2 summary sections, and an eval score chip showing ≥ 4/5.
2. **J2** — The agent's first iteration on an extraction calls a SUMMARIZE-phase tool (`buildClinicalImpression`) before `ReviewSubmitted` has been recorded (mock LLM path). `SanitizationGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the document eventually completes correctly.
3. **J3** — A document whose mock-LLM trajectory produces a summary section citing a `sourceFieldId` not present in `ValidationResult.approvedFields` is scored 1 with a rationale naming the untraced field; the UI flags the card.
4. **J4** — A document submitted with `sanitized = false` (simulated by the mock) attempts an EXTRACT-phase tool call; the guardrail rejects it with `"phi-not-sanitized"` before the tool body runs; the rejection is visible in the audit log.
5. **J5** — The clinician submits corrections to two extracted field values during review; the workflow resumes with the corrected values in `ValidationResult.approvedFields`; the summary uses the corrected values and the eval confirms source provenance.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named medical-document-pipeline demonstrating the sequential-pipeline x
healthcare cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-healthcare-medical-document-pipeline.
Java package io.akka.samples.medicaldocumentprocessingassistant. Akka 3.6.0.
HTTP port 9837.

Components to wire (exactly):

- 1 AutonomousAgent MedicalDocumentAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/medical-document-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — ALL three tool sets (ExtractTools, ValidateTools, SummarizeTools) are
  registered on the agent; phase gating is the job of SanitizationGuardrail. The
  before-tool-call guardrail (SanitizationGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 4-iteration budget.

- 1 Workflow DocumentPipelineWorkflow per documentId with five steps:
  * sanitizeStep — applies PHI/PII masking using bundled patterns from
    src/main/resources/reference/phi-patterns.json; emits DocumentSanitized{maskedDocument}
    on the entity; sets sanitized = true. stepTimeout 10s.
  * extractStep — emits ExtractionStarted on the entity, then calls componentClient
    .forAutonomousAgent(MedicalDocumentAgent.class, "agent-" + documentId).runSingleTask(
      TaskDef.instructions("DocumentType: " + documentType + "\nMaskedText: " + maskedText
        + "\nPhase: EXTRACT\nExtract all demographics, diagnoses, medications, and vitals.")
        .metadata("documentId", documentId)
        .metadata("phase", "EXTRACT")
        .taskType(DocumentTasks.EXTRACT_FIELDS)
    ). Reads result to get ExtractedFields. Writes DocumentEntity.recordExtractedFields(
    fields). stepTimeout 60s.
  * reviewGateStep — emits ReviewRequested on the entity, setting status to
    AWAITING_REVIEW. The step PAUSES and waits for an external signal
    (POST /api/documents/{id}/review) that writes ReviewSubmitted{validationResult} onto
    the entity, which wakes the step via a workflow resume callback. stepTimeout 48h. On
    timeout, transitions to FAILED with reason "review-timeout".
  * summarizeStep — emits SummarizationStarted, then runSingleTask with
    TaskDef.instructions(formatSummarizeContext(validationResult, extractedFields,
    documentType)) and metadata.phase = "SUMMARIZE", taskType WRITE_SUMMARY. Writes
    DocumentEntity.recordSummary(summary). stepTimeout 60s.
  * evalStep — runs the deterministic ClinicalAccuracyScorer over (clinicalSummary,
    validationResult, extractedFields) and writes DocumentEntity.recordEvaluation(eval).
    stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(DocumentPipelineWorkflow::error). The error step writes
  DocumentFailed and ends.

- 1 EventSourcedEntity DocumentEntity (one per documentId). State DocumentRecord{
  documentId, documentType: String, maskedDocument: Optional<MaskedDocument>,
  extractedFields: Optional<ExtractedFields>, validationResult: Optional<ValidationResult>,
  clinicalSummary: Optional<ClinicalSummary>, eval: Optional<EvalResult>,
  status: DocumentStatus, sanitized: boolean, receivedAt: Instant,
  finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  DocumentStatus enum: RECEIVED, SANITIZING, SANITIZED, EXTRACTING, EXTRACTED,
  AWAITING_REVIEW, REVIEW_SUBMITTED, SUMMARIZING, SUMMARIZED, EVALUATED, FAILED.
  Events: DocumentReceived{documentType, rawText}, SanitizationStarted,
  DocumentSanitized{maskedDocument}, ExtractionStarted, FieldsExtracted{extractedFields},
  ReviewRequested, ReviewSubmitted{validationResult}, SummarizationStarted,
  SummaryWritten{clinicalSummary}, EvaluationScored{eval},
  GuardrailRejected{phase, tool, reason, rejectedAt}, DocumentFailed{reason}.
  Commands: receive, startSanitize, recordSanitized, startExtract, recordExtractedFields,
  requestReview, submitReview, startSummarize, recordSummary, recordEvaluation,
  recordGuardrailRejection, fail, getDocument. emptyState() returns
  DocumentRecord.initial("") with all Optional fields as Optional.empty(),
  guardrailRejections = List.of(), sanitized = false, and no commandContext() reference
  (Lesson 3).

- 1 View DocumentView with row type DocumentRow that mirrors DocumentRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes DocumentEntity events. ONE
  query getAllDocuments: SELECT * AS documents FROM document_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * DocumentEndpoint at /api with POST /documents (body {documentType, rawText}; mints
    documentId; calls DocumentEntity.receive(documentType, rawText); starts
    DocumentPipelineWorkflow with id "pipeline-" + documentId; returns {documentId}),
    GET /documents (list from getAllDocuments, sorted newest-first), GET /documents/{id}
    (one row), GET /documents/sse (Server-Sent Events from the view's stream-updates),
    POST /documents/{id}/review (clinician submits validation decision; calls
    DocumentEntity.submitReview(validationResult); wakes the workflow), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- DocumentTasks.java declaring three Task<R> constants:
    EXTRACT_FIELDS = Task.name("Extract fields").description("Extract structured clinical
      fields from the masked document text by calling extractDemographics, extractDiagnoses,
      extractMedications, and extractVitals").resultConformsTo(ExtractedFields.class);
    VALIDATE_FIELDS = Task.name("Validate fields").description("Check extracted fields for
      completeness and value plausibility using checkFieldCompleteness and
      checkValuePlausibility").resultConformsTo(ValidationResult.class);
    WRITE_SUMMARY = Task.name("Write summary").description("Compose a ClinicalSummary whose
      sections are traceable to approved fields").resultConformsTo(ClinicalSummary.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- DocumentPhase.java — enum {EXTRACT, VALIDATE, SUMMARIZE}. Each function-tool method
  carries the constant phase.

- ExtractTools.java — @FunctionTool extractDemographics(String maskedText) ->
  List<ClinicalField>; @FunctionTool extractDiagnoses(String maskedText) ->
  List<ClinicalField>; @FunctionTool extractMedications(String maskedText) ->
  List<ClinicalField>; @FunctionTool extractVitals(String maskedText) ->
  List<ClinicalField>. Each reads maskedText and applies regex + keyword rules from
  src/main/resources/reference/extraction-patterns.json.

- ValidateTools.java — @FunctionTool checkFieldCompleteness(ExtractedFields) ->
  List<ValidationFinding> (checks mandatory fields are non-empty; one finding per gap);
  @FunctionTool checkValuePlausibility(ExtractedFields) -> List<ValidationFinding>
  (checks vitals and lab values against ranges in clinical-ranges.json; one finding per
  out-of-range value).

- SummarizeTools.java — @FunctionTool buildPatientContext(ValidationResult) -> String
  (assembles de-identified demographic block); @FunctionTool buildClinicalImpression(
  List<ValidationFinding>, ExtractedFields) -> String; @FunctionTool
  buildRecommendations(List<ValidationFinding>) -> String.

- SanitizationGuardrail.java — implements the before-tool-call hook. On EXTRACT-phase
  tools: checks DocumentEntity.sanitized is true; rejects with "phi-not-sanitized" if
  false. On all phases: checks phase ordering using the accept matrix from Section 8.
  On reject ALSO calls DocumentEntity.recordGuardrailRejection(phase, tool, reason) so the
  rejection is visible in the UI rejection-log strip and in the audit log.

- ClinicalAccuracyScorer.java — pure deterministic logic (no LLM). Inputs: ClinicalSummary,
  ValidationResult, ExtractedFields. Outputs: EvalResult with score and rationale. Four
  checks: (1) mandatory field coverage, (2) value provenance, (3) section completeness,
  (4) section word count. Score range 1–5. Rationale names the largest gap.

- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9837 and the three model-provider blocks (anthropic
  claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash) reading the canonical
  env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/documents/ — three sample document files (one per seeded
  type: discharge-summary-cardiology.txt, radiology-report-chest-ct.txt,
  referral-letter-orthopedics.txt) with realistic but fully synthetic content — no real
  patient data.

- src/main/resources/reference/phi-patterns.json — regex patterns for PHI detection
  covering patient name, DOB, MRN, address, and phone number formats.

- src/main/resources/reference/extraction-patterns.json — keyword and regex rules for
  extracting clinical fields per document type.

- src/main/resources/reference/clinical-ranges.json — reference ranges for common vitals
  (HR 60–100 bpm, SBP 90–140 mmHg, DBP 60–90 mmHg, SpO2 ≥ 95%, Temp 36.1–37.2 °C).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root with 4 controls (S1, S2, H1, E1) matching the
  mechanisms in Section 8.

- risk-survey.yaml at the project root pre-filled for the healthcare domain.

- prompts/medical-document-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Medical Document Processing
  Assistant", prerequisites, generate-the-system, what-you-get, customise-before-
  generating, what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of document cards; right = selected-document detail with masked-text
  preview, extracted fields table, validation findings, clinician review form when
  AWAITING_REVIEW, clinical summary sections, eval-score chip, rejection-log strip).
  Browser title exactly:
  <title>Akka Sample: Medical Document Processing Assistant</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider with per-task dispatch.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch. Each branch reads src/main/resources/mock-responses/<task-id>.json, picks one
  entry pseudo-randomly per call (seedFor(documentId)), and deserialises into the task's
  typed return.
- Per-task mock-response shapes:
    extract-fields.json — 5 ExtractedFields entries, one per seeded document type plus two
      extras, each with populated demographics, diagnoses, medications, and vitals lists.
      Plus 1 deliberately phase-violating entry whose tool_calls starts with
      buildClinicalImpression (a SUMMARIZE-phase tool called during EXTRACT) — the guardrail
      rejects it. Violation selected on the first iteration of every third document
      (modulo seed) so J2 is reproducible.
    validate-fields.json — 5 ValidationResult entries, each with approvedFields and a mix
      of OK/WARNING findings. Tool calls: checkFieldCompleteness + checkValuePlausibility.
    write-summary.json — 5 ClinicalSummary entries. Each cites sourceFieldIds present in
      the paired ValidationResult.approvedFields. Plus 1 hallucinated-source entry whose
      first section cites a fieldId absent from approvedFields — evalStep scores it 1; J3
      verifies this.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: MedicalDocumentAgent extends akka.javasdk.agent.autonomous.AutonomousAgent.
  DocumentTasks.java MUST exist.
- Lesson 4: explicit stepTimeout on every workflow step (sanitizeStep 10s, extractStep 60s,
  reviewGateStep 48h, summarizeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on DocumentRecord is Optional<T>.
- Lesson 7: DocumentTasks.java with EXTRACT_FIELDS, VALIDATE_FIELDS, WRITE_SUMMARY is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9837 declared in akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata user-facing.
- Lesson 12: UI fits 1080px content column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: mermaid CSS overrides and themeVariables block included.
- Lesson 25: API-key sourcing — never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute only. Exactly five
  <section class="tab-panel"> elements.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
