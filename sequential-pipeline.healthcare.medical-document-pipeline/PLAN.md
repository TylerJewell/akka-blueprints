# PLAN — medical-document-pipeline

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[DocumentEndpoint]:::ep
  Entity[DocumentEntity]:::ese
  WF[DocumentPipelineWorkflow]:::wf
  Agent[MedicalDocumentAgent]:::agent
  Extract[ExtractTools]:::tool
  Validate[ValidateTools]:::tool
  Summarize[SummarizeTools]:::tool
  Guard[SanitizationGuardrail]:::guard
  Scorer[ClinicalAccuracyScorer]:::guard
  View[DocumentView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  API -->|start| WF
  WF -->|sanitizeStep PHI/PII mask| Entity
  WF -->|extractStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Extract
  Agent -->|invokes| Validate
  Agent -->|invokes| Summarize
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ExtractedFields / ValidationResult / ClinicalSummary| WF
  WF -->|reviewGateStep pause| Entity
  API -->|POST review| Entity
  Entity -->|ReviewSubmitted| WF
  WF -->|recordSummary| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant C as Clinician (UI)
  participant API as DocumentEndpoint
  participant E as DocumentEntity
  participant W as DocumentPipelineWorkflow
  participant A as MedicalDocumentAgent
  participant G as SanitizationGuardrail
  participant T as Tools (Extract/Validate/Summarize)
  participant Sc as ClinicalAccuracyScorer

  C->>API: POST /api/documents { documentType, rawText }
  API->>E: receive(documentType, rawText)
  E-->>API: { documentId }
  API->>W: start(documentId)
  W->>E: startSanitize
  W->>E: recordSanitized(maskedDocument)
  W->>A: runSingleTask(EXTRACT_FIELDS, maskedText)
  A->>G: before-tool-call(extractDiagnoses, EXTRACT)
  G-->>A: accept (sanitized=true, status EXTRACTING)
  A->>T: extractDemographics + extractDiagnoses + extractMedications + extractVitals
  T-->>A: List<ClinicalField> per group
  A-->>W: ExtractedFields
  W->>E: recordExtractedFields
  W->>E: requestReview
  E-.->>C: SSE event(AWAITING_REVIEW)
  C->>API: POST /api/documents/{id}/review { approvedFields, reviewedBy }
  API->>E: submitReview(validationResult)
  E-->>W: ReviewSubmitted (workflow resumes)
  W->>A: runSingleTask(WRITE_SUMMARY, validationResult)
  A->>G: before-tool-call(buildPatientContext, SUMMARIZE)
  G-->>A: accept (status SUMMARIZING, validationResult present)
  A->>T: buildPatientContext + buildClinicalImpression + buildRecommendations
  T-->>A: ClinicalSummary sections
  A-->>W: ClinicalSummary
  W->>E: recordSummary
  W->>Sc: score(clinicalSummary, validationResult, extractedFields)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>C: SSE event(EVALUATED)
```

## State machine — `DocumentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZING: SanitizationStarted
  SANITIZING --> SANITIZED: DocumentSanitized
  SANITIZED --> EXTRACTING: ExtractionStarted
  EXTRACTING --> EXTRACTED: FieldsExtracted
  EXTRACTED --> AWAITING_REVIEW: ReviewRequested
  AWAITING_REVIEW --> REVIEW_SUBMITTED: ReviewSubmitted
  REVIEW_SUBMITTED --> SUMMARIZING: SummarizationStarted
  SUMMARIZING --> SUMMARIZED: SummaryWritten
  SUMMARIZED --> EVALUATED: EvaluationScored
  SANITIZING --> FAILED: DocumentFailed
  EXTRACTING --> FAILED: DocumentFailed
  AWAITING_REVIEW --> FAILED: DocumentFailed(review-timeout)
  SUMMARIZING --> FAILED: DocumentFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget, a step timeout, or a review timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  DocumentEntity ||--o{ DocumentReceived : emits
  DocumentEntity ||--o{ SanitizationStarted : emits
  DocumentEntity ||--o{ DocumentSanitized : emits
  DocumentEntity ||--o{ ExtractionStarted : emits
  DocumentEntity ||--o{ FieldsExtracted : emits
  DocumentEntity ||--o{ ReviewRequested : emits
  DocumentEntity ||--o{ ReviewSubmitted : emits
  DocumentEntity ||--o{ SummarizationStarted : emits
  DocumentEntity ||--o{ SummaryWritten : emits
  DocumentEntity ||--o{ EvaluationScored : emits
  DocumentEntity ||--o{ GuardrailRejected : emits
  DocumentEntity ||--o{ DocumentFailed : emits
  DocumentView }o--|| DocumentEntity : projects
  DocumentPipelineWorkflow }o--|| DocumentEntity : reads-and-writes
  MedicalDocumentAgent ||--o{ ExtractedFields : returns
  MedicalDocumentAgent ||--o{ ValidationResult : returns
  MedicalDocumentAgent ||--o{ ClinicalSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DocumentEndpoint` | `api/DocumentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/DocumentRecord.java`, events in `domain/DocumentEvent.java`) |
| `DocumentPipelineWorkflow` | `application/DocumentPipelineWorkflow.java` |
| `MedicalDocumentAgent` | `application/MedicalDocumentAgent.java` (tasks in `application/DocumentTasks.java`) |
| `ExtractTools` | `application/ExtractTools.java` |
| `ValidateTools` | `application/ValidateTools.java` |
| `SummarizeTools` | `application/SummarizeTools.java` |
| `SanitizationGuardrail` | `application/SanitizationGuardrail.java` |
| `ClinicalAccuracyScorer` | `application/ClinicalAccuracyScorer.java` |
| `DocumentView` | `application/DocumentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `sanitizeStep` 10 s, `extractStep` 60 s, `reviewGateStep` 48 h, `summarizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DocumentPipelineWorkflow::error)`.
- **Idempotency**: each workflow uses `"pipeline-" + documentId` as the workflow id; restart of the same documentId is rejected by the workflow runtime. The agent instance id is `"agent-" + documentId` so each document has its own per-task conversation memory.
- **Review gate**: `reviewGateStep` parks the workflow using the Akka Workflow pause/resume pattern. The external `POST /api/documents/{id}/review` call triggers `DocumentEntity.submitReview(...)`, which emits `ReviewSubmitted` and resumes the workflow. The 48 h step timeout converts to a `DocumentFailed` event with `reason = "review-timeout"` if no decision arrives.
- **One agent per document**: `MedicalDocumentAgent` runs two active LLM tasks per document — EXTRACT and WRITE_SUMMARY. VALIDATE is the clinician's review step, not an LLM call, so the agent's three-task pattern is EXTRACT → [human review] → WRITE_SUMMARY. `DocumentTasks.VALIDATE_FIELDS` is declared for type-safety and prompt coherence but its LLM invocation is replaced by the human review gate in the workflow.
- **Sanitization is synchronous and deterministic**: `sanitizeStep` runs rule-based masking in-process. No LLM call, no external service. The same document text always produces the same masked output.
- **Eval is synchronous and deterministic**: `ClinicalAccuracyScorer` runs in-process inside `evalStep`. No LLM call — the same summary always scores the same. This maintains the single-agent invariant.
- **No saga / no compensation**: every step is either deterministic in-process work, an append-only event write, a single-task agent call, or a human decision gate. A failed document stays at the last successful event; the UI shows the partial state.
