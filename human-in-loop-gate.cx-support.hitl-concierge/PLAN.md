# PLAN — hitl-concierge

Architectural sketch for HITL Concierge. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0e1e2a", "primaryTextColor": "#7EC8E3", "primaryBorderColor": "#7EC8E3", "lineColor": "#888", "secondaryColor": "#1c1330", "tertiaryColor": "#1f1900", "background": "#0d0d0d", "mainBkg": "#0d0d0d", "nodeBorder": "#555", "clusterBkg": "#111", "titleColor": "#ffffff", "edgeLabelBackground": "#0d0d0d", "transitionLabelColor": "#cccccc"}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ConciergeEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ConciergeWorkflow]:::wf
  TA[TriageAgent]:::agent
  FA[FulfillmentAgent]:::agent
  CE[CaseEntity]:::ese
  CV[CasesView]:::view

  EP -->|cases| WF
  WF -->|triage task| TA
  WF -->|fulfil task| FA
  TA -->|recordTriage| CE
  FA -->|recordFulfilment| CE
  EP -->|approve / decline| CE
  WF -->|poll status| CE
  CE -.->|events| CV
  EP -->|getAllCases / SSE| CV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0e1e2a", "primaryTextColor": "#e0e0e0", "primaryBorderColor": "#555", "lineColor": "#888", "secondaryColor": "#1c1330", "background": "#0d0d0d", "mainBkg": "#0d0d0d", "transitionLabelColor": "#cccccc"}}}%%
sequenceDiagram
  autonumber
  actor Specialist
  participant EP as ConciergeEndpoint
  participant WF as ConciergeWorkflow
  participant TA as TriageAgent
  participant CE as CaseEntity
  participant FA as FulfillmentAgent

  Specialist->>EP: POST /api/cases {customerRequest}
  EP->>WF: start(caseId, customerRequest)
  WF->>TA: runSingleTask(TRIAGE)
  TA-->>WF: TriageResult{summary, proposedResolution, urgency}
  WF->>CE: recordTriage -> TRIAGED
  Note over WF,CE: await-approval task paused; workflow polls status every 5s
  Specialist->>EP: POST /api/cases/{id}/approve
  EP->>CE: approve -> APPROVED
  WF->>CE: getCase -> APPROVED
  WF->>FA: runSingleTask(FULFIL) [guard: status == APPROVED]
  FA-->>WF: FulfilledCase{confirmationId, fulfilledAt}
  WF->>CE: recordFulfilment -> FULFILLED
```

## State machine

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1f1900", "primaryTextColor": "#ffffff", "primaryBorderColor": "#F5C518", "lineColor": "#888", "background": "#0d0d0d", "mainBkg": "#0d0d0d", "transitionLabelColor": "#cccccc", "stateLabelColor": "#ffffff"}}}%%
stateDiagram-v2
  [*] --> TRIAGED: CaseTriaged
  TRIAGED --> APPROVED: CaseApproved
  TRIAGED --> DECLINED: CaseDeclined
  APPROVED --> FULFILLED: CaseFulfilled
  DECLINED --> [*]
  FULFILLED --> [*]
```

## Entity model

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1f1900", "primaryTextColor": "#F5C518", "primaryBorderColor": "#F5C518", "lineColor": "#888", "background": "#0d0d0d", "mainBkg": "#0d0d0d"}}}%%
erDiagram
  CASE ||--o{ CASE_EVENT : emits
  CASE {
    string id
    string customerRequest
    string status
    string triageSummary
    string proposedResolution
    string urgency
    string confirmationId
  }
  CASE_EVENT {
    string type
    string occurredAt
  }
  CASES_VIEW {
    string id
    string status
    string urgency
  }
  CASE ||--|| CASES_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| TriageAgent | `application/TriageAgent.java` |
| FulfillmentAgent | `application/FulfillmentAgent.java` |
| ConciergeWorkflow | `application/ConciergeWorkflow.java` |
| ConciergeTasks | `application/ConciergeTasks.java` |
| CaseEntity | `application/CaseEntity.java` |
| CasesView | `application/CasesView.java` |
| ConciergeEndpoint | `api/ConciergeEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Case / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `triageStep` and `fulfilStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry forever (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `CaseEntity.getCase`, and on `TRIAGED` self-schedules a 5-second resume timer until the specialist transitions the status.
- **Idempotency.** `caseId` is the workflow id and the entity id; re-delivery of `recordTriage` / `recordFulfilment` is absorbed by event-applier checks on current status.
- **Fulfilment guard.** Before the fulfilment tool runs, the before-tool-call guardrail re-reads `CaseEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because fulfilment is the terminal write.
