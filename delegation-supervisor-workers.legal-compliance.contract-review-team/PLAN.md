# PLAN — Contract Assistant (Multi-Agent)

Architectural sketch for `/akka:specify`. Mirrors `SPEC.md` Section 4 component names exactly. Mermaid sources here are rendered on the Architecture tab of the embedded UI; carry the Lesson 24 CSS overrides into the generated `index.html`.

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#141414','primaryBorderColor':'#F5C518','primaryTextColor':'#ffffff','lineColor':'#7EC8E3','nodeTextColor':'#ffffff','fontFamily':'Instrument Sans'}}}%%
flowchart TB
  RE[ReviewEndpoint<br/>HttpEndpoint]:::ep
  AE[AppEndpoint<br/>HttpEndpoint]:::ep
  CQ[ContractQueue<br/>EventSourcedEntity]:::ese
  CC[ContractQueueConsumer<br/>Consumer]:::con
  WF[ReviewWorkflow<br/>Workflow]:::wf
  RS[ReviewSupervisor<br/>AutonomousAgent]:::ag
  CA[ClauseAnalyst<br/>AutonomousAgent]:::ag
  RI[RiskScorer<br/>AutonomousAgent]:::ag
  RL[Redliner<br/>AutonomousAgent]:::ag
  CE[ContractReviewEntity<br/>EventSourcedEntity]:::ese
  VW[ReviewView<br/>View]:::vw
  SIM[ContractSimulator<br/>TimedAction]:::ta
  EV[ReviewSampler<br/>TimedAction]:::ta

  RE -->|POST /reviews| CQ
  SIM -.->|every 90s| CQ
  CQ -.->|ContractSubmitted| CC
  CC -->|start workflow| WF
  WF -->|sanitize| WF
  WF -->|PLAN_REVIEW| RS
  WF -->|EXTRACT_CLAUSES| CA
  WF -->|SCORE_RISK| RI
  WF -->|DRAFT_REDLINES| RL
  WF -->|CONSOLIDATE| RS
  WF -->|commands| CE
  CE -.->|events| VW
  EV -.->|every 5m| VW
  EV -->|recordEval| CE
  RE -->|getAllReviews / SSE| VW
  RE -->|approve / reject| CE
  AE --> STATIC[static-resources]:::static

  classDef ep fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ese fill:#141414,stroke:#F5C518,color:#fff;
  classDef vw fill:#141414,stroke:#3fb950,color:#fff;
  classDef wf fill:#141414,stroke:#ff5f57,color:#fff;
  classDef ag fill:#141414,stroke:#B388FF,color:#fff;
  classDef con fill:#141414,stroke:#7EC8E3,color:#fff;
  classDef ta fill:#141414,stroke:#F5C518,color:#fff;
  classDef static fill:#0A0A0A,stroke:#333,color:#aaa;
```

Solid arrows: synchronous commands. Dashed arrows: event subscriptions. Dotted arrows: scheduled ticks.

## Interaction sequence

```mermaid
sequenceDiagram
  participant U as User / Simulator
  participant RE as ReviewEndpoint
  participant CQ as ContractQueue
  participant WF as ReviewWorkflow
  participant RS as ReviewSupervisor
  participant CA as ClauseAnalyst
  participant RI as RiskScorer
  participant RL as Redliner
  participant CE as ContractReviewEntity
  participant L as Lawyer

  U->>RE: POST /api/reviews {contractRef, contractText}
  RE->>CQ: enqueueContract
  CQ-->>WF: ContractQueueConsumer starts workflow
  WF->>CE: createReview (QUEUED)
  WF->>WF: sanitizeStep (legal-sector filter)
  alt sanitizer rejects
    WF->>CE: rejectBySanitizer (REJECTED)
  else sanitizer passes
    WF->>RS: PLAN_REVIEW -> ReviewPlan
    WF->>CE: status IN_REVIEW
    par parallel fan-out
      WF->>CA: EXTRACT_CLAUSES -> ClauseSummary
    and
      WF->>RI: SCORE_RISK -> RiskReport
    and
      WF->>RL: DRAFT_REDLINES -> RedlineSet
    end
    Note over WF: join; if any step times out (60s) -> degradeStep
    WF->>RS: CONSOLIDATE(clauseSummary, riskReport, redlines) -> ReviewPackage
    WF->>WF: guardrailStep vets the package
    alt guardrail passes
      WF->>CE: consolidate (AWAITING_APPROVAL)
      WF->>WF: approvalStep (waits up to 24h)
      L->>RE: POST /api/reviews/{id}/approve or /reject
      RE->>CE: approve (APPROVED) or rejectByLawyer (REJECTED)
    else guardrail fails
      WF->>CE: block (BLOCKED)
    end
  end
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> IN_REVIEW: sanitizer passes
  QUEUED --> REJECTED: sanitizer rejects
  IN_REVIEW --> AWAITING_APPROVAL: consolidate + guardrail pass
  IN_REVIEW --> DEGRADED: a worker timed out
  IN_REVIEW --> BLOCKED: guardrail fail
  AWAITING_APPROVAL --> APPROVED: lawyer approves
  AWAITING_APPROVAL --> REJECTED: lawyer rejects
  AWAITING_APPROVAL --> DEGRADED: approval gate times out
  DEGRADED --> [*]
  BLOCKED --> [*]
  REJECTED --> [*]
  APPROVED --> APPROVED: ReviewEvalScored
  APPROVED --> [*]
