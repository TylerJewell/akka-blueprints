# PLAN — reply-classifier

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

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
  classDef crm fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[ReplyEndpoint]:::ep
  Entity[ReplyEntity]:::ese
  WF[ClassificationWorkflow]:::wf
  Agent[ReplyClassifierAgent]:::agent
  Guard[CrmMutationGuardrail]:::guard
  CRM[PipedriveClient]:::crm
  View[ReplyView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start workflow| WF
  WF -->|classifyStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -.->|allow / reject| Agent
  Agent -->|ReplyClassification| WF
  WF -->|recordClassification| Entity
  WF -->|crmUpdateStep| CRM
  CRM -->|CrmUpdateResult| WF
  WF -->|recordCrmUpdate / skipCrmUpdate| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ReplyEndpoint
  participant E as ReplyEntity
  participant W as ClassificationWorkflow
  participant A as ReplyClassifierAgent
  participant G as CrmMutationGuardrail
  participant C as PipedriveClient

  U->>API: POST /api/replies
  API->>E: submit(submission)
  E-->>API: { replyId }
  API->>W: start(replyId)
  W->>E: markClassifying
  W->>A: runSingleTask(context + attachment)
  A->>G: before-tool-call(updateDealStage)
  G-->>A: allow
  A-->>W: ReplyClassification(INTERESTED, UPDATE_STAGE)
  W->>E: recordClassification
  W->>C: updateDealStage(dealId, QUALIFIED)
  C-->>W: CrmUpdateResult(success=true)
  W->>E: recordCrmUpdate(result)
  E-.->>U: SSE event(CRM_UPDATED)
```

## State machine — `ReplyEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> CLASSIFYING: ClassificationStarted
  CLASSIFYING --> CLASSIFIED: ReplyClassified
  CLASSIFIED --> CRM_UPDATED: CrmUpdateSucceeded
  CLASSIFIED --> CRM_SKIPPED: CrmUpdateSkipped
  CLASSIFYING --> FAILED: ReplyFailed (agent error)
  RECEIVED --> FAILED: ReplyFailed (workflow start error)
  CRM_UPDATED --> [*]
  CRM_SKIPPED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ReplyEntity ||--o{ ReplyReceived : emits
  ReplyEntity ||--o{ ClassificationStarted : emits
  ReplyEntity ||--o{ ReplyClassified : emits
  ReplyEntity ||--o{ CrmUpdateSucceeded : emits
  ReplyEntity ||--o{ CrmUpdateSkipped : emits
  ReplyEntity ||--o{ ReplyFailed : emits
  ReplyView }o--|| ReplyEntity : projects
  ClassificationWorkflow }o--|| ReplyEntity : reads-and-writes
  ReplyClassifierAgent ||--o{ ReplyClassification : returns
  CrmMutationGuardrail }o--|| ReplyClassifierAgent : guards
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ReplyEndpoint` | `api/ReplyEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ReplyEntity` | `application/ReplyEntity.java` (state in `domain/Reply.java`, events in `domain/ReplyEvent.java`) |
| `ClassificationWorkflow` | `application/ClassificationWorkflow.java` |
| `ReplyClassifierAgent` | `application/ReplyClassifierAgent.java` (tasks in `application/ReplyTasks.java`) |
| `CrmMutationGuardrail` | `application/CrmMutationGuardrail.java` |
| `PipedriveClient` | `application/PipedriveClient.java` (interface + `SimulatedPipedriveClient.java`) |
| `ReplyView` | `application/ReplyView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `classifyStep` 60 s, `crmUpdateStep` 15 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ClassificationWorkflow::error)`. The 60 s on `classifyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"classify-" + replyId` as the workflow id; starting a second workflow for the same `replyId` is a no-op because the entity is already past `RECEIVED`.
- **One agent per reply**: the AutonomousAgent instance id is `"classifier-" + replyId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `CrmMutationGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `classifyStep` fails over to `error` and the entity transitions to `FAILED`.
- **CRM update is a workflow step, not an agent tool**: `PipedriveClient.updateDealStage` is called in `crmUpdateStep`, not directly from agent code. The guardrail governs the agent's *intent* (proposed tool call); the workflow's step makes the actual external call after the guardrail passes.
- **No saga / no compensation**: each step is either an append-only entity write or a single external call. There is nothing to roll back beyond marking the entity `FAILED`.
