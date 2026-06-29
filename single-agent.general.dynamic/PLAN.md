# PLAN — dynamic-route-agent

Architectural sketch. All four mermaid diagrams + the component table are required. The generated system renders these on the Architecture tab and must inherit the Lesson 24 state-label CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#001A1A,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#1A1600,stroke:#F5C518,color:#fff;
  classDef view fill:#0D001A,stroke:#A855F7,color:#fff;
  classDef ta fill:#0F0F0F,stroke:#888,color:#fff;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef ext fill:#000,stroke:#444,color:#888,stroke-dasharray:4 3;

  EXT([POST /api/requests]):::ext --> RE
  SIM["RequestSimulator<br/>TimedAction"]:::ta -.->|every 45s| RE
  RE["RouteEndpoint<br/>HttpEndpoint"]:::ep -->|G1 validateConfig| RE
  RE -->|run AgentRequest| DA
  DA["DynamicAgent<br/>Agent"]:::agent -->|AgentResult| RE
  RE -->|G2 validateOutput| RE
  RE -->|submit / recordCompletion / recordBlock| ENT
  ENT["RequestEntity<br/>EventSourcedEntity"]:::ese -.->|events| VIEW
  VIEW["RequestsView<br/>View"]:::view -.->|SSE| UI
  UI([Browser SSE]):::ext
  APP["AppEndpoint<br/>HttpEndpoint"]:::ep --> UI
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Client
  participant API as RouteEndpoint
  participant DA as DynamicAgent
  participant ENT as RequestEntity
  participant VIEW as RequestsView

  Client->>API: POST /api/requests { route, text, targetLanguage? }
  Note over API: G1 validateConfig (before-agent-invocation)
  API->>ENT: submit(record)
  ENT-->>VIEW: RequestSubmitted event
  VIEW-->>Client: SSE status=SUBMITTED
  API->>DA: run(AgentRequest) with per-request AgentSetup
  DA-->>API: AgentResult(output)
  Note over API: G2 validateOutput (before-agent-response)
  API->>ENT: recordCompletion(output)
  ENT-->>VIEW: RequestCompleted event
  VIEW-->>Client: SSE status=COMPLETED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED: submit
  SUBMITTED --> COMPLETED: recordCompletion
  SUBMITTED --> BLOCKED: recordBlock (G1 or G2)
  COMPLETED --> [*]
  BLOCKED --> [*]
```

## Entity model

```mermaid
erDiagram
  REQUEST ||--o{ REQUESTSUBMITTED : emits
  REQUEST ||--o{ REQUESTCOMPLETED : emits
  REQUEST ||--o{ REQUESTBLOCKED : emits
  REQUESTSVIEW ||--|| REQUEST : projects
  REQUEST {
    string id PK
    Route route
    string inputText
    RequestStatus status
    Instant submittedAt
    string targetLanguage
    string output
    Instant completedAt
    string blockedReason
    Instant blockedAt
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| DynamicAgent | `application/DynamicAgent.java` |
| RequestEntity | `application/RequestEntity.java` |
| RequestsView | `application/RequestsView.java` |
| RouteEndpoint | `api/RouteEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| RequestSimulator | `application/RequestSimulator.java` |
| RequestRecord / AgentRequest / AgentResult / events | `domain/` |

## Concurrency notes

- The endpoint step that calls `DynamicAgent` uses an explicit 60-second timeout (Lesson 4); LLM calls exceed the 5-second default.
- Idempotency: each request gets a fresh UUID `requestId` at submit time; the entity rejects a second `submit` for the same id.
- No saga or compensation: the flow is single-agent and synchronous. A guardrail rejection (G1 or G2) is a terminal `BLOCKED` transition, not a rollback — nothing downstream has been committed when G1 fires, and only the entity record is written when G2 fires.
- The simulator and the live UI submit through the same endpoint path, so guardrails apply identically to both.
