# PLAN — deep-research-hitl

Architectural sketch for HITL Deep Research. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0e1e2a", "primaryTextColor": "#7EC8E3", "primaryBorderColor": "#7EC8E3", "lineColor": "#cccccc", "secondaryColor": "#1c1330", "tertiaryColor": "#1f1900"}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[ResearchEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[ResearchWorkflow]:::wf
  PLA[PlannerAgent]:::agent
  RA[ResearchAgent]:::agent
  SA[SynthesisAgent]:::agent
  RE[ResearchEntity]:::ese
  RV[ReportsView]:::view

  EP -->|research-request| WF
  WF -->|plan task| PLA
  WF -->|investigate tasks| RA
  WF -->|synthesise task| SA
  PLA -->|recordPlan| RE
  RA -->|recordFindings| RE
  SA -->|recordDraft| RE
  EP -->|approve / reject| RE
  WF -->|poll status| RE
  RE -.->|events| RV
  EP -->|getAllReports / SSE| RV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Researcher
  participant EP as ResearchEndpoint
  participant WF as ResearchWorkflow
  participant PLA as PlannerAgent
  participant RA as ResearchAgent
  participant SA as SynthesisAgent
  participant RE as ResearchEntity

  Researcher->>EP: POST /api/research-request {query}
  EP->>WF: start(reportId, query)
  WF->>PLA: runSingleTask(PLAN)
  PLA-->>WF: ResearchPlan{queryId, subTopics}
  WF->>RE: recordPlan -> PLANNING→INVESTIGATING
  loop for each subTopic
    WF->>RA: runSingleTask(INVESTIGATE, subTopic)
    RA-->>WF: SubTopicFindings{subTopic, summary, sources}
    WF->>RE: recordFindings
  end
  WF->>SA: runSingleTask(SYNTHESISE)
  SA-->>WF: ReportDraft{title, body, sourcesUsed}
  WF->>RE: recordDraft -> AWAITING_REVIEW
  Note over WF,RE: awaitReviewStep polls every 5s
  Researcher->>EP: POST /api/reports/{id}/approve
  EP->>RE: approve -> APPROVED
  WF->>RE: getReport -> APPROVED
  WF->>RE: recordDelivery [guard: status == APPROVED]
  RE-->>RV: ReportDelivered -> DELIVERED
```

## State machine

```mermaid
%%{init: {"theme": "base", "themeVariables": {"transitionLabelColor": "#cccccc"}}}%%
stateDiagram-v2
  [*] --> PLANNING: QueryPlanned
  PLANNING --> INVESTIGATING: InvestigationStarted
  INVESTIGATING --> SYNTHESISING: SynthesisStarted
  SYNTHESISING --> AWAITING_REVIEW: ReportAwaitingReview
  AWAITING_REVIEW --> APPROVED: ReviewApproved
  AWAITING_REVIEW --> NEEDS_REVISION: ReviewRejected
  NEEDS_REVISION --> SYNTHESISING: re-synthesise
  APPROVED --> DELIVERED: ReportDelivered
  DELIVERED --> [*]
```

## Entity model

```mermaid
erDiagram
  REPORT ||--o{ REPORT_EVENT : emits
  REPORT {
    string id
    string query
    string status
    list subTopics
    list findingsSummaries
    string reportTitle
    string reportBody
    string deliveredAt
  }
  REPORT_EVENT {
    string type
    string occurredAt
  }
  REPORTS_VIEW {
    string id
    string status
    string reportTitle
  }
  REPORT ||--|| REPORTS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| PlannerAgent | `application/PlannerAgent.java` |
| ResearchAgent | `application/ResearchAgent.java` |
| SynthesisAgent | `application/SynthesisAgent.java` |
| ResearchWorkflow | `application/ResearchWorkflow.java` |
| ResearchTasks | `application/ResearchTasks.java` |
| ResearchEntity | `application/ResearchEntity.java` |
| ReportsView | `application/ReportsView.java` |
| ResearchEndpoint | `api/ResearchEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Report / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `planStep`, each `investigateStep` iteration, and `synthesiseStep` call agents; all set `stepTimeout(120s)` to absorb LLM latency across the multi-step fan-out. `deliverStep` sets `stepTimeout(60s)` (Lesson 4).
- **Await-review task.** The workflow does not block a thread; `awaitReviewStep` reads `ResearchEntity.getReport`, and on `AWAITING_REVIEW` self-schedules a 5-second resume timer until the human transitions the status. On `NEEDS_REVISION` the workflow loops back to `synthesiseStep`.
- **Idempotency.** `reportId` is the workflow id and the entity id; re-delivery of `recordPlan` / `recordFindings` / `recordDraft` / `recordDelivery` is absorbed by event-applier checks on current status.
- **Deliver guard.** Before the deliver tool runs, the before-tool-call guardrail re-reads `ResearchEntity.status`; if it is not `APPROVED`, the call is blocked and the report stays in `AWAITING_REVIEW`.
