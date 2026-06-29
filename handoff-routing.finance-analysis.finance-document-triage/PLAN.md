# PLAN — finance-document-triage

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'fontFamily':'Instrument Sans, system-ui, sans-serif'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef autonomous fill:#0e2a1e,stroke:#3fb950,color:#3fb950;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Sim[DocumentSimulator]:::ta
  Queue[DocumentQueue]:::ese
  PII[PiiSanitizer]:::cons
  Sector[SectorSanitizer]:::cons
  Clf[ClassifierAgent]:::agent
  Inv[InvoiceProcessor]:::autonomous
  Loan[LoanApplicationProcessor]:::autonomous
  Comp[ComplianceReviewProcessor]:::autonomous
  Judge[ClassificationJudge]:::agent
  Guard[OutputGuardrail]:::agent
  WF[DocumentWorkflow]:::wf
  Entity[DocumentEntity]:::ese
  View[DocumentView]:::view
  Scorer[ClassificationEvalScorer]:::cons
  API[DocumentEndpoint]:::ep
  App[AppEndpoint]:::ep

  Sim -.->|every 30s| Queue
  Queue -.->|subscribes| PII
  PII -->|registerIncoming + attachPiiSanitized| Entity
  Entity -.->|DocumentPiiSanitized| Sector
  Sector -->|attachSectorSanitized| Entity
  Sector -->|start workflow| WF
  WF -->|classify| Clf
  WF -->|PROCESS task| Inv
  WF -->|PROCESS task| Loan
  WF -->|PROCESS task| Comp
  WF -->|check draft| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordClassificationScore| Entity
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  App -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (invoice happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Sim as DocumentSimulator
  participant Q as DocumentQueue
  participant P as PiiSanitizer
  participant Sx as SectorSanitizer
  participant E as DocumentEntity
  participant W as DocumentWorkflow
  participant C as ClassifierAgent
  participant I as InvoiceProcessor
  participant G as OutputGuardrail
  participant Sc as ClassificationEvalScorer
  participant J as ClassificationJudge

  Sim->>Q: receive(IncomingDocument)
  Q->>P: InboundDocumentReceived
  P->>E: registerIncoming + attachPiiSanitized
  E->>Sx: DocumentPiiSanitized event
  Sx->>E: attachSectorSanitized
  Sx->>W: start(documentId, sanitized)
  W->>C: classify(sanitized)
  C-->>W: ClassificationDecision{INVOICE}
  W->>E: recordClassification(decision) [emits DocumentClassified]
  E->>Sc: DocumentClassified event
  Sc->>J: score(sanitized, decision)
  J-->>Sc: ClassificationScore
  Sc->>E: recordClassificationScore [emits ClassificationScored]
  W->>E: recordRouting(INVOICE) [emits DocumentRouted]
  W->>I: runSingleTask(PROCESS, prompt)
  I-->>W: ProcessingResult
  W->>E: recordDraft(result) [emits ResultDrafted]
  W->>G: check(sanitized, result)
  G-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(result) [emits ResultPublished, status PROCESSED]
```

The eval-event sequence (steps 9–12) runs concurrently with the workflow's continuation — `ClassificationEvalScorer` is a Consumer reading the entity's event stream, independent of `DocumentWorkflow`. Both writes target the same `DocumentEntity`; the entity's commands are idempotent on `documentId`.

## State machine — `DocumentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> PII_SANITIZED: DocumentPiiSanitized
  PII_SANITIZED --> SECTOR_SANITIZED: DocumentSectorSanitized
  SECTOR_SANITIZED --> CLASSIFIED: DocumentClassified
  CLASSIFIED --> ROUTED_INVOICE: docType = INVOICE
  CLASSIFIED --> ROUTED_LOAN: docType = LOAN_APPLICATION
  CLASSIFIED --> ROUTED_COMPLIANCE: docType = COMPLIANCE_REVIEW
  ROUTED_INVOICE --> RESULT_DRAFTED: ResultDrafted
  ROUTED_LOAN --> RESULT_DRAFTED: ResultDrafted
  ROUTED_COMPLIANCE --> RESULT_DRAFTED: ResultDrafted
  RESULT_DRAFTED --> PROCESSED: guardrail.allowed
  RESULT_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> PROCESSED: operator unblock
  PROCESSED --> [*]
  ESCALATED --> [*]
```

The `ClassificationScored` event does not change `status`; it attaches the eval result. The state machine omits it for clarity.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  DocumentEntity ||--o{ DocumentReceived : emits
  DocumentEntity ||--o{ DocumentPiiSanitized : emits
  DocumentEntity ||--o{ DocumentSectorSanitized : emits
  DocumentEntity ||--o{ DocumentClassified : emits
  DocumentEntity ||--o{ DocumentRouted : emits
  DocumentEntity ||--o{ ResultDrafted : emits
  DocumentEntity ||--o{ GuardrailVerdictAttached : emits
  DocumentEntity ||--o{ ResultPublished : emits
  DocumentEntity ||--o{ ResultBlocked : emits
  DocumentEntity ||--o{ DocumentEscalated : emits
  DocumentEntity ||--o{ ClassificationScored : emits
  DocumentView }o--|| DocumentEntity : projects
  DocumentQueue ||--o{ InboundDocumentReceived : emits
  PiiSanitizer }o--|| DocumentQueue : subscribes
  SectorSanitizer }o--|| DocumentEntity : subscribes
  ClassificationEvalScorer }o--|| DocumentEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DocumentSimulator` | `application/DocumentSimulator.java` |
| `DocumentQueue` | `application/DocumentQueue.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `SectorSanitizer` | `application/SectorSanitizer.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `InvoiceProcessor` | `application/InvoiceProcessor.java` |
| `LoanApplicationProcessor` | `application/LoanApplicationProcessor.java` |
| `ComplianceReviewProcessor` | `application/ComplianceReviewProcessor.java` |
| `ClassificationJudge` | `application/ClassificationJudge.java` |
| `OutputGuardrail` | `application/OutputGuardrail.java` |
| `DocumentWorkflow` | `application/DocumentWorkflow.java` |
| `DocumentEntity` | `application/DocumentEntity.java` (state in `domain/Document.java`, events in `domain/DocumentEvent.java`) |
| `DocumentView` | `application/DocumentView.java` |
| `ClassificationEvalScorer` | `application/ClassificationEvalScorer.java` |
| `DocumentEndpoint` | `api/DocumentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/FinanceTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `invoiceStep` / `loanStep` / `complianceStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the document to `ESCALATED` with the failure reason captured.
- **Two-Consumer sanitization chain.** `PiiSanitizer` runs first (subscribed to `DocumentQueue`). `SectorSanitizer` runs second (subscribed to `DocumentEntity` on `DocumentPiiSanitized`). Only `SectorSanitizer` starts the `DocumentWorkflow`. This guarantees both redaction passes complete before any LLM call.
- **Idempotency.** Every per-document primitive is keyed by `documentId`: `DocumentEntity` id is `documentId`; `DocumentWorkflow` id is `documentId`; agent sessions for `ClassifierAgent`, `ClassificationJudge`, and `OutputGuardrail` use `documentId`. Duplicate sanitize events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `ClassificationEvalScorer` (Consumer) and `DocumentWorkflow` both append events to the same `DocumentEntity`. Order is not guaranteed but does not matter: `ClassificationScored` only mutates `classificationScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** Once the processor returns its `ProcessingResult`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback — a blocked result sits in `BLOCKED` until an operator unblocks via `POST /api/documents/{id}/unblock`.
- **Simulator throughput.** `DocumentSimulator` drips one document every 30 s; the system can comfortably process each document end-to-end inside that window with mock or real LLMs.
