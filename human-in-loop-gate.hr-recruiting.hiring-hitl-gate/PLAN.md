# PLAN — hiring-hitl-gate

Architectural sketch for Human-in-the-Loop Hiring. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[HiringEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[HiringWorkflow]:::wf
  HDP[HiringDecisionProposer]:::agent
  MP[MeetingProposer]:::agent
  AE[ApplicationEntity]:::ese
  AV[ApplicationsView]:::view

  EP -->|hiring-request| WF
  WF -->|propose task| HDP
  WF -->|meeting task| MP
  HDP -->|recordProposals| AE
  MP -->|recordProposals| AE
  EP -->|approve / decline| AE
  WF -->|poll status| AE
  WF -->|recordOutcome| AE
  AE -.->|events| AV
  EP -->|getAllApplications / SSE| AV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Recruiter
  participant EP as HiringEndpoint
  participant WF as HiringWorkflow
  participant HDP as HiringDecisionProposer
  participant MP as MeetingProposer
  participant AE as ApplicationEntity
  participant DRS as DecisionsReachedService

  Recruiter->>EP: POST /api/hiring-request {candidateName, role, ...}
  EP->>WF: start(applicationId, candidate)
  WF->>HDP: runSingleTask(HIRE_DECISION)
  HDP-->>WF: HiringProposal{recommendation, rationale}
  WF->>MP: runSingleTask(MEETING)
  MP-->>WF: MeetingProposal{subject, body, suggestedSlots}
  WF->>AE: recordProposals -> PROPOSED
  Note over WF,AE: await-validator task paused; workflow polls status every 5s
  Recruiter->>EP: POST /api/applications/{id}/approve
  EP->>AE: approve -> APPROVED
  WF->>AE: getApplication -> APPROVED
  WF->>DRS: recordOutcome [guard: status == APPROVED]
  DRS-->>WF: RecordedOutcome{decidedAt}
  WF->>AE: recordOutcome -> DECIDED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PROPOSED: ApplicationProposed
  PROPOSED --> APPROVED: ApplicationApproved
  PROPOSED --> DECLINED: ApplicationDeclined
  APPROVED --> DECIDED: OutcomeRecorded
  DECLINED --> [*]
  DECIDED --> [*]
```

## Entity model

```mermaid
erDiagram
  APPLICATION ||--o{ APPLICATION_EVENT : emits
  APPLICATION {
    string id
    string candidateName
    string role
    string status
    string hiringRecommendation
    string meetingSubject
    string decidedAt
  }
  APPLICATION_EVENT {
    string type
    string occurredAt
  }
  APPLICATIONS_VIEW {
    string id
    string status
    string hiringRecommendation
  }
  APPLICATION ||--|| APPLICATIONS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| HiringDecisionProposer | `application/HiringDecisionProposer.java` |
| MeetingProposer | `application/MeetingProposer.java` |
| HiringWorkflow | `application/HiringWorkflow.java` |
| HiringTasks | `application/HiringTasks.java` |
| ApplicationEntity | `application/ApplicationEntity.java` |
| ApplicationsView | `application/ApplicationsView.java` |
| HiringEndpoint | `api/HiringEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Application / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `proposeStep` calls both agents; it sets `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would expire before the agents return (Lesson 4).
- **Await-validator task.** The workflow does not block a thread; `awaitValidatorStep` reads `ApplicationEntity.getApplication`, and on `PROPOSED` self-schedules a 5-second resume timer until the hiring manager transitions the status.
- **Idempotency.** `applicationId` is the workflow id and entity id; re-delivery of `recordProposals` / `recordOutcome` is absorbed by event-applier checks on current status.
- **Outcome guard.** Before the outcome-recording tool runs, the before-tool-call guardrail re-reads `ApplicationEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because outcome recording is the terminal write.
- **Special-category sanitizer.** The before-agent-response sanitizer on `HiringDecisionProposer` removes inferred protected-attribute signals before the `HiringProposal` is written to `ApplicationEntity`.
