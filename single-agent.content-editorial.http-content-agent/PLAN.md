# PLAN — http-content-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture
tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24;
without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ContentEndpoint]:::ep
  Entity[ContentJobEntity]:::ese
  WF[ContentWorkflow]:::wf
  Agent[ContentGeneratorAgent]:::agent
  Guard[DraftGuardrail]:::guard
  View[ContentView]:::view
  App[AppEndpoint]:::ep

  API -->|requestJob| Entity
  API -->|start workflow| WF
  WF -->|generateStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|reject or pass| Agent
  Agent -->|GeneratedContent| WF
  WF -->|approveContent| Entity
  WF -->|failJob| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ContentEndpoint
  participant E as ContentJobEntity
  participant W as ContentWorkflow
  participant A as ContentGeneratorAgent
  participant G as DraftGuardrail

  U->>API: POST /api/jobs
  API->>E: requestJob(brief)
  E-->>API: { jobId }
  API->>W: start(jobId)
  W->>E: startGeneration
  W->>A: runSingleTask(brief as instructions)
  A->>G: before-agent-response(candidate draft)
  G-->>A: accept
  A-->>W: GeneratedContent
  W->>E: approveContent(content)
  E-.->>U: SSE event(APPROVED)
```

## State machine — `ContentJobEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> GENERATING: GenerationStarted
  GENERATING --> APPROVED: ContentApproved
  GENERATING --> FAILED: JobFailed (agent error or guardrail-exhaustion)
  APPROVED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ContentJobEntity ||--o{ JobRequested : emits
  ContentJobEntity ||--o{ GenerationStarted : emits
  ContentJobEntity ||--o{ ContentApproved : emits
  ContentJobEntity ||--o{ JobFailed : emits
  ContentView }o--|| ContentJobEntity : projects
  ContentWorkflow }o--|| ContentJobEntity : reads-and-writes
  ContentGeneratorAgent ||--o{ GeneratedContent : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ContentEndpoint` | `api/ContentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ContentJobEntity` | `application/ContentJobEntity.java` (state in `domain/ContentJob.java`, events in `domain/ContentJobEvent.java`) |
| `ContentWorkflow` | `application/ContentWorkflow.java` |
| `ContentGeneratorAgent` | `application/ContentGeneratorAgent.java` (tasks in `application/ContentTasks.java`) |
| `DraftGuardrail` | `application/DraftGuardrail.java` |
| `ContentView` | `application/ContentView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `generateStep` 90 s, `error` 5 s. Default step recovery
  `maxRetries(2).failoverTo(ContentWorkflow::error)`. The 90 s on `generateStep` accommodates
  LLM latency on longer-form content (Lesson 4).
- **Idempotency**: the workflow uses `"content-" + jobId` as the workflow id; re-delivery of
  the `ContentEndpoint` start call is harmless because the workflow id is fixed.
- **One agent per job**: the AutonomousAgent instance id is `"generator-" + jobId`, giving each
  task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps
  guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `DraftGuardrail` rejects a candidate response, the rejection
  names the specific failed check(s) in the structured error returned to the agent loop. The
  loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation the workflow's
  `generateStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either an entity command write or a single-task
  agent call. There is nothing external to roll back.
