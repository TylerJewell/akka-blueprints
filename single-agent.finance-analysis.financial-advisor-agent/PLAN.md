# PLAN — financial-advisor-agent

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

  API[AdvisoryEndpoint]:::ep
  Entity[AdvisoryEntity]:::ese
  Sanitizer[ClientDataSanitizer]:::cons
  WF[AdvisoryWorkflow]:::wf
  Agent[FinancialAdvisorAgent]:::agent
  Guard[RecommendationGuardrail]:::guard
  View[AdvisoryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|AdvisoryRequested| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|adviseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|AdvisoryResponse| WF
  WF -->|recordResponse| Entity
  WF -->|auditStep digest| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as AdvisoryEndpoint
  participant E as AdvisoryEntity
  participant S as ClientDataSanitizer
  participant W as AdvisoryWorkflow
  participant A as FinancialAdvisorAgent
  participant G as RecommendationGuardrail

  U->>API: POST /api/advisories
  API->>E: submit(request)
  E-->>API: { advisoryId }
  E-.->>S: AdvisoryRequested
  S->>S: strip regulated identifiers
  S->>E: attachSanitized
  S->>W: start(advisoryId)
  W->>E: poll getAdvisory
  E-->>W: sanitized.isPresent()
  W->>E: markAdvising
  W->>A: runSingleTask(question + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: AdvisoryResponse
  W->>E: recordResponse(response)
  W->>E: recordAudit(digestHex)
  E-.->>U: SSE event(AUDITED)
```

## State machine — `AdvisoryEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> SANITIZED: ClientDataSanitized
  SANITIZED --> ADVISING: AdvisingStarted
  ADVISING --> RESPONSE_RECORDED: ResponseRecorded
  RESPONSE_RECORDED --> AUDITED: AuditLogged
  ADVISING --> FAILED: AdvisoryFailed (agent error)
  REQUESTED --> FAILED: AdvisoryFailed (sanitizer error)
  AUDITED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  AdvisoryEntity ||--o{ AdvisoryRequested : emits
  AdvisoryEntity ||--o{ ClientDataSanitized : emits
  AdvisoryEntity ||--o{ AdvisingStarted : emits
  AdvisoryEntity ||--o{ ResponseRecorded : emits
  AdvisoryEntity ||--o{ AuditLogged : emits
  AdvisoryEntity ||--o{ AdvisoryFailed : emits
  AdvisoryView }o--|| AdvisoryEntity : projects
  ClientDataSanitizer }o--|| AdvisoryEntity : subscribes
  AdvisoryWorkflow }o--|| AdvisoryEntity : reads-and-writes
  FinancialAdvisorAgent ||--o{ AdvisoryResponse : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AdvisoryEndpoint` | `api/AdvisoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AdvisoryEntity` | `application/AdvisoryEntity.java` (state in `domain/Advisory.java`, events in `domain/AdvisoryEvent.java`) |
| `ClientDataSanitizer` | `application/ClientDataSanitizer.java` |
| `AdvisoryWorkflow` | `application/AdvisoryWorkflow.java` |
| `FinancialAdvisorAgent` | `application/FinancialAdvisorAgent.java` (tasks in `application/AdvisoryTasks.java`) |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `AdvisoryView` | `application/AdvisoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `adviseStep` 60 s, `auditStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AdvisoryWorkflow::error)`. The 60 s on `adviseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"advisory-" + advisoryId` as the workflow id; the `ClientDataSanitizer` Consumer is allowed to redeliver `AdvisoryRequested` events because `AdvisoryEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized advisory is a no-op.
- **One agent per advisory**: the AutonomousAgent instance id is `"advisor-" + advisoryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecommendationGuardrail` rejects a candidate response, the rejection returns as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `adviseStep` fails over to `error` and the entity transitions to `FAILED`.
- **Audit step is synchronous and deterministic**: the `auditStep` computes a SHA-256 digest over the serialized `AdvisoryResponse` and appends it to the entity. No LLM call, no external service. This is what keeps the single-agent invariant: exactly one component talks to a model.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
