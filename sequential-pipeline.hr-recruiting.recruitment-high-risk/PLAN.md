# PLAN — recruitment-high-risk

Architectural sketch. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef kve fill:#201500,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  SIM[RequisitionSimulator]:::ta -.->|drips| RQ[RequisitionQueue]:::ese
  EP[RecruitmentEndpoint]:::ep -->|enqueue| RQ
  EP -->|halt/resume| SC[SystemControlEntity]:::kve
  RQ -.->|events| CONS[RequisitionConsumer]:::cons
  CONS -->|checks| SC
  CONS -->|starts| WF[RecruitmentWorkflow]:::wf
  WF -->|checks| SC
  WF -->|source-tool guarded| SA[SourcingAgent]:::agent
  WF -->|sanitize| SAN[ProfileSanitizer]:::ta
  WF -->|screen| SCR[ScreeningAgent]:::agent
  WF -->|match| MA[MatchingAgent]:::agent
  WF -->|writes| CE[CandidateEntity]:::ese
  CE -.->|events| CV[CandidatesView]:::view
  MON[DeployerMonitor]:::ta -.->|reads| CV
  EP -->|reads| CV
  EP -->|reads| MON
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions; dotted ticks are scheduled actions.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant EP as RecruitmentEndpoint
  participant RQ as RequisitionQueue
  participant C as RequisitionConsumer
  participant SC as SystemControlEntity
  participant WF as RecruitmentWorkflow
  participant SA as SourcingAgent
  participant CE as CandidateEntity
  U->>EP: POST /api/requisitions
  EP->>RQ: enqueue(Requisition)
  RQ-->>C: RequisitionQueued
  C->>SC: status()
  Note over C,SC: skip if halted
  C->>WF: start (per candidate slot)
  WF->>SC: status()
  WF->>SA: source-candidates (before-tool-call guardrail)
  Note over WF,SA: non-allowlisted domain -> SourcingBlocked, end
  SA-->>WF: SourcedProfile
  WF->>WF: sanitizeStep (redact special-category)
  WF->>CE: recordSanitized
  WF->>CE: recordScreened / recordMatched
  CE-->>EP: SSE update
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SOURCED
  SOURCED --> SANITIZED
  SOURCED --> BLOCKED: non-allowlisted domain
  SOURCED --> HALTED: operator halt
  SANITIZED --> SCREENED
  SCREENED --> MATCHED: pass
  SCREENED --> REJECTED: fail
  MATCHED --> [*]
  REJECTED --> [*]
  BLOCKED --> [*]
  HALTED --> [*]
```

## Entity model

```mermaid
erDiagram
  REQUISITION_QUEUE ||--o{ CANDIDATE : "spawns"
  CANDIDATE ||--|| CANDIDATES_VIEW : "projects"
  SYSTEM_CONTROL ||--o{ CANDIDATE : "gates"
  CANDIDATE {
    string id
    string status
    double matchScore
    string redactedCategories
    string blockedDomain
  }
  REQUISITION_QUEUE {
    string roleTitle
    int candidateCount
  }
  SYSTEM_CONTROL {
    boolean halted
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| `SourcingAgent` | `application/SourcingAgent.java` |
| `ScreeningAgent` | `application/ScreeningAgent.java` |
| `MatchingAgent` | `application/MatchingAgent.java` |
| `RecruitmentTasks` | `application/RecruitmentTasks.java` |
| `ProfileSanitizer` | `application/ProfileSanitizer.java` |
| `RecruitmentWorkflow` | `application/RecruitmentWorkflow.java` |
| `CandidateEntity` | `application/CandidateEntity.java` |
| `RequisitionQueue` | `application/RequisitionQueue.java` |
| `SystemControlEntity` | `application/SystemControlEntity.java` |
| `CandidatesView` | `application/CandidatesView.java` |
| `RequisitionConsumer` | `application/RequisitionConsumer.java` |
| `RequisitionSimulator` | `application/RequisitionSimulator.java` |
| `DeployerMonitor` | `application/DeployerMonitor.java` |
| `RecruitmentEndpoint` | `api/RecruitmentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |

## Concurrency notes

- Workflow step timeouts: `sourceStep`, `screenStep`, `matchStep` each `ofSeconds(60)` (Lesson 4). `sanitizeStep` is local (no LLM) and uses the default timeout. `WorkflowSettings` is nested in `Workflow` — no import (Lesson 5).
- Idempotency: each candidate workflow keys on a fresh UUID; `CandidateEntity` event-appliers are idempotent on replay.
- Compensation: `defaultStepRecovery(maxRetries(2).failoverTo(error))`; the error step writes `CandidateRejected` with the failure reason so no candidate is left mid-pipeline.
- Halt gate is checked at consumer entry and at the workflow's first step; an in-flight workflow that sees a halt writes `EvaluationHalted` and ends rather than completing.
