# PLAN — steered-renewal-agent

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

  API[RenewalEndpoint]:::ep
  Entity[LoanEntity]:::ese
  WF[RenewalWorkflow]:::wf
  Agent[RenewalAgent]:::agent
  Guard[PolicyEnforcer]:::guard
  View[RenewalView]:::view
  App[AppEndpoint]:::ep

  API -->|requestRenewal| Entity
  API -->|start workflow| WF
  WF -->|enrichStep load data| Entity
  WF -->|attachEnrichedData| Entity
  WF -->|decideStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|reject or pass| Agent
  Agent -->|RenewalDecision| WF
  WF -->|recordDecision| Entity
  WF -->|notifyStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as RenewalEndpoint
  participant E as LoanEntity
  participant W as RenewalWorkflow
  participant A as RenewalAgent
  participant G as PolicyEnforcer

  U->>API: POST /api/renewals
  API->>E: requestRenewal(patronId, loanId)
  E-->>API: { renewalId }
  API->>W: start(renewalId)
  W->>W: enrichStep (load patron + loan)
  W->>E: attachEnrichedData(patron, loan)
  W->>E: markDeciding
  W->>A: runSingleTask(patron + loan context)
  A->>G: before-agent-response(candidate)
  G-->>A: accept (policy satisfied)
  A-->>W: RenewalDecision(APPROVED)
  W->>E: recordDecision(decision)
  W->>E: recordNotification(notification)
  E-.->>U: SSE event(COMPLETED)
```

## State machine — `LoanEntity`

```mermaid
stateDiagram-v2
  [*] --> REQUESTED
  REQUESTED --> ENRICHED: LoanEnriched
  ENRICHED --> DECIDING: DecisionStarted
  DECIDING --> DECISION_RECORDED: DecisionRecorded
  DECISION_RECORDED --> COMPLETED: NotificationSent
  DECIDING --> FAILED: RenewalFailed (agent error)
  REQUESTED --> FAILED: RenewalFailed (enrich error)
  COMPLETED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  LoanEntity ||--o{ RenewalRequested : emits
  LoanEntity ||--o{ LoanEnriched : emits
  LoanEntity ||--o{ DecisionStarted : emits
  LoanEntity ||--o{ DecisionRecorded : emits
  LoanEntity ||--o{ NotificationSent : emits
  LoanEntity ||--o{ RenewalFailed : emits
  RenewalView }o--|| LoanEntity : projects
  RenewalWorkflow }o--|| LoanEntity : reads-and-writes
  RenewalAgent ||--o{ RenewalDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RenewalEndpoint` | `api/RenewalEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `LoanEntity` | `application/LoanEntity.java` (state in `domain/Renewal.java`, events in `domain/RenewalEvent.java`) |
| `RenewalWorkflow` | `application/RenewalWorkflow.java` |
| `RenewalAgent` | `application/RenewalAgent.java` (tasks in `application/RenewalTasks.java`) |
| `PolicyEnforcer` | `application/PolicyEnforcer.java` |
| `RenewalView` | `application/RenewalView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `enrichStep` 10 s, `decideStep` 60 s, `notifyStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(RenewalWorkflow::error)`. The 60 s on `decideStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"renewal-" + renewalId` as the workflow id; `RenewalEndpoint` mints the `renewalId` before calling `LoanEntity.requestRenewal`, so a duplicate POST with the same IDs starts a new renewal with a new ID — no ambiguity.
- **One agent per renewal**: the AutonomousAgent instance id is `"renewal-agent-" + renewalId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `PolicyEnforcer` rejects a candidate decision, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail policy checks, the workflow's `decideStep` fails over to `error` and the entity transitions to `FAILED`.
- **Notification is synchronous and deterministic**: `notifyStep` builds a `NotificationRecord` in-process. No LLM call, no external service. This is a deliberate single-agent guarantee.
- **No external rollback**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to compensate.
