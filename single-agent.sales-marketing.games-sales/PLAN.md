# PLAN — games-sales

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

  API[SalesEndpoint]:::ep
  Entity[SessionEntity]:::ese
  WF[SessionWorkflow]:::wf
  Agent[SalesAssistantAgent]:::agent
  Guard[RecommendationGuardrail]:::guard
  View[RecommendationView]:::view
  App[AppEndpoint]:::ep

  API -->|startSession| Entity
  API -->|startFollowUp| Entity
  Entity -.->|SessionStarted| WF
  Entity -.->|FollowUpStarted| WF
  WF -->|queryStep runSingleTask| Agent
  WF -->|followUpStep runSingleTask + attachment| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|SalesRecommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|recordFollowUp| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as SalesEndpoint
  participant E as SessionEntity
  participant W as SessionWorkflow
  participant A as SalesAssistantAgent
  participant G as RecommendationGuardrail

  U->>API: POST /api/sessions
  API->>E: startSession(preferences)
  E-->>API: { sessionId }
  E-.->>W: SessionStarted
  W->>A: runSingleTask(query + preferences)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: SalesRecommendation
  W->>E: recordRecommendation(recommendation)
  E-.->>U: SSE event(RECOMMENDED)
  U->>API: POST /api/sessions/{id}/follow-up
  API->>E: startFollowUp(refinementText)
  E-.->>W: FollowUpStarted
  W->>A: runSingleTask(refinement + prior attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: SalesRecommendation
  W->>E: recordFollowUp(followUpRecommendation)
  E-.->>U: SSE event(FOLLOW_UP_RECOMMENDED)
```

## State machine — `SessionEntity`

```mermaid
stateDiagram-v2
  [*] --> QUERYING
  QUERYING --> RECOMMENDED: RecommendationRecorded
  RECOMMENDED --> FOLLOW_UP_QUERYING: FollowUpStarted
  FOLLOW_UP_QUERYING --> FOLLOW_UP_RECOMMENDED: FollowUpRecorded
  QUERYING --> FAILED: SessionFailed (agent error)
  FOLLOW_UP_QUERYING --> FAILED: SessionFailed (agent error)
  RECOMMENDED --> [*]
  FOLLOW_UP_RECOMMENDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  SessionEntity ||--o{ SessionStarted : emits
  SessionEntity ||--o{ RecommendationRecorded : emits
  SessionEntity ||--o{ FollowUpStarted : emits
  SessionEntity ||--o{ FollowUpRecorded : emits
  SessionEntity ||--o{ SessionFailed : emits
  RecommendationView }o--|| SessionEntity : projects
  SessionWorkflow }o--|| SessionEntity : reads-and-writes
  SalesAssistantAgent ||--o{ SalesRecommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SalesEndpoint` | `api/SalesEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SessionEntity` | `application/SessionEntity.java` (state in `domain/Session.java`, events in `domain/SessionEvent.java`) |
| `SessionWorkflow` | `application/SessionWorkflow.java` |
| `SalesAssistantAgent` | `application/SalesAssistantAgent.java` (tasks in `application/SessionTasks.java`) |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `RecommendationView` | `application/RecommendationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `queryStep` 60 s, `followUpStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SessionWorkflow::error)`. The 60 s on each agent step accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"session-" + sessionId + "-" + queryIndex` as the workflow id; re-delivering a `SessionStarted` event to an already-running workflow is a no-op because the workflow checks session status before starting.
- **One agent per session**: the AutonomousAgent instance id is `"agent-" + sessionId`, giving each session its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecommendationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. If all 3 iterations fail validation, the workflow's step fails over to `error` and the entity transitions to `FAILED`.
- **Follow-up context via attachment**: the prior `SalesRecommendation` is serialised to JSON and passed as a task attachment named `prior-recommendation.json`, never inlined into the instruction text.
- **No saga / no compensation**: each step either writes an append-only event or calls the agent. There is nothing external to roll back.
