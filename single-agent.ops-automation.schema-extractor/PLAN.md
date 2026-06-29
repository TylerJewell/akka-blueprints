# PLAN — schema-extractor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ExtractionEndpoint]:::ep
  Entity[ExtractionJobEntity]:::ese
  Sanitizer[DocumentSanitizer]:::cons
  WF[ExtractionWorkflow]:::wf
  Agent[ExtractionAgent]:::agent
  Guard[RecordGuardrail]:::guard
  View[ExtractionView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|DocumentSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|extractStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Guard -.->|accept / reject| Agent
  Agent -->|ExtractedRecord| WF
  WF -->|recordExtracted| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ExtractionEndpoint
  participant E as ExtractionJobEntity
  participant S as DocumentSanitizer
  participant W as ExtractionWorkflow
  participant A as ExtractionAgent
  participant G as RecordGuardrail

  U->>API: POST /api/jobs
  API->>E: submit(request)
  E-->>API: { jobId }
  E-.->>S: DocumentSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(jobId)
  W->>E: poll getJob
  E-->>W: sanitized.isPresent()
  W->>E: markExtracting
  W->>A: runSingleTask(schema + attachment)
  A->>G: after-llm-response(candidate)
  G-->>A: accept
  A-->>W: ExtractedRecord
  W->>E: recordExtracted(record)
  E-.->>U: SSE event(RECORD_EXTRACTED)
```

## State machine — `ExtractionJobEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: DocumentSanitized
  SANITIZED --> EXTRACTING: ExtractionStarted
  EXTRACTING --> RECORD_EXTRACTED: RecordExtracted
  EXTRACTING --> FAILED: ExtractionFailed (agent error)
  SUBMITTED --> FAILED: ExtractionFailed (sanitizer error)
  RECORD_EXTRACTED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExtractionJobEntity ||--o{ DocumentSubmitted : emits
  ExtractionJobEntity ||--o{ DocumentSanitized : emits
  ExtractionJobEntity ||--o{ ExtractionStarted : emits
  ExtractionJobEntity ||--o{ RecordExtracted : emits
  ExtractionJobEntity ||--o{ ExtractionFailed : emits
  ExtractionView }o--|| ExtractionJobEntity : projects
  DocumentSanitizer }o--|| ExtractionJobEntity : subscribes
  ExtractionWorkflow }o--|| ExtractionJobEntity : reads-and-writes
  ExtractionAgent ||--o{ ExtractedRecord : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExtractionEndpoint` | `api/ExtractionEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ExtractionJobEntity` | `application/ExtractionJobEntity.java` (state in `domain/ExtractionJob.java`, events in `domain/ExtractionEvent.java`) |
| `DocumentSanitizer` | `application/DocumentSanitizer.java` |
| `ExtractionWorkflow` | `application/ExtractionWorkflow.java` |
| `ExtractionAgent` | `application/ExtractionAgent.java` (tasks in `application/ExtractionTasks.java`) |
| `RecordGuardrail` | `application/RecordGuardrail.java` |
| `ExtractionView` | `application/ExtractionView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `extractStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ExtractionWorkflow::error)`. The 60 s on `extractStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"extraction-" + jobId` as the workflow id; the `DocumentSanitizer` Consumer is allowed to redeliver `DocumentSubmitted` events because `ExtractionJobEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized job is a no-op.
- **One agent per job**: the AutonomousAgent instance id is `"extractor-" + jobId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecordGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `extractStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
