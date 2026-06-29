# PLAN — kyc-screener

Architectural sketch for KYC Screener. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ScreeningEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ScreeningWorkflow]:::wf
  SA[ScreenerAgent]:::agent
  VA[VerdictAgent]:::agent
  CE[CaseEntity]:::ese
  CV[CasesView]:::view

  EP -->|screening-request| WF
  WF -->|screen task| SA
  WF -->|close task| VA
  SA -->|recordScreening| CE
  VA -->|recordClose| CE
  EP -->|approve / reject| CE
  WF -->|poll status| CE
  CE -.->|events| CV
  EP -->|getAllCases / SSE| CV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Officer
  participant EP as ScreeningEndpoint
  participant WF as ScreeningWorkflow
  participant SA as ScreenerAgent
  participant CE as CaseEntity
  participant VA as VerdictAgent

  Officer->>EP: POST /api/screening-request {entityId, documents}
  EP->>WF: start(caseId, entityId, documents)
  WF->>SA: runSingleTask(SCREEN)
  SA-->>WF: ScreeningResult{entityId, findings, recommendation}
  WF->>CE: recordScreening -> SCREENED
  Note over WF,CE: await-approval task paused; workflow polls status every 5s
  Officer->>EP: POST /api/cases/{id}/approve
  EP->>CE: approve -> OFFICER_APPROVED
  WF->>CE: getCase -> OFFICER_APPROVED
  WF->>VA: runSingleTask(CLOSE) [guard: status == OFFICER_APPROVED]
  VA-->>WF: ClosedCase{caseId, disposition, closedAt}
  WF->>CE: recordClose -> CLOSED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> SCREENED: CaseScreened
  SCREENED --> OFFICER_APPROVED: CaseOfficerApproved
  SCREENED --> REJECTED: CaseRejected
  OFFICER_APPROVED --> CLOSED: CaseClosed
  REJECTED --> [*]
  CLOSED --> [*]
```

## Entity model

```mermaid
erDiagram
  CASE ||--o{ CASE_EVENT : emits
  CASE {
    string id
    string entityId
    string status
    string findings
    string recommendation
    string disposition
  }
  CASE_EVENT {
    string type
    string occurredAt
  }
  CASES_VIEW {
    string id
    string status
  }
  CASE ||--|| CASES_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ScreenerAgent | `application/ScreenerAgent.java` |
| VerdictAgent | `application/VerdictAgent.java` |
| ScreeningWorkflow | `application/ScreeningWorkflow.java` |
| ScreeningTasks | `application/ScreeningTasks.java` |
| CaseEntity | `application/CaseEntity.java` |
| CasesView | `application/CasesView.java` |
| ScreeningEndpoint | `api/ScreeningEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Case / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `screenStep` and `closeStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry forever (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `CaseEntity.getCase`, and on `SCREENED` self-schedules a 5-second resume timer until the compliance officer transitions the status.
- **Idempotency.** `caseId` is the workflow id and the entity id; re-delivery of `recordScreening` / `recordClose` is absorbed by event-applier checks on current status.
- **Close guard.** Before the close tool runs, the before-agent-response guardrail verifies that `ScreeningResult.findings` cites at least one source document reference; if absent the result is rejected and the agent retries. The before-tool-call check on `VerdictAgent` re-reads `CaseEntity.status`; if it is not `OFFICER_APPROVED`, the close call is blocked.
