# Architecture

The system runs a fixed three-stage pipeline per candidate, governed by four
controls. The four mermaid diagrams below are the source the Architecture tab
renders.

## Component graph

A requisition enters through `RecruitmentEndpoint` or is dripped by
`RequisitionSimulator` into `RequisitionQueue`. `RequisitionConsumer` reads each
queued requisition, checks `SystemControlEntity` for a halt, and starts one
`RecruitmentWorkflow` per candidate slot. The workflow drives `SourcingAgent`,
`ProfileSanitizer`, `ScreeningAgent`, and `MatchingAgent` in order, writing each
transition into `CandidateEntity`. `CandidatesView` projects those events for the
endpoint's list and SSE stream, and `DeployerMonitor` reads the same view to
aggregate outcome metrics.

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

  SIM[RequisitionSimulator]:::ta -.-> RQ[RequisitionQueue]:::ese
  EP[RecruitmentEndpoint]:::ep --> RQ
  EP --> SC[SystemControlEntity]:::kve
  RQ -.-> CONS[RequisitionConsumer]:::cons
  CONS --> SC
  CONS --> WF[RecruitmentWorkflow]:::wf
  WF --> SC
  WF --> SA[SourcingAgent]:::agent
  WF --> SAN[ProfileSanitizer]:::ta
  WF --> SCR[ScreeningAgent]:::agent
  WF --> MA[MatchingAgent]:::agent
  WF --> CE[CandidateEntity]:::ese
  CE -.-> CV[CandidatesView]:::view
  MON[DeployerMonitor]:::ta -.-> CV
  EP --> CV
```

## Interaction sequence

The primary journey: a submitted requisition becomes a sourced, sanitized,
screened, and matched candidate. The guardrail and halt branches end the run
early.

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

`CandidateStatus` lifecycle. BLOCKED, REJECTED, MATCHED, and HALTED are terminal.

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
