# PLAN — invoice-processing

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
  classDef san fill:#0e1e2a,stroke:#60a5fa,color:#60a5fa;

  API[InvoiceEndpoint]:::ep
  Entity[InvoiceEntity]:::ese
  WF[InvoicePipelineWorkflow]:::wf
  Agent[InvoiceAgent]:::agent
  Extract[ExtractTools]:::tool
  Validate[ValidateTools]:::tool
  Post[PostTools]:::tool
  Guard[PhaseGuardrail]:::guard
  San[VendorPiiSanitizer]:::san
  Scorer[PostingQualityScorer]:::guard
  View[InvoiceView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  API -->|approve/deny| Entity
  WF -->|extractStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Extract
  Agent -->|invokes| Validate
  Agent -->|invokes| Post
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ExtractedInvoice| WF
  WF -->|sanitize| San
  San -->|SanitizationReport + sanitized ExtractedInvoice| WF
  WF -->|recordSanitization / recordExtraction| Entity
  WF -->|validateStep runSingleTask| Agent
  Agent -->|ValidatedInvoice| WF
  WF -->|recordValidation| Entity
  WF -->|approvalStep — pause if high-value| Entity
  WF -->|postStep runSingleTask| Agent
  Agent -->|PostedInvoice| WF
  WF -->|recordPosting| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, below-threshold invoice)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as InvoiceEndpoint
  participant E as InvoiceEntity
  participant W as InvoicePipelineWorkflow
  participant A as InvoiceAgent
  participant G as PhaseGuardrail
  participant T as Tools (Extract/Validate/Post)
  participant San as VendorPiiSanitizer
  participant Sc as PostingQualityScorer

  U->>API: POST /api/invoices { rawText }
  API->>E: create(rawText)
  E-->>API: { invoiceId }
  API->>W: start(invoiceId, rawText)
  W->>E: startExtraction
  W->>A: runSingleTask(EXTRACT_INVOICE, rawText)
  A->>G: before-tool-call(parseHeader, EXTRACT)
  G-->>A: accept (status EXTRACTING)
  A->>T: parseHeader + parseLineItems
  T-->>A: InvoiceHeader + List<LineItem>
  A-->>W: ExtractedInvoice
  W->>San: sanitize(ExtractedInvoice)
  San-->>W: sanitized ExtractedInvoice + SanitizationReport
  W->>E: recordSanitization + recordExtraction
  W->>A: runSingleTask(VALIDATE_LINE_ITEMS, sanitizedExtract)
  A->>G: before-tool-call(lookupGlAccount, VALIDATE)
  G-->>A: accept (status VALIDATING and extracted present)
  A->>T: lookupGlAccount × N + checkBalance
  T-->>A: List<ValidatedLineItem> + BalanceResult
  A-->>W: ValidatedInvoice
  W->>E: recordValidation
  Note over W: totalAmount 3 200.00 ≤ threshold — skip approval
  W->>A: runSingleTask(POST_JOURNAL_ENTRY, validatedInvoice)
  A->>G: before-tool-call(buildJournalEntry, POST)
  G-->>A: accept (status POSTING and validated present)
  A->>T: buildJournalEntry + confirmPosting
  T-->>A: JournalEntry + PostingConfirmation
  A-->>W: PostedInvoice
  W->>E: recordPosting
  W->>Sc: score(postedInvoice, validatedInvoice)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `InvoiceEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> EXTRACTING: ExtractionStarted
  EXTRACTING --> EXTRACTED: ExtractionCompleted + SanitizationApplied
  EXTRACTED --> VALIDATING: ValidationStarted
  VALIDATING --> VALIDATED: ValidationCompleted
  VALIDATED --> APPROVAL_REQUESTED: ApprovalRequested (high-value)
  VALIDATED --> POSTING: PostingStarted (below threshold)
  APPROVAL_REQUESTED --> APPROVED: ApprovalGranted
  APPROVAL_REQUESTED --> DENIED: ApprovalDenied
  APPROVED --> POSTING: PostingStarted
  POSTING --> POSTED: PostingCompleted
  POSTED --> EVALUATED: EvaluationScored
  EXTRACTING --> FAILED: InvoiceFailed
  VALIDATING --> FAILED: InvoiceFailed
  POSTING --> FAILED: InvoiceFailed
  APPROVAL_REQUESTED --> FAILED: InvoiceFailed (timeout)
  EVALUATED --> [*]
  DENIED --> [*]
  FAILED --> [*]
```

`PhaseGuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  InvoiceEntity ||--o{ InvoiceCreated : emits
  InvoiceEntity ||--o{ ExtractionStarted : emits
  InvoiceEntity ||--o{ SanitizationApplied : emits
  InvoiceEntity ||--o{ ExtractionCompleted : emits
  InvoiceEntity ||--o{ ValidationStarted : emits
  InvoiceEntity ||--o{ ValidationCompleted : emits
  InvoiceEntity ||--o{ ApprovalRequested : emits
  InvoiceEntity ||--o{ ApprovalGranted : emits
  InvoiceEntity ||--o{ ApprovalDenied : emits
  InvoiceEntity ||--o{ PostingStarted : emits
  InvoiceEntity ||--o{ PostingCompleted : emits
  InvoiceEntity ||--o{ EvaluationScored : emits
  InvoiceEntity ||--o{ PhaseGuardrailRejected : emits
  InvoiceEntity ||--o{ InvoiceFailed : emits
  InvoiceView }o--|| InvoiceEntity : projects
  InvoicePipelineWorkflow }o--|| InvoiceEntity : reads-and-writes
  InvoiceAgent ||--o{ ExtractedInvoice : returns
  InvoiceAgent ||--o{ ValidatedInvoice : returns
  InvoiceAgent ||--o{ PostedInvoice : returns
  VendorPiiSanitizer ||--o{ SanitizationReport : produces
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `InvoiceEndpoint` | `api/InvoiceEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `InvoiceEntity` | `application/InvoiceEntity.java` (state in `domain/InvoiceRecord.java`, events in `domain/InvoiceEvent.java`) |
| `InvoicePipelineWorkflow` | `application/InvoicePipelineWorkflow.java` |
| `InvoiceAgent` | `application/InvoiceAgent.java` (tasks in `application/InvoiceTasks.java`) |
| `ExtractTools` | `application/ExtractTools.java` |
| `ValidateTools` | `application/ValidateTools.java` |
| `PostTools` | `application/PostTools.java` |
| `PhaseGuardrail` | `application/PhaseGuardrail.java` |
| `VendorPiiSanitizer` | `application/VendorPiiSanitizer.java` |
| `PostingQualityScorer` | `application/PostingQualityScorer.java` |
| `InvoiceView` | `application/InvoiceView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `extractStep` 60 s, `validateStep` 60 s, `postStep` 60 s, `evalStep` 5 s, `approvalStep` 172 800 s (48 h), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(InvoicePipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + invoiceId` as the workflow id; restart of the same invoiceId is rejected by the workflow runtime. The agent instance id is `"agent-" + invoiceId` so each invoice has its own per-task conversation memory.
- **One agent per invoice**: `InvoiceAgent` runs three tasks per invoice — EXTRACT, VALIDATE, POST — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Sanitizer is not the agent's responsibility**: `VendorPiiSanitizer` runs in-process inside `extractStep`, between the agent's typed return and the entity write. The agent never sees the raw PII after the sanitizer fires, and the sanitizer never makes an LLM call.
- **Approval gate is workflow-managed**: `approvalStep` reads the entity state (not the agent) to decide whether to suspend. When the reviewer calls the approve/deny endpoint, the entity emits the corresponding event and the workflow polls its own entity to detect the decision and resume.
- **Eval is synchronous and deterministic**: `PostingQualityScorer` runs in-process inside `evalStep`. No LLM call, no external service.
- **Task-boundary handoff is the dependency contract**: `extractStep` writes `ExtractionCompleted` BEFORE returning; `validateStep` reads the recorded sanitized `ExtractedInvoice` from the entity to build its task's instruction context; `postStep` reads the `ValidatedInvoice`. The agent itself is stateless across phases.
