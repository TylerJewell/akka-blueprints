# PLAN — passport-renewal-coordinator

Architectural sketch consumed by `/akka:plan` (or skipped if `/akka:specify` covers it). Diagrams render on the generated system's Architecture tab.

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

  Coord[RenewalCoordinatorAgent]:::agent
  DocRev[DocumentReviewAgent]:::agent

  WF[RenewalWorkflow]:::wf
  App[ApplicationEntity]:::ese
  CW[CaseworkerControlEntity]:::ese
  Queue[ApplicationQueue]:::ese
  View[ApplicationView]:::view
  Consumer[ApplicationConsumer]:::cons
  Sim[RequestSimulator]:::ta
  Stale[StaleApplicationMonitor]:::ta
  API[ApplicationEndpoint]:::ep
  AppEP[AppEndpoint]:::ep

  API -->|submit| Queue
  API -->|approve/reject| CW
  Sim -.->|every 90s| Queue
  Queue -.->|subscribes| Consumer
  Consumer -->|start workflow| WF
  WF -->|PLAN_RENEWAL / DECIDE_NEXT / COMPOSE_ANSWER| Coord
  WF -->|REVIEW_DOCUMENT| DocRev
  WF -->|emit events| App
  WF -->|poll decision| CW
  App -.->|projects| View
  API -->|query/SSE| View
  Stale -.->|every 60s| App
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant API as ApplicationEndpoint
  participant Q as ApplicationQueue
  participant C as ApplicationConsumer
  participant W as RenewalWorkflow
  participant RC as RenewalCoordinatorAgent
  participant DR as DocumentReviewAgent
  participant E as ApplicationEntity
  participant CW as CaseworkerControlEntity
  participant CS as Caseworker
  participant V as ApplicationView

  U->>API: POST /api/applications {renewal fields}
  API->>Q: append ApplicationSubmitted
  API-->>U: 202 {applicationId}
  Q->>C: ApplicationSubmitted
  C->>W: start({applicationId})
  W->>E: emit ApplicationCreated (SUBMITTED→PLANNING)
  W->>RC: PLAN_RENEWAL(facts, nationality)
  RC-->>W: RenewalLedger
  W->>E: emit ApplicationPlanned (→REVIEWING_DOCS)
  loop until all docs pass
    W->>RC: DECIDE_NEXT(ledgers)
    RC-->>W: Continue(ReviewDecision)
    W->>W: PiiTokenizer.tokenize(request) → PiiContext
    W->>DR: REVIEW_DOCUMENT(documentId, PiiContext)
    DR-->>W: DocumentReviewResult
    W->>E: emit DocumentReviewRecorded (ReviewEntry)
  end
  W->>E: emit ReviewCompleted
  W->>E: emit CaseworkerApprovalRequested (→AWAITING_CASEWORKER)
  W->>CW: poll get(applicationId)
  CS->>API: POST /applications/{id}/caseworker-approve
  API->>CW: recordDecision(APPROVED, reason)
  W->>CW: get → APPROVED
  W->>E: emit CaseworkerApproved (→SUBMITTING)
  W->>W: call agency endpoint fixture
  W->>E: emit AgencySubmitted
  W->>RC: COMPOSE_ANSWER
  RC-->>W: ApplicationAnswer
  W->>E: emit ApplicationCompleted (→COMPLETED)
  E-->>V: project
  V-->>U: SSE update
```

## State machine — `ApplicationEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> PLANNING: ApplicationCreated
  PLANNING --> REVIEWING_DOCS: ApplicationPlanned
  REVIEWING_DOCS --> REVIEWING_DOCS: DocumentReviewRecorded
  REVIEWING_DOCS --> BLOCKED: DocumentsMissing
  REVIEWING_DOCS --> AWAITING_CASEWORKER: ReviewCompleted
  AWAITING_CASEWORKER --> SUBMITTING: CaseworkerApproved
  AWAITING_CASEWORKER --> REJECTED: CaseworkerRejected
  SUBMITTING --> COMPLETED: ApplicationCompleted
  BLOCKED --> [*]
  REJECTED --> [*]
  COMPLETED --> [*]
```

## Entity model

```mermaid
erDiagram
  ApplicationEntity ||--o{ ApplicationCreated : emits
  ApplicationEntity ||--o{ ApplicationPlanned : emits
  ApplicationEntity ||--o{ DocumentReviewRecorded : emits
  ApplicationEntity ||--o{ DocumentsMissing : emits
  ApplicationEntity ||--o{ ReviewCompleted : emits
  ApplicationEntity ||--o{ CaseworkerApprovalRequested : emits
  ApplicationEntity ||--o{ CaseworkerApproved : emits
  ApplicationEntity ||--o{ CaseworkerRejected : emits
  ApplicationEntity ||--o{ AgencySubmitted : emits
  ApplicationEntity ||--o{ ApplicationCompleted : emits
  ApplicationEntity ||--o{ ApplicationBlocked : emits
  ApplicationView }o--|| ApplicationEntity : projects
  CaseworkerControlEntity ||--o{ CaseworkerDecisionRecorded : emits
  ApplicationQueue ||--o{ ApplicationSubmitted : emits
  ApplicationConsumer }o--|| ApplicationQueue : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `RenewalCoordinatorAgent` | `application/RenewalCoordinatorAgent.java` |
| `DocumentReviewAgent` | `application/DocumentReviewAgent.java` |
| `RenewalWorkflow` | `application/RenewalWorkflow.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/Application.java`, events in `domain/ApplicationEvent.java`) |
| `CaseworkerControlEntity` | `application/CaseworkerControlEntity.java` |
| `ApplicationQueue` | `application/ApplicationQueue.java` |
| `ApplicationView` | `application/ApplicationView.java` |
| `ApplicationConsumer` | `application/ApplicationConsumer.java` |
| `RequestSimulator` | `application/RequestSimulator.java` |
| `StaleApplicationMonitor` | `application/StaleApplicationMonitor.java` |
| `PiiTokenizer` | `application/PiiTokenizer.java` |
| `CoordinatorTasks` | `application/CoordinatorTasks.java` |
| `ReviewerTasks` | `application/ReviewerTasks.java` |
| `ApplicationEndpoint` | `api/ApplicationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Workflow step timeouts:** `planStep` 60 s, `proposeStep` 45 s, `documentReviewStep` 90 s, `agencySubmitStep` 30 s, `completeStep` 60 s. `caseworkerWaitStep` has no timeout — it is a deliberate indefinite gate; the caseworker must act. Default recovery: `maxRetries(2).failoverTo(RenewalWorkflow::error)`.
- **Missing-doc halt:** if `DocumentReviewResult.ok=false`, the workflow immediately transitions to `blockedStep` without further loop iterations. The `blockReason` names the document and the `missingReason`.
- **Caseworker poll:** `caseworkerWaitStep` reads `CaseworkerControlEntity.get(applicationId)` every 5 s. No caching. A decision recorded between poll ticks is picked up on the next tick.
- **PII isolation:** `piiTokenizeStep` executes in the workflow between `proposeStep` and `documentReviewStep`. The `RenewalRequest`'s raw fields never leave the tokenize step — only `PiiContext` is forwarded to subsequent steps and to agent calls.
- **Idempotency:** `ApplicationEndpoint.submit` uses `(applicantId, expiryDate)` over a 30 s window to deduplicate `POST /api/applications`.
- **Stale detection:** `StaleApplicationMonitor` ticks every 60 s; applications in `REVIEWING_DOCS` for more than 10 minutes are marked `BLOCKED`. The workflow's next `recordStep` checks entity status and exits if `BLOCKED`.
