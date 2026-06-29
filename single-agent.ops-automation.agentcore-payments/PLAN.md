# PLAN: Agent-Initiated Payment Settlement

## 1. Component graph

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#00b4d8", "primaryTextColor": "#fff", "lineColor": "#0077b6", "background": "#f8f9fa"}}}%%
flowchart LR
    Client([HTTP Client])
    Endpoint[PaymentTaskEndpoint]
    Entity[(PaymentTaskEntity\nevent-sourced)]
    Workflow[PaymentApprovalWorkflow]
    Agent[/PaymentAgentSession\]
    View[[PaymentLedgerView\nkv-projection]]
    LLM{{LLM}}
    Reviewer([Human Reviewer])

    Client -->|POST /tasks| Endpoint
    Client -->|GET /tasks/id/stream| Endpoint
    Client -->|POST /tasks/id/approvals| Endpoint
    Endpoint -->|submitTask| Entity
    Endpoint -->|approvalDecision| Workflow
    Endpoint -->|subscribe| View
    Entity -->|startAgent| Agent
    Entity -->|startWorkflow| Workflow
    Agent -->|submitPaymentIntent| Entity
    Agent -->|getTaskStatus| Entity
    Agent <-->|tool calls| LLM
    Workflow -->|approvePayment / rejectPayment| Entity
    Reviewer -->|POST approvals| Endpoint
    Entity -->|events| View
```

---

## 2. Interaction sequence — happy path (J1)

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#00b4d8", "primaryTextColor": "#fff", "lineColor": "#0077b6", "background": "#f8f9fa"}}}%%
sequenceDiagram
    participant C as Client
    participant E as PaymentTaskEndpoint
    participant T as PaymentTaskEntity
    participant A as PaymentAgentSession
    participant W as PaymentApprovalWorkflow
    participant L as LLM
    participant R as Reviewer

    C->>E: POST /tasks {taskId, description, budget, threshold}
    E->>T: submitTask command
    T-->>T: emit TaskCreated
    T-->>A: start agent session
    A->>L: decompose task into payment intents
    L-->>A: paymentIntent #1 (amountCents < threshold)
    A->>T: submitPaymentIntent #1
    T-->>T: emit PaymentIntentSubmitted, PaymentSettled
    A->>L: next payment intent?
    L-->>A: paymentIntent #2 (amountCents >= threshold)
    A->>T: submitPaymentIntent #2
    T-->>T: emit PaymentIntentSubmitted, PaymentHeld
    T-->>W: start PaymentApprovalWorkflow(paymentId #2)
    W-->>R: notification: payment held for review
    C->>E: GET /tasks/task-001/stream (SSE)
    E-->>C: event: PaymentHeld
    R->>E: POST /tasks/task-001/approvals {APPROVE}
    E->>W: approvalDecision(APPROVE)
    W->>T: approvePayment(paymentId #2)
    T-->>T: emit PaymentApproved, PaymentSettled
    E-->>C: event: PaymentApproved, PaymentSettled
    A->>L: any remaining intents?
    L-->>A: no further payments needed
    A->>T: completeTask
    T-->>T: emit TaskCompleted
    E-->>C: event: TaskCompleted
```

---

## 3. State machine — PaymentTaskEntity

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#00b4d8", "primaryTextColor": "#fff", "lineColor": "#0077b6", "background": "#f8f9fa"}}}%%
stateDiagram-v2
    [*] --> PENDING : TaskCreated
    PENDING --> ACTIVE : AgentStarted
    ACTIVE --> ACTIVE : PaymentSettled\n(below threshold)
    ACTIVE --> AWAITING_APPROVAL : PaymentHeld\n(above threshold)
    AWAITING_APPROVAL --> ACTIVE : PaymentApproved\nor PaymentRejected
    ACTIVE --> BUDGET_EXCEEDED : BudgetCapReached
    AWAITING_APPROVAL --> BUDGET_EXCEEDED : BudgetCapReached\n(while approval pending)
    ACTIVE --> COMPLETED : TaskCompleted
    ACTIVE --> FAILED : TaskFailed
    BUDGET_EXCEEDED --> [*]
    COMPLETED --> [*]
    FAILED --> [*]

    classDef awaitingApproval fill:#ff9f1c,color:#000
    classDef budgetExceeded fill:#e63946,color:#fff
    classDef completed fill:#2a9d8f,color:#fff
    classDef active fill:#00b4d8,color:#fff

    class AWAITING_APPROVAL awaitingApproval
    class BUDGET_EXCEEDED budgetExceeded
    class COMPLETED completed
    class ACTIVE active
```

---

## 4. Entity model (ER)

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#00b4d8", "primaryTextColor": "#fff", "lineColor": "#0077b6", "background": "#f8f9fa"}}}%%
erDiagram
    PaymentTask {
        string taskId PK
        string description
        long budgetCents
        long approvalThresholdCents
        long accumulatedSpentCents
        TaskStatus status
        Instant createdAt
        Instant completedAt
    }
    PaymentRecord {
        string paymentId PK
        string taskId FK
        string targetApi
        long amountCents
        PaymentStatus status
        boolean approvalRequired
        string reviewedBy
        Instant settledAt
    }
    PaymentApprovalWorkflow {
        string workflowId PK
        string taskId FK
        string paymentId FK
        ApprovalStatus decision
        Instant decidedAt
        Instant expiresAt
    }
    PaymentLedgerRow {
        string taskId PK
        long budgetCents
        long spentCents
        long remainingCents
        TaskStatus status
        int totalPayments
        int pendingApprovals
    }

    PaymentTask ||--o{ PaymentRecord : "contains"
    PaymentTask ||--o| PaymentApprovalWorkflow : "may have active"
    PaymentTask ||--|| PaymentLedgerRow : "projected to"
```

---

## 5. Concurrency and timing notes

**Approval timeout:** `PaymentApprovalWorkflow` uses `workflow.timer().createSingleTimer(timeout, "approval-timeout")` (Lesson 4). On expiry, the workflow emits a soft rejection — the task does not fail; the agent can proceed with a lower-cost alternative.

**Idempotency:** Submitting the same `paymentId` twice to `PaymentTaskEntity` is a no-op after the first `PaymentIntentSubmitted` event (Lesson 10). The workflow is keyed on `paymentId`.

**Budget cap atomicity:** The cap check (`accumulatedSpentCents + amountCents >= budgetCents`) runs synchronously inside the entity command handler before emitting any event. There is no race window where two concurrent intents could both slip past the cap (Lesson 9).

**Agent-per-task isolation:** Each `PaymentAgentSession` is scoped to a single `taskId`. Concurrent tasks run independent agent sessions with no shared state.

**SSE backpressure:** The `PaymentLedgerView` is a key-value projection. The SSE subscription in `PaymentTaskEndpoint` uses `streamUpdatesForKey(taskId)` — no polling, no timer (Lesson 11).
