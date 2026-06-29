# PLAN — High-Volume Document Analyzer

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  DE[DocumentEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  BQ[BatchQueue<br/>EventSourcedEntity]:::ese
  BC[BatchRequestConsumer<br/>Consumer]:::con
  WF[BatchWorkflow<br/>Workflow]:::wf
  CO[BatchCoordinator<br/>AutonomousAgent]:::ag
  EX[Extractor<br/>AutonomousAgent]:::ag
  CL[Classifier<br/>AutonomousAgent]:::ag
  DOC[DocumentEntity<br/>EventSourcedEntity]:::ese
  BAT[BatchEntity<br/>EventSourcedEntity]:::ese
  VW[DocumentView<br/>View]:::vw
  SIM[DocumentSimulator<br/>TimedAction]:::ta
  QS[QualitySampler<br/>TimedAction]:::ta

  DE -->|POST /batches| BQ
  SIM -.->|every 90s| BQ
  BQ -.->|BatchSubmitted| BC
  BC -->|start workflow per doc| WF
  WF -->|PARTITION| CO
  WF -->|EXTRACT| EX
  WF -->|CLASSIFY| CL
  WF -->|MERGE| CO
  WF -->|commands| DOC
  WF -->|commands| BAT
  DOC -.->|events| VW
  BAT -.->|events| VW
  QS -.->|every 5m| VW
  QS -->|recordQuality| DOC
  DE -->|getAllDocuments / SSE| VW
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant DE as DocumentEndpoint
  participant BQ as BatchQueue
  participant WF as BatchWorkflow
  participant CO as BatchCoordinator
  participant EX as Extractor
  participant CL as Classifier
  participant DOC as DocumentEntity
  participant BAT as BatchEntity

  U->>DE: POST /api/documents/batches {rawDocuments}
  DE->>BQ: submitBatch
  BQ-->>WF: BatchRequestConsumer starts one workflow per doc
  WF->>BAT: createBatch + startBatch (PENDING → IN_PROGRESS)
  WF->>DOC: queueDocument (QUEUED)
  WF->>CO: PARTITION -> WorkPartition
  WF->>DOC: startProcessing (PROCESSING)
  par parallel fan-out
    WF->>EX: EXTRACT -> ExtractedFields
  and
    WF->>CL: CLASSIFY -> DocumentClassification
  end
  Note over WF: join; if either step times out (60s) -> degradeStep
  WF->>CO: MERGE(fields, classification) -> ProcessedDocument
  WF->>WF: sanitizeStep scans sanitizedText for PII
  WF->>DOC: markProcessed (PROCESSED)
  WF->>BAT: documentDone -> BatchComplete or BatchPartiallyComplete
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> PROCESSING: startProcessing
  PROCESSING --> PROCESSED: merge + sanitize pass
  PROCESSING --> DEGRADED: a worker timed out
  DEGRADED --> [*]
  PROCESSED --> PROCESSED: QualityScored
  PROCESSED --> [*]
```

## Entity model

```mermaid
erDiagram
  BATCH ||--o{ DOCUMENT : contains
  DOCUMENT ||--o| EXTRACTED_FIELDS : produces
  DOCUMENT ||--o| DOCUMENT_CLASSIFICATION : produces
  DOCUMENT ||--o| PROCESSED_DOCUMENT : finalizes
  BATCH_QUEUE ||--|| BATCH : seeds
  DOCUMENT {
    string documentId
    string batchId
    enum status
    int qualityScore
    instant createdAt
  }
  BATCH {
    string batchId
    string submittedBy
    enum status
    int totalDocuments
    int processedCount
    int degradedCount
    instant createdAt
  }
  BATCH_QUEUE {
    string batchId
    string submittedBy
    int documentCount
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `BatchCoordinator` | AutonomousAgent | `application/BatchCoordinator.java` |
| `Extractor` | AutonomousAgent | `application/Extractor.java` |
| `Classifier` | AutonomousAgent | `application/Classifier.java` |
| `DocumentTasks` | Task constants | `application/DocumentTasks.java` |
| `BatchWorkflow` | Workflow | `application/BatchWorkflow.java` |
| `DocumentEntity` | EventSourcedEntity | `domain/DocumentEntity.java` |
| `BatchEntity` | EventSourcedEntity | `domain/BatchEntity.java` |
| `BatchQueue` | EventSourcedEntity | `domain/BatchQueue.java` |
| `DocumentView` | View | `application/DocumentView.java` |
| `BatchRequestConsumer` | Consumer | `application/BatchRequestConsumer.java` |
| `DocumentSimulator` | TimedAction | `application/DocumentSimulator.java` |
| `QualitySampler` | TimedAction | `application/QualitySampler.java` |
| `DocumentEndpoint` | HttpEndpoint | `api/DocumentEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `extractStep` and `classifyStep` each get 60s; `mergeStep` gets 90s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `extractStep` and `classifyStep` run concurrently via `CompletionStage` zip, not two sequential step calls.
- **Idempotency:** the workflow id is the `documentId`. Re-delivery of the same `BatchSubmitted` event resolves to the same workflow instance per document — no duplicate processing.
- **Degrade path (compensation):** if either worker times out, `defaultStepRecovery` routes to `degradeStep`, which merges from whichever partial output exists and ends with `DocumentDegraded`. No infinite retry. The batch continues with remaining documents.
- **Quality sampling:** `QualitySampler` reads `DocumentView.getAllDocuments` (no enum WHERE clause — Lesson 2) and filters client-side for the oldest `PROCESSED` document lacking a `qualityScore`.
- **Batch completion:** `BatchEntity` receives a `documentDone` command for each finished document (PROCESSED or DEGRADED). When `processedCount + degradedCount == totalDocuments`, it emits `BatchComplete` if `degradedCount == 0`, else `BatchPartiallyComplete`.
