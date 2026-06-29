# PLAN — ghostwriter

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

  API[DraftEndpoint]:::ep
  Entity[DraftEntity]:::ese
  Sanitizer[CorpusSanitizer]:::cons
  WF[DraftWorkflow]:::wf
  Agent[GhostwriterAgent]:::agent
  Guard[DraftOutputGuardrail]:::guard
  View[DraftView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|BriefSubmitted| Sanitizer
  Sanitizer -->|attachSanitizedCorpus| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|draftStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Guard -->|pass or reject| Agent
  Agent -->|DraftResult| WF
  WF -->|recordDraft| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as DraftEndpoint
  participant E as DraftEntity
  participant S as CorpusSanitizer
  participant W as DraftWorkflow
  participant A as GhostwriterAgent
  participant G as DraftOutputGuardrail

  U->>API: POST /api/drafts
  API->>E: submit(brief)
  E-->>API: { draftId }
  E-.->>S: BriefSubmitted
  S->>S: redact PII in each sample
  S->>E: attachSanitizedCorpus
  S->>W: start(draftId)
  W->>E: poll getDraft
  E-->>W: sanitizedCorpus.isPresent()
  W->>E: markDrafting
  W->>A: runSingleTask(brief instructions + sample attachments)
  A->>G: after-llm-response(candidate)
  G-->>A: accept
  A-->>W: DraftResult
  W->>E: recordDraft(result)
  E-.->>U: SSE event(DRAFT_READY)
```

## State machine — `DraftEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: CorpusSanitized
  SANITIZED --> DRAFTING: DraftingStarted
  DRAFTING --> DRAFT_READY: DraftReady
  DRAFTING --> FAILED: DraftFailed (agent error or guardrail exhaustion)
  SUBMITTED --> FAILED: DraftFailed (sanitizer error)
  DRAFT_READY --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DraftEntity ||--o{ BriefSubmitted : emits
  DraftEntity ||--o{ CorpusSanitized : emits
  DraftEntity ||--o{ DraftingStarted : emits
  DraftEntity ||--o{ DraftReady : emits
  DraftEntity ||--o{ DraftFailed : emits
  DraftView }o--|| DraftEntity : projects
  CorpusSanitizer }o--|| DraftEntity : subscribes
  DraftWorkflow }o--|| DraftEntity : reads-and-writes
  GhostwriterAgent ||--o{ DraftResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DraftEndpoint` | `api/DraftEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DraftEntity` | `application/DraftEntity.java` (state in `domain/Draft.java`, events in `domain/DraftEvent.java`) |
| `CorpusSanitizer` | `application/CorpusSanitizer.java` |
| `DraftWorkflow` | `application/DraftWorkflow.java` |
| `GhostwriterAgent` | `application/GhostwriterAgent.java` (tasks in `application/DraftTasks.java`) |
| `DraftOutputGuardrail` | `application/DraftOutputGuardrail.java` |
| `DraftView` | `application/DraftView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `draftStep` 90 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DraftWorkflow::error)`. The 90 s on `draftStep` accommodates multi-sample corpus processing plus LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"draft-" + draftId` as the workflow id; `CorpusSanitizer` may redeliver `BriefSubmitted` events because `DraftEntity.attachSanitizedCorpus` is event-version-guarded — a second sanitize attempt against an already-sanitized draft is a no-op.
- **One agent per draft**: the AutonomousAgent instance id is `"ghostwriter-" + draftId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `DraftOutputGuardrail` rejects a candidate response, the rejection returns as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, `draftStep` fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