```

## Entity model

```mermaid
erDiagram
  CONTRACT_REVIEW ||--o| CLAUSE_SUMMARY : has
  CONTRACT_REVIEW ||--o| RISK_REPORT : has
  CONTRACT_REVIEW ||--o| REDLINE_SET : has
  CONTRACT_REVIEW ||--o| REVIEW_PACKAGE : produces
  CONTRACT_QUEUE ||--|| CONTRACT_REVIEW : seeds
  CLAUSE_SUMMARY ||--o{ CLAUSE_ENTRY : contains
  RISK_REPORT ||--o{ RISK_FLAG : contains
  REDLINE_SET ||--o{ REDLINE_SUGGESTION : contains
  CONTRACT_REVIEW {
    string reviewId
    string contractRef
    enum status
    string submittedBy
    int evalScore
    instant createdAt
  }
  CONTRACT_QUEUE {
    string reviewId
    string contractRef
    string submittedBy
    instant submittedAt
  }
```

## Component table

| Component | Akka primitive | File path |
|---|---|---|
| `ReviewSupervisor` | AutonomousAgent | `application/ReviewSupervisor.java` |
| `ClauseAnalyst` | AutonomousAgent | `application/ClauseAnalyst.java` |
| `RiskScorer` | AutonomousAgent | `application/RiskScorer.java` |
| `Redliner` | AutonomousAgent | `application/Redliner.java` |
| `ReviewTasks` | Task constants | `application/ReviewTasks.java` |
| `ReviewWorkflow` | Workflow | `application/ReviewWorkflow.java` |
| `ContractReviewEntity` | EventSourcedEntity | `domain/ContractReviewEntity.java` |
| `ContractQueue` | EventSourcedEntity | `domain/ContractQueue.java` |
| `ReviewView` | View | `application/ReviewView.java` |
| `ContractQueueConsumer` | Consumer | `application/ContractQueueConsumer.java` |
| `ContractSimulator` | TimedAction | `application/ContractSimulator.java` |
| `ReviewSampler` | TimedAction | `application/ReviewSampler.java` |
| `ReviewEndpoint` | HttpEndpoint | `api/ReviewEndpoint.java` |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` |

## Concurrency notes

- **Step timeouts (Lesson 4):** `extractStep`, `scoreStep`, and `redlineStep` each get 60s; `consolidateStep` gets 90s; `approvalStep` gets 86400s. The 5s default fails every LLM call. `WorkflowSettings` is nested inside `Workflow` — no import.
- **Parallel fan-out:** `extractStep`, `scoreStep`, and `redlineStep` run concurrently via `CompletionStage` allOf, not three sequential step calls.
- **Idempotency:** the workflow id is the `reviewId`. Re-delivery of the same `ContractSubmitted` event resolves to the same workflow instance — no duplicate review.
- **Degrade path (compensation):** if any worker times out, `defaultStepRecovery` routes to `degradeStep`, which consolidates from whichever partial outputs exist and ends with `ReviewDegraded`. No infinite retry.
- **HITL gate:** `approvalStep` suspends the workflow; the step has a 24-hour timeout to cover normal business hours. Timeout produces `ReviewDegraded` (approval window lapsed).
- **Eval sampling:** `ReviewSampler` reads `ReviewView.getAllReviews` (no enum WHERE clause) and filters client-side for the oldest `APPROVED` review lacking an `evalScore`.
