# PLAN — support-hitl-durable

Architectural sketch for Support Agent with HITL. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[SupportEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[SupportWorkflow]:::wf
  TA[TriageAgent]:::agent
  RA[ResolutionAgent]:::agent
  TE[TicketEntity]:::ese
  TV[TicketsView]:::view

  EP -->|submit-ticket| WF
  WF -->|triage task| TA
  WF -->|resolve task| RA
  TA -->|recordTriage| TE
  RA -->|recordResolution| TE
  EP -->|approve / escalate| TE
  WF -->|poll status| TE
  TE -.->|events| TV
  EP -->|getAllTickets / SSE| TV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Operator
  participant EP as SupportEndpoint
  participant WF as SupportWorkflow
  participant TA as TriageAgent
  participant TE as TicketEntity
  participant RA as ResolutionAgent

  Operator->>EP: POST /api/tickets {subject, body}
  EP->>WF: start(ticketId, subject, body)
  WF->>TA: runSingleTask(TRIAGE)
  TA-->>WF: TicketTriage{category, priority, summary}
  WF->>TE: recordTriage -> TRIAGED
  Note over WF,TE: await-decision task paused; workflow polls status every 5s
  Operator->>EP: POST /api/tickets/{id}/approve
  EP->>TE: approve -> APPROVED
  WF->>TE: getTicket -> APPROVED
  WF->>RA: runSingleTask(RESOLVE) [guard: status == APPROVED]
  RA-->>WF: TicketResolution{response, resolvedAt}
  WF->>TE: recordResolution -> RESOLVED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> TRIAGED: TicketTriaged
  TRIAGED --> APPROVED: TicketApproved
  TRIAGED --> ESCALATED: TicketEscalated
  APPROVED --> RESOLVED: TicketResolved
  ESCALATED --> [*]
  RESOLVED --> [*]
```

## Entity model

```mermaid
erDiagram
  TICKET ||--o{ TICKET_EVENT : emits
  TICKET {
    string id
    string subject
    string status
    string triageCategory
    string triageSummary
    string resolution
  }
  TICKET_EVENT {
    string type
    string occurredAt
  }
  TICKETS_VIEW {
    string id
    string status
  }
  TICKET ||--|| TICKETS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| TriageAgent | `application/TriageAgent.java` |
| ResolutionAgent | `application/ResolutionAgent.java` |
| SupportWorkflow | `application/SupportWorkflow.java` |
| SupportTasks | `application/SupportTasks.java` |
| TicketEntity | `application/TicketEntity.java` |
| TicketsView | `application/TicketsView.java` |
| SupportEndpoint | `api/SupportEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Ticket / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `triageStep` and `resolveStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would expire and retry before the agent can respond (Lesson 4).
- **Await-decision task.** The workflow does not block a thread; `awaitDecisionStep` reads `TicketEntity.getTicket`, and on `TRIAGED` self-schedules a 5-second resume timer until the operator transitions the status.
- **Idempotency.** `ticketId` is the workflow id and the entity id; re-delivery of `recordTriage` / `recordResolution` is absorbed by event-applier checks on current status.
- **Resolution guard.** Before the resolution tool runs, the before-agent-response guardrail checks that the output meets content policy; if it does not, the response is rejected before it is persisted. The before-tool-call path also re-reads `TicketEntity.status`; if it is not `APPROVED`, the call is blocked with no compensation path required.
