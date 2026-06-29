# PLAN — order-processing

Architectural sketch for HITL Order Processing. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#0e1e2a', 'primaryTextColor': '#7EC8E3', 'primaryBorderColor': '#7EC8E3', 'lineColor': '#888', 'secondaryColor': '#1c1330', 'tertiaryColor': '#1f1900'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[OrderEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[OrderWorkflow]:::wf
  VA[ValidationAgent]:::agent
  FA[FulfillmentAgent]:::agent
  OE[OrderEntity]:::ese
  OV[OrdersView]:::view

  EP -->|order submission| WF
  WF -->|validate task| VA
  WF -->|fulfill task| FA
  VA -->|recordReview| OE
  FA -->|recordFulfillment| OE
  EP -->|approve / reject| OE
  WF -->|poll status| OE
  OE -.->|events| OV
  EP -->|getAllOrders / SSE| OV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Operator
  participant EP as OrderEndpoint
  participant WF as OrderWorkflow
  participant VA as ValidationAgent
  participant OE as OrderEntity
  participant FA as FulfillmentAgent

  Operator->>EP: POST /api/orders {customerId, lineItems}
  EP->>WF: start(orderId, orderPayload)
  WF->>VA: runSingleTask(VALIDATE)
  VA-->>WF: OrderReview{lineItems, estimatedTotal, riskFlags}
  WF->>OE: recordReview -> PENDING_APPROVAL
  Note over WF,OE: await-approval task paused; workflow polls status every 5s
  Operator->>EP: POST /api/orders/{id}/approve
  EP->>OE: approve -> APPROVED
  WF->>OE: getOrder -> APPROVED
  WF->>FA: runSingleTask(FULFILL) [guard: status == APPROVED]
  FA-->>WF: DispatchConfirmation{trackingId, dispatchedAt}
  WF->>OE: recordFulfillment -> FULFILLED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> PENDING_APPROVAL: OrderSubmitted
  PENDING_APPROVAL --> APPROVED: OrderApproved
  PENDING_APPROVAL --> REJECTED: OrderRejected
  APPROVED --> FULFILLED: OrderFulfilled
  REJECTED --> [*]
  FULFILLED --> [*]
```

## Entity model

```mermaid
erDiagram
  ORDER ||--o{ ORDER_EVENT : emits
  ORDER {
    string id
    string customerId
    string status
    string lineItemsSummary
    string estimatedTotal
    string trackingId
  }
  ORDER_EVENT {
    string type
    string occurredAt
  }
  ORDERS_VIEW {
    string id
    string status
  }
  ORDER ||--|| ORDERS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| ValidationAgent | `application/ValidationAgent.java` |
| FulfillmentAgent | `application/FulfillmentAgent.java` |
| OrderWorkflow | `application/OrderWorkflow.java` |
| OrderTasks | `application/OrderTasks.java` |
| OrderEntity | `application/OrderEntity.java` |
| OrdersView | `application/OrdersView.java` |
| OrderEndpoint | `api/OrderEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| Order / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `validateStep` and `fulfillStep` call agents; both set `stepTimeout(60s)` to absorb LLM latency. The default 5 s step timeout would retry forever (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `OrderEntity.getOrder`, and on `PENDING_APPROVAL` self-schedules a 5-second resume timer until the human transitions the status.
- **Idempotency.** `orderId` is the workflow id and the entity id; re-delivery of `recordReview` / `recordFulfillment` is absorbed by event-applier checks on current status.
- **Fulfillment guard.** Before the dispatch tool runs, the before-tool-call guardrail re-reads `OrderEntity.status`; if it is not `APPROVED`, the call is blocked. No compensation path is needed because dispatch is the terminal write.
