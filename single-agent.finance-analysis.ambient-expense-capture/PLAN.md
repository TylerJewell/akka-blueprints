# PLAN — ambient-expense-agent

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ExpenseEndpoint]:::ep
  Entity[ExpenseEntity]:::ese
  Sanitizer[ReceiptSanitizer]:::cons
  WF[ExpenseWorkflow]:::wf
  Agent[ExpenseCaptureAgent]:::agent
  Guard[SubmissionGuardrail]:::guard
  View[ExpenseView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|ReceiptSubmitted| Sanitizer
  Sanitizer -->|attachSanitized| Entity
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| Entity
  WF -->|captureStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|pass or reject| Agent
  Agent -->|ExpenseReport| WF
  WF -->|recordReport| Entity
  WF -->|submitStep| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ExpenseEndpoint
  participant E as ExpenseEntity
  participant S as ReceiptSanitizer
  participant W as ExpenseWorkflow
  participant A as ExpenseCaptureAgent
  participant G as SubmissionGuardrail

  U->>API: POST /api/expenses
  API->>E: submit(request)
  E-->>API: { expenseId }
  E-.->>S: ReceiptSubmitted
  S->>S: redact PII
  S->>E: attachSanitized
  S->>W: start(expenseId)
  W->>E: poll getSubmission
  E-->>W: sanitized.isPresent()
  W->>E: markCapturing
  W->>A: runSingleTask(taxonomy + attachment)
  A->>G: before-tool-call(line-item submission)
  G-->>A: pass
  A-->>W: ExpenseReport
  W->>E: recordReport(report)
  W->>E: markSystemSubmitted
  E-.->>U: SSE event(SUBMITTED_TO_SYSTEM)
```

## State machine — `ExpenseEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> SANITIZED: ReceiptSanitized
  SANITIZED --> CAPTURING: CaptureStarted
  CAPTURING --> REPORT_READY: ReportReady
  REPORT_READY --> SUBMITTED_TO_SYSTEM: SystemSubmitted
  CAPTURING --> FAILED: SubmissionFailed (agent error)
  SUBMITTED --> FAILED: SubmissionFailed (sanitizer error)
  SUBMITTED_TO_SYSTEM --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ExpenseEntity ||--o{ ReceiptSubmitted : emits
  ExpenseEntity ||--o{ ReceiptSanitized : emits
  ExpenseEntity ||--o{ CaptureStarted : emits
  ExpenseEntity ||--o{ ReportReady : emits
  ExpenseEntity ||--o{ SystemSubmitted : emits
  ExpenseEntity ||--o{ SubmissionFailed : emits
  ExpenseView }o--|| ExpenseEntity : projects
  ReceiptSanitizer }o--|| ExpenseEntity : subscribes
  ExpenseWorkflow }o--|| ExpenseEntity : reads-and-writes
  ExpenseCaptureAgent ||--o{ ExpenseReport : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ExpenseEndpoint` | `api/ExpenseEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ExpenseEntity` | `application/ExpenseEntity.java` (state in `domain/ExpenseSubmission.java`, events in `domain/ExpenseEvent.java`) |
| `ReceiptSanitizer` | `application/ReceiptSanitizer.java` |
| `ExpenseWorkflow` | `application/ExpenseWorkflow.java` |
| `ExpenseCaptureAgent` | `application/ExpenseCaptureAgent.java` (tasks in `application/ExpenseTasks.java`) |
| `SubmissionGuardrail` | `application/SubmissionGuardrail.java` |
| `ExpenseView` | `application/ExpenseView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `captureStep` 60 s, `submitStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ExpenseWorkflow::error)`. The 60 s on `captureStep` accommodates LLM latency and multi-item extraction (Lesson 4).
- **Idempotency**: every workflow uses `"expense-" + expenseId` as the workflow id; the `ReceiptSanitizer` Consumer is allowed to redeliver `ReceiptSubmitted` events because `ExpenseEntity.attachSanitized` is event-version-guarded — a second sanitize attempt on an already-sanitized submission is a no-op.
- **One agent per submission**: the AutonomousAgent instance id is `"capture-" + expenseId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4 — sufficient for a multi-item receipt where several items may need reclassification.
- **Guardrail-driven correction**: when `SubmissionGuardrail` rejects a tool call for a single line item, the rejection is returned as a structured error to the agent loop. The agent can correct the item's category or amount, or mark it `BLOCKED` and proceed to the next item. A full budget exhaustion results in the workflow `captureStep` failing over to `error`.
- **No saga / no compensation**: every step is pure read, append-only event write, or a single-task agent call. There is nothing external to roll back in the simulated expense-system path.
