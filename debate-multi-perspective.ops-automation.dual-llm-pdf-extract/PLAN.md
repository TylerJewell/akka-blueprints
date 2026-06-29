# PLAN — dual-llm-pdf-extract

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab. All four mermaid diagrams use the Akka theme palette; the state diagram carries the Lesson 24 CSS overrides so state names render white and edge labels are not clipped.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Claude[ClaudeExtractor]:::agent
  Gemini[GeminiExtractor]:::agent
  Reconciler[ExtractionReconciler]:::agent
  Judge[EvalJudge]:::agent

  WF[ExtractionWorkflow]:::wf
  Extraction[ExtractionEntity]:::ese
  Queue[DocumentQueue]:::ese
  View[ExtractionView]:::view
  Consumer[DocumentConsumer]:::cons
  Sim[PdfSimulator]:::ta
  Sampler[AgreementSampler]:::ta
  API[ExtractionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|enqueue document| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|extract| Claude
  WF -->|extract| Gemini
  WF -->|reconcile| Reconciler
  WF -->|emit events| Extraction
  Extraction -.->|projects| View
  API -->|query / SSE| View
  Sampler -.->|every 5m| Judge
  Sampler -.->|every 5m| Extraction
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions and scheduled ticks. The PII sanitizer is a deterministic helper invoked inside `ExtractionWorkflow.sanitizeStep` — it has no component box because it makes no Akka call of its own.

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ExtractionEndpoint
  participant Q as DocumentQueue
  participant C as DocumentConsumer
  participant W as ExtractionWorkflow
  participant Cl as ClaudeExtractor
  participant Ge as GeminiExtractor
  participant R as ExtractionReconciler
  participant E as ExtractionEntity
  participant V as ExtractionView

  U->>API: POST /api/extractions {filename, extractedText}
  API->>Q: append DocumentReceived
  API-->>U: 202 {extractionId}
  Q->>C: DocumentReceived
  C->>W: start({extractionId, filename, rawText})
  W->>E: emit ExtractionCreated (INTAKE)
  Note over W: sanitizeStep redacts PII from rawText<br/>raw text never persisted
  W->>E: emit DocumentSanitized (EXTRACTING)
  par
    W->>Cl: extractClaude(redactedText)
    Cl-->>W: RawExtraction(CLAUDE)
  and
    W->>Ge: extractGemini(redactedText)
    Ge-->>W: RawExtraction(GEMINI)
  end
  W->>R: reconcile(claudeExtraction, geminiExtraction)
  R-->>W: MergedExtraction
  W->>E: emit ExtractionReconciled (RECONCILED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ExtractionEntity`

```mermaid
stateDiagram-v2
  [*] --> INTAKE
  INTAKE --> EXTRACTING: DocumentSanitized (PII redacted)
  EXTRACTING --> RECONCILED: both extractors returned; reconciler merged
  EXTRACTING --> DEGRADED: an extractor timed out
  RECONCILED --> RECONCILED: AgreementScored (no status change)
  RECONCILED --> [*]
  DEGRADED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExtractionEntity ||--o{ ExtractionCreated : emits
  ExtractionEntity ||--o{ DocumentSanitized : emits
  ExtractionEntity ||--o{ ClaudeExtractionAttached : emits
  ExtractionEntity ||--o{ GeminiExtractionAttached : emits
  ExtractionEntity ||--o{ ExtractionReconciled : emits
  ExtractionEntity ||--o{ ExtractionDegraded : emits
  ExtractionEntity ||--o{ AgreementScored : emits
  ExtractionView }o--|| ExtractionEntity : projects
  DocumentQueue ||--o{ DocumentReceived : emits
  DocumentConsumer }o--|| DocumentQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ClaudeExtractor` | `application/ClaudeExtractor.java` |
| `GeminiExtractor` | `application/GeminiExtractor.java` |
| `ExtractionReconciler` | `application/ExtractionReconciler.java` |
| `EvalJudge` | `application/EvalJudge.java` |
| `ExtractionTasks` | `application/ExtractionTasks.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ExtractionWorkflow` | `application/ExtractionWorkflow.java` |
| `ExtractionEntity` | `application/ExtractionEntity.java` (state in `domain/Extraction.java`, events in `domain/ExtractionEvent.java`) |
| `DocumentQueue` | `application/DocumentQueue.java` |
| `ExtractionView` | `application/ExtractionView.java` |
| `DocumentConsumer` | `application/DocumentConsumer.java` |
| `PdfSimulator` | `application/PdfSimulator.java` |
| `AgreementSampler` | `application/AgreementSampler.java` |
| `ExtractionEndpoint` | `api/ExtractionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Bootstrap` | `Bootstrap.java` |

Akka component count: **2 http-endpoint · 2 timed-action · 1 view · 1 workflow · 1 service-setup · 4 autonomous-agent · 1 consumer · 2 event-sourced-entity**.

## Concurrency notes

- **Workflow step timeouts:** wrap the two extractor calls and the reconcile call in `WorkflowSettings.builder().stepTimeout(MyStep, Duration.ofSeconds(60))`. The default 5-second step timeout is far too short for LLM calls (Lesson 4).
- **Parallel fork:** `claudeStep` and `geminiStep` use Akka's parallel-step idiom (CompletionStage zip). Both extraction calls must be initiated before either is awaited; sequential calls would defeat the debate-multi-perspective pattern and eliminate the cross-model comparison signal.
- **Degraded path:** on any extractor timeout, transition to reconciliation from partial input rather than failing the whole workflow. `failureReason` names the missing extractor; status is `DEGRADED`. The reconciler still produces a `MergedExtraction` but with `disagreementCount = 0` and a note that only one model returned.
- **Sanitizer ordering:** `sanitizeStep` runs before either extractor is called. The raw text lives only in the workflow's transient start command and is never written to `ExtractionEntity` — only `redactedText` is persisted. This realises control S1.
- **Idempotency:** `ExtractionEndpoint.submit` uses `(filename, submittedBy)` over a 10-second window as the idempotency key to avoid double-creation on client retry.
- **View indexing:** `ExtractionView` exposes one query, `getAllExtractions`, with no `WHERE status` clause — Akka cannot auto-index the `ExtractionStatus` enum column. Callers filter by status client-side.
- **Eval sampling:** `AgreementSampler` selects the oldest `RECONCILED` extraction with no `agreementScore`, one per tick. `AgreementScored` does not change status; it only populates the score and rationale.
- **emptyState:** `ExtractionEntity.emptyState()` returns `Extraction.initial("", "")` with placeholder identity values and never references `commandContext()` (Lesson 3).
