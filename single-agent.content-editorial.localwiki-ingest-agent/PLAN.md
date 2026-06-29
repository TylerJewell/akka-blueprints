# PLAN — localwiki-ingest-agent

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

  API[IngestEndpoint]:::ep
  Entity[IngestEntity]:::ese
  Sanitizer[ContentSanitizer]:::cons
  WF[IngestWorkflow]:::wf
  Agent[WikiIngestAgent]:::agent
  Guard[WritePageGuardrail]:::guard
  View[WikiPageView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|SourceSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|ingestStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|pass/reject| Agent
  Agent -->|WikiPage| WF
  WF -->|recordPage| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as IngestEndpoint
  participant E as IngestEntity
  participant S as ContentSanitizer
  participant W as IngestWorkflow
  participant A as WikiIngestAgent
  participant G as WritePageGuardrail

  U->>API: POST /api/ingests
  API->>E: submit(submission)
  E-->>API: { ingestId }
  E-.->>S: SourceSubmitted
  S->>S: fetch + redact PII
  S->>E: attachSanitized
  S->>W: start(ingestId)
  W->>E: poll getIngest
  E-->>W: sanitized.isPresent()
  W->>E: markIngesting
  W->>A: runSingleTask(filing instructions + attachment)
  A->>G: before-tool-call(write_wiki_page args)
  G-->>A: accept
  A-->>W: WikiPage (tool result + task result)
  W->>E: recordPage(page)
  E-.->>U: SSE event(PAGE_FILED)
```

## State machine — `IngestEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ContentSanitized
  SANITIZED --> INGESTING: IngestStarted
  INGESTING --> PAGE_FILED: PageFiled
  SUBMITTED --> FAILED: IngestFailed (sanitizer / fetch error)
  INGESTING --> FAILED: IngestFailed (agent error / guardrail-exhausted)
  PAGE_FILED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  IngestEntity ||--o{ SourceSubmitted : emits
  IngestEntity ||--o{ ContentSanitized : emits
  IngestEntity ||--o{ IngestStarted : emits
  IngestEntity ||--o{ PageFiled : emits
  IngestEntity ||--o{ IngestFailed : emits
  WikiPageView }o--|| IngestEntity : projects
  ContentSanitizer }o--|| IngestEntity : subscribes
  IngestWorkflow }o--|| IngestEntity : reads-and-writes
  WikiIngestAgent ||--o{ WikiPage : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `IngestEndpoint` | `api/IngestEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `IngestEntity` | `application/IngestEntity.java` (state in `domain/Ingest.java`, events in `domain/IngestEvent.java`) |
| `ContentSanitizer` | `application/ContentSanitizer.java` |
| `IngestWorkflow` | `application/IngestWorkflow.java` |
| `WikiIngestAgent` | `application/WikiIngestAgent.java` (tasks in `application/IngestTasks.java`) |
| `WritePageGuardrail` | `application/WritePageGuardrail.java` |
| `WikiPageView` | `application/WikiPageView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `ingestStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(IngestWorkflow::error)`. The 60 s on `ingestStep` accommodates LLM latency plus the in-process fetch simulation (Lesson 4).
- **Idempotency**: every workflow uses `"ingest-" + ingestId` as the workflow id; the `ContentSanitizer` Consumer is allowed to redeliver `SourceSubmitted` events because `IngestEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized ingest is a no-op.
- **One agent per ingest**: the AutonomousAgent instance id is `"ingest-agent-" + ingestId`, which gives each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `WritePageGuardrail` rejects a proposed tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `ingestStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
