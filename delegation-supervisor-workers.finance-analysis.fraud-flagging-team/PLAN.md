# PLAN — fraud-flagging-team

Architectural sketch for the delegation-supervisor-workers × finance-analysis cell. All four mermaid diagrams + the component table are required. The generated system renders these on the Architecture tab; include the Lesson 24 theme variables and CSS overrides in `index.html`.

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

  SIM[TransactionSimulator]:::ta -.->|every 30s| TQ[TransactionQueue]:::ese
  FE[FraudEndpoint]:::ep -->|enqueue / confirm / dismiss| TQ
  TQ -.->|TransactionQueued| CONS[TransactionConsumer]:::cons
  CONS -->|open case + start| WF[FraudReviewWorkflow]:::wf
  WF -->|score| FSW[FraudScoringWorker]:::agent
  WF -->|check| CW[ComplianceWorker]:::agent
  WF -->|assess| RAW[RiskAssessmentWorker]:::agent
  WF -->|synthesize| SUP[SupervisorAgent]:::agent
  WF -->|events| CE[CaseEntity]:::ese
  CE -.->|case events| CV[CasesView]:::view
  MON[StuckCaseMonitor]:::ta -.->|every 30s| CV
  MON -->|markEscalated| CE
  FE -->|read / SSE| CV
  FE -->|confirm / dismiss| CE
  AE[AppEndpoint]:::ep -->|static UI| FE
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as Analyst / Feed
  participant FE as FraudEndpoint
  participant WF as FraudReviewWorkflow
  participant W as Workers (x3)
  participant SUP as SupervisorAgent
  participant CE as CaseEntity
  U->>FE: POST /api/cases (transaction)
  FE->>CE: openCase
  FE-->>U: { caseId } (ANALYZING)
  WF->>W: delegate score / check / assess
  W-->>WF: FraudScore, ComplianceFinding, CustomerRiskAssessment
  WF->>SUP: runSingleTask(SYNTHESIZE)
  SUP-->>WF: SupervisorVerdict
  WF->>CE: recordAnalysis + flag | clear
  Note over WF,CE: FLAGGED -> workflow pauses in awaitAnalyst
  U->>FE: POST /api/cases/{id}/confirm
  FE->>CE: confirm
  WF->>CE: before-tool-call guardrail (status==CONFIRMED) -> recordAction
  Note over CE: ACTIONED (terminal)
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> ANALYZING
  ANALYZING --> FLAGGED: supervisor flags
  ANALYZING --> CLEARED: supervisor clears
  FLAGGED --> CONFIRMED: analyst confirms
  FLAGGED --> DISMISSED: analyst dismisses
  FLAGGED --> ESCALATED: review window elapsed
  CONFIRMED --> ACTIONED: gated account action
  CLEARED --> [*]
  DISMISSED --> [*]
  ESCALATED --> [*]
  ACTIONED --> [*]
```

## Entity model

```mermaid
erDiagram
  TRANSACTION_QUEUE ||--o{ CASE : "opens"
  CASE ||--o{ CASE_EVENT : "emits"
  CASE ||--|| CASES_VIEW : "projects"
  CASE {
    string id
    string customerId
    double amount
    enum status
    double fraudScore
    string riskTier
    string supervisorRationale
  }
  CASE_EVENT {
    string type "CaseOpened|AnalysisRecorded|CaseFlagged|CaseCleared|CaseConfirmed|CaseDismissed|CaseEscalated|CaseActioned"
  }
  CASES_VIEW {
    string id
    enum status
  }
```

## Component table

| Component | Akka primitive | Path (generated) |
|---|---|---|
| FraudReviewWorkflow | Workflow | `application/FraudReviewWorkflow.java` |
| SupervisorAgent | AutonomousAgent | `application/SupervisorAgent.java` |
| FraudTasks | task constants | `application/FraudTasks.java` |
| FraudScoringWorker | Agent | `application/FraudScoringWorker.java` |
| ComplianceWorker | Agent | `application/ComplianceWorker.java` |
| RiskAssessmentWorker | Agent | `application/RiskAssessmentWorker.java` |
| CaseEntity | EventSourcedEntity | `application/CaseEntity.java` |
| TransactionQueue | EventSourcedEntity | `application/TransactionQueue.java` |
| CasesView | View | `application/CasesView.java` |
| TransactionConsumer | Consumer | `application/TransactionConsumer.java` |
| TransactionSimulator | TimedAction | `application/TransactionSimulator.java` |
| StuckCaseMonitor | TimedAction | `application/StuckCaseMonitor.java` |
| FraudEndpoint | HttpEndpoint | `api/FraudEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |
| FraudCase / events / records | domain | `domain/*.java` |
| Redaction | helper | `domain/Redaction.java` |

## Concurrency notes

- **Step timeouts.** `delegateStep`, `synthesizeStep`, and `actionStep` call LLM agents; each sets `stepTimeout(ofSeconds(60))` (default 5s is too short — Lesson 4). `defaultStepRecovery(maxRetries(2).failoverTo(error))`.
- **Idempotency.** One workflow per case, keyed by the case UUID. `TransactionConsumer` derives the case id deterministically from the queued transaction event offset so a redelivered event does not open a second case.
- **Human gate.** `awaitAnalystStep` self-schedules a 5-second resume timer while the case is `FLAGGED`; the confirm/dismiss endpoints write to `CaseEntity` and the next poll observes the new status. No external queue.
- **Compensation.** The account action is the only side-effecting step and runs after the human gate; the before-tool-call guardrail re-checks `status == CONFIRMED` so a stale resume cannot trigger an action on a dismissed or escalated case.
- **Escalation.** `StuckCaseMonitor` filters `getAllCases` client-side (no enum WHERE — Lesson 2) for `FLAGGED` cases past the review window and calls `markEscalated`, which ends the paused workflow on its next poll.
