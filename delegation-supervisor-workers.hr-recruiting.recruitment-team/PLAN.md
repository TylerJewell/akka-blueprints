# PLAN — `recruitment-team`

Architectural sketch for the delegation-supervisor-workers × hr-recruiting cell. All four mermaid diagrams render on the Architecture tab with the Lesson 24 CSS overrides applied.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  SIM[ApplicationSimulator]:::ta -.-> QUEUE[InboundApplicationQueue]:::ese
  EP[ScreeningEndpoint]:::ep --> QUEUE
  QUEUE -. events .-> CONS[ApplicationConsumer]:::cons
  CONS --> WF[ScreeningWorkflow]:::wf
  WF --> MATCH[MatchAgent]:::agent
  WF --> SCREEN[ScreenAgent]:::agent
  WF --> SUP[SupervisorAgent]:::agent
  WF --> CE[CandidateEntity]:::ese
  CE -. events .-> VIEW[CandidatesView]:::view
  EP --> CE
  EP --> VIEW
  DRIFT[FairnessDriftMonitor]:::ta -.-> VIEW
  DRIFT --> CE
  STUCK[StuckDecisionMonitor]:::ta -.-> VIEW
  STUCK --> CE
  APP[AppEndpoint]:::ep --> VIEW
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as Recruiter / Simulator
  participant Q as InboundApplicationQueue
  participant C as ApplicationConsumer
  participant W as ScreeningWorkflow
  participant S as ResumeSanitizer
  participant A as Worker agents
  participant E as CandidateEntity
  U->>Q: enqueueApplication(roleId, resume)
  Q-->>C: ApplicationQueued
  C->>W: start(candidateId)
  W->>S: redact protected attributes
  S-->>W: sanitized resume
  W->>E: recordSanitized
  W->>A: match, screen, supervise
  A-->>W: typed results
  W->>E: recordMatch, recordScreen, recordEval, requestDecision
  Note over W,E: status AWAITING_DECISION — human gate
  U->>E: approve / reject
  W->>E: finalize
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ResumeSanitized
  SANITIZED --> MATCHED: CandidateMatched
  MATCHED --> SCREENED: CandidateScreened
  SCREENED --> AWAITING_DECISION: DecisionRequested
  AWAITING_DECISION --> APPROVED: CandidateApproved
  AWAITING_DECISION --> REJECTED: CandidateRejected
  AWAITING_DECISION --> ESCALATED: CandidateEscalated
  APPROVED --> [*]
  REJECTED --> [*]
  ESCALATED --> [*]
```

## Entity model

```mermaid
erDiagram
  CANDIDATE_ENTITY ||--o{ CANDIDATE_EVENT : emits
  CANDIDATE_ENTITY ||--|| CANDIDATES_VIEW : projects
  INBOUND_QUEUE ||--o{ QUEUED_EVENT : emits
  CANDIDATE_ENTITY {
    string id
    string roleId
    string status
    int matchScore
    int evalScore
    bool driftFlagged
  }
  CANDIDATE_EVENT {
    string type
  }
  CANDIDATES_VIEW {
    string id
    string status
  }
  INBOUND_QUEUE {
    string roleId
    string resume
  }
```

## Component table

| Component | Path (generated) |
|---|---|
| `MatchAgent` | `application/MatchAgent.java` |
| `ScreenAgent` | `application/ScreenAgent.java` |
| `SupervisorAgent` | `application/SupervisorAgent.java` |
| `ScreeningTasks` | `application/ScreeningTasks.java` |
| `ResumeSanitizer` | `application/ResumeSanitizer.java` |
| `ScreeningWorkflow` | `application/ScreeningWorkflow.java` |
| `CandidateEntity` | `application/CandidateEntity.java` |
| `InboundApplicationQueue` | `application/InboundApplicationQueue.java` |
| `CandidatesView` | `application/CandidatesView.java` |
| `ApplicationConsumer` | `application/ApplicationConsumer.java` |
| `ApplicationSimulator` | `application/ApplicationSimulator.java` |
| `FairnessDriftMonitor` | `application/FairnessDriftMonitor.java` |
| `StuckDecisionMonitor` | `application/StuckDecisionMonitor.java` |
| `ScreeningEndpoint` | `api/ScreeningEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `Candidate`, records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `matchStep`, `screenStep`, `superviseStep` each set `stepTimeout(60s)` because they call agents; the default 5 s would fail (Lesson 4). `sanitizeStep` is in-process and uses the default.
- **Recovery.** `defaultStepRecovery(maxRetries(2).failoverTo(errorStep))`.
- **Idempotency.** The workflow id is the candidate id; `ApplicationConsumer` derives a deterministic candidate id per queued application so replays do not start duplicate workflows.
- **Await loop.** `awaitDecisionStep` self-schedules a 5 s resume timer while in `AWAITING_DECISION`; the human command and the `StuckDecisionMonitor` escalation are the two exits.
- **No saga.** No cross-service compensation; all state is in `CandidateEntity`.
