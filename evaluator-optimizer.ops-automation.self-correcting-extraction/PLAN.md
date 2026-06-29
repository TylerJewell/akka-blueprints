# PLAN — self-correcting-extraction

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams are rendered on the generated system's Architecture tab.

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

  Extractor[ExtractionAgent]:::agent
  Scorer[ScorerAgent]:::agent

  WF[ExtractionWorkflow]:::wf
  Job[ExtractionJobEntity]:::ese
  Mem[MemoryEntity]:::ese
  Queue[DocumentQueue]:::ese
  View[JobsView]:::view
  Consumer[DocumentConsumer]:::cons
  Sim[DocumentSimulator]:::ta
  Eval[EvalSampler]:::ta
  API[ExtractionEndpoint]:::ep
  App[AppEndpoint]:::ep

  API -->|submit document| Queue
  Sim -.->|every 60s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|extract / correct| Extractor
  WF -->|score| Scorer
  WF -->|emit events| Job
  WF -->|write verified fields| Mem
  Scorer -.->|read ground truth| Mem
  Job -.->|projects| View
  API -->|query / SSE| View
  API -->|read memory| Mem
  Eval -.->|every 30s| Job
```

## Interaction sequence — J1 (convergence on attempt 2)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ExtractionEndpoint
  participant Q as DocumentQueue
  participant C as DocumentConsumer
  participant W as ExtractionWorkflow
  participant X as ExtractionAgent
  participant S as ScorerAgent
  participant M as MemoryEntity
  participant E as ExtractionJobEntity
  participant V as JobsView

  U->>API: POST /api/jobs {documentType, rawText}
  API->>Q: append DocumentSubmitted
  API-->>U: 202 {jobId}
  Q->>C: DocumentSubmitted
  C->>W: start({jobId, documentType, rawText, budgetCap=4})
  W->>E: emit JobCreated (EXTRACTING)

  W->>X: EXTRACT(documentType, rawText)
  X-->>W: FieldMap #1 (confidence 0.55)
  W->>E: emit AttemptExtracted (n=1)
  W->>M: getMemory(documentType)
  M-->>W: MemoryContext (prior confirmed fields)
  W->>S: SCORE(FieldMap #1, memoryContext)
  S-->>W: ScorerVerdict{CORRECT, confidence=0.55, 2 bullets}
  W->>E: emit AttemptScored (n=1, CORRECT)
  W->>E: status SCORING → EXTRACTING

  W->>X: CORRECT_EXTRACTION(documentType, rawText, windowBuffer, notes)
  X-->>W: FieldMap #2 (confidence 0.93)
  W->>E: emit AttemptExtracted (n=2)
  W->>S: SCORE(FieldMap #2, memoryContext)
  S-->>W: ScorerVerdict{PASS, confidence=0.93}
  W->>E: emit AttemptScored (n=2, PASS)
  W->>E: emit JobVerified (n=2)
  W->>M: writeMemory(confirmedFields)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ExtractionJobEntity`

```mermaid
stateDiagram-v2
  [*] --> EXTRACTING
  EXTRACTING --> SCORING: AttemptExtracted
  SCORING --> EXTRACTING: ScorerDecision = CORRECT, attempts < budget
  SCORING --> VERIFIED: ScorerDecision = PASS
  SCORING --> BUDGET_EXHAUSTED: ScorerDecision = CORRECT, attempts = budget
  VERIFIED --> [*]
  BUDGET_EXHAUSTED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExtractionJobEntity ||--o{ JobCreated : emits
  ExtractionJobEntity ||--o{ AttemptExtracted : emits
  ExtractionJobEntity ||--o{ AttemptScored : emits
  ExtractionJobEntity ||--o{ JobVerified : emits
  ExtractionJobEntity ||--o{ JobBudgetExhausted : emits
  ExtractionJobEntity ||--o{ EvalRecorded : emits
  MemoryEntity ||--o{ MemoryRecordWritten : emits
  JobsView }o--|| ExtractionJobEntity : projects
  DocumentQueue ||--o{ DocumentSubmitted : emits
  DocumentConsumer }o--|| DocumentQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExtractionAgent` | `application/ExtractionAgent.java` |
| `ScorerAgent` | `application/ScorerAgent.java` |
| `ExtractionTasks` | `application/ExtractionTasks.java` |
| `ExtractionWorkflow` | `application/ExtractionWorkflow.java` |
| `ExtractionJobEntity` | `application/ExtractionJobEntity.java` (state in `domain/ExtractionJob.java`, events in `domain/JobEvent.java`) |
| `MemoryEntity` | `application/MemoryEntity.java` |
| `DocumentQueue` | `application/DocumentQueue.java` |
| `JobsView` | `application/JobsView.java` |
| `DocumentConsumer` | `application/DocumentConsumer.java` |
| `DocumentSimulator` | `application/DocumentSimulator.java` |
| `EvalSampler` | `application/EvalSampler.java` |
| `ExtractionEndpoint` | `api/ExtractionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MockModelProvider` (option (a) only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `extractStep` and `scoreStep` each carry `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep))` — the workflow degrades to `BUDGET_EXHAUSTED` on irrecoverable agent failure rather than hanging.
- **Window buffer:** the workflow state carries a `List<FieldMap>` capped at 3 entries (configured via `self-correcting-extraction.extraction.window-buffer-size`). Each extractStep prepends the new FieldMap and trims to the cap before passing the buffer to the next CORRECT_EXTRACTION call.
- **Memory write ordering:** `verifyStep` writes to `MemoryEntity` only after `JobVerified` is emitted on `ExtractionJobEntity`; the order is enforced by step sequencing, not by a compensating saga.
- **Idempotency:** `ExtractionEndpoint.submit` uses `(documentType, submittedBy)` over a 10 s window as the dedup key. `EvalSampler` keys its `recordEval` calls on `(jobId, attemptNumber)` so a tick that fires twice is a no-op.
- **budgetCap ceiling:** read from `self-correcting-extraction.extraction.budget-cap` (default 4). The workflow checks `attemptCount < budgetCap` BEFORE calling `extractStep` for the next iteration.
- **Memory reads are advisory:** the Scorer receives the memory context as a hint; if `MemoryEntity` has no record for a document type, the context is an empty map and the Scorer falls back to rubric-only scoring.
