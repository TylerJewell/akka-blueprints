# PLAN — sba-loan-processor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef tool fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;
  classDef sanitizer fill:#1a0e2a,stroke:#c084fc,color:#c084fc;

  API[LoanEndpoint]:::ep
  Entity[LoanApplicationEntity]:::ese
  WF[LoanPipelineWorkflow]:::wf
  Agent[LoanUnderwritingAgent]:::agent
  Intake[IntakeTools]:::tool
  UW[UnderwriteTools]:::tool
  Decision[DecisionTools]:::tool
  Report[ReportTools]:::tool
  PII[PiiSanitizer]:::sanitizer
  PA[ProtectedAttributeSanitizer]:::sanitizer
  Guard[PhaseGuardrail]:::guard
  Scorer[FairLendingScorer]:::guard
  View[LoanApplicationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start| WF
  WF -->|intakeStep runSingleTask| Agent
  Agent -.->|before-call| PII
  Agent -.->|before-call| PA
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Intake
  Agent -->|invokes| UW
  Agent -->|invokes| Decision
  Agent -->|invokes| Report
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|CreditProfile / Analysis / Decision / Memo| WF
  WF -->|recordCreditProfile/Analysis/Decision| Entity
  WF -->|hitlStep pause| Entity
  API -->|POST review| WF
  WF -->|evalStep score| Scorer
  Scorer -->|FairnessEvalResult| WF
  WF -->|recordFairnessEval| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant O as Officer (UI)
  participant API as LoanEndpoint
  participant E as LoanApplicationEntity
  participant W as LoanPipelineWorkflow
  participant A as LoanUnderwritingAgent
  participant Sn as Sanitizers (PII/PA)
  participant G as PhaseGuardrail
  participant T as Tools (Intake/UW/Decision/Report)
  participant Sc as FairLendingScorer

  O->>API: POST /api/loans { applicantRef, loanAmount, loanPurpose }
  API->>E: submit(applicantRef, loanAmount, loanPurpose)
  E-->>API: { applicationId }
  API->>W: start(applicationId)
  W->>E: startIntake
  W->>Sn: sanitize(applicantRecord) — PII stripped
  W->>A: runSingleTask(INTAKE_APPLICATION, sanitizedRecord)
  A->>G: before-tool-call(lookupCreditScore, INTAKE)
  G-->>A: accept (status INTAKE_IN_PROGRESS)
  A->>T: lookupCreditScore + fetchBusinessProfile
  T-->>A: CreditProfile
  A-->>W: CreditProfile
  W->>E: recordCreditProfile
  W->>Sn: sanitize — protected attributes masked
  W->>A: runSingleTask(UNDERWRITE_APPLICATION, sanitizedCreditProfile)
  A->>G: before-tool-call(analyzeCashFlow, UNDERWRITE)
  G-->>A: accept (status UNDERWRITING, creditProfile present)
  A->>T: analyzeCashFlow + valuateCollateral
  T-->>A: UnderwritingAnalysis
  A-->>W: UnderwritingAnalysis
  W->>E: recordAnalysis
  W->>A: runSingleTask(MAKE_DECISION, analysis)
  A->>G: before-tool-call(computeDscr, DECISION)
  G-->>A: accept (status DECISION_IN_PROGRESS, analysis present)
  A->>T: computeDscr + classifyRiskTier
  T-->>A: LoanDecision
  A-->>W: LoanDecision
  W->>E: recordDecision
  W->>Sc: score(decision, analysis, creditProfile)
  Sc-->>W: FairnessEvalResult
  W->>E: recordFairnessEval
  W->>E: requestOfficerReview → PENDING_REVIEW
  E-.->>O: SSE event(PENDING_REVIEW)
  O->>API: POST /api/loans/{id}/review { outcome: APPROVED }
  API->>W: resume(officerReview)
  W->>E: officerApprove
  W->>A: runSingleTask(GENERATE_REPORT, decision+analysis)
  A->>T: formatDecisionRationale + compileConditions
  T-->>A: UnderwritingMemo
  A-->>W: UnderwritingMemo
  W->>E: recordMemo
  E-.->>O: SSE event(OFFICER_APPROVED)
```

## State machine — `LoanApplicationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> INTAKE_IN_PROGRESS: IntakeStarted
  INTAKE_IN_PROGRESS --> INTAKE_COMPLETE: IntakeCompleted
  INTAKE_COMPLETE --> UNDERWRITING: UnderwriteStarted
  UNDERWRITING --> UNDERWRITING_COMPLETE: UnderwritingCompleted
  UNDERWRITING_COMPLETE --> DECISION_IN_PROGRESS: DecisionStarted
  DECISION_IN_PROGRESS --> DECISION_MADE: DecisionMade
  DECISION_MADE --> PENDING_REVIEW: OfficerReviewRequested
  PENDING_REVIEW --> OFFICER_APPROVED: OfficerApproved
  PENDING_REVIEW --> OFFICER_DENIED: OfficerDenied
  INTAKE_IN_PROGRESS --> FAILED: ApplicationFailed
  UNDERWRITING --> FAILED: ApplicationFailed
  DECISION_IN_PROGRESS --> FAILED: ApplicationFailed
  OFFICER_APPROVED --> [*]
  OFFICER_DENIED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task. Only an exhausted retry budget or a step timeout transitions to FAILED.

FairnessEvaluated fires between DECISION_MADE and PENDING_REVIEW and is an audit-only event — it does not change the application status.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  LoanApplicationEntity ||--o{ ApplicationSubmitted : emits
  LoanApplicationEntity ||--o{ IntakeStarted : emits
  LoanApplicationEntity ||--o{ IntakeCompleted : emits
  LoanApplicationEntity ||--o{ UnderwriteStarted : emits
  LoanApplicationEntity ||--o{ UnderwritingCompleted : emits
  LoanApplicationEntity ||--o{ DecisionStarted : emits
  LoanApplicationEntity ||--o{ DecisionMade : emits
  LoanApplicationEntity ||--o{ FairnessEvaluated : emits
  LoanApplicationEntity ||--o{ MemoGenerated : emits
  LoanApplicationEntity ||--o{ OfficerReviewRequested : emits
  LoanApplicationEntity ||--o{ OfficerApproved : emits
  LoanApplicationEntity ||--o{ OfficerDenied : emits
  LoanApplicationEntity ||--o{ GuardrailRejected : emits
  LoanApplicationEntity ||--o{ ApplicationFailed : emits
  LoanApplicationView }o--|| LoanApplicationEntity : projects
  LoanPipelineWorkflow }o--|| LoanApplicationEntity : reads-and-writes
  LoanUnderwritingAgent ||--o{ CreditProfile : returns
  LoanUnderwritingAgent ||--o{ UnderwritingAnalysis : returns
  LoanUnderwritingAgent ||--o{ LoanDecision : returns
  LoanUnderwritingAgent ||--o{ UnderwritingMemo : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `LoanEndpoint` | `api/LoanEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `LoanApplicationEntity` | `application/LoanApplicationEntity.java` (state in `domain/LoanApplicationRecord.java`, events in `domain/LoanApplicationEvent.java`) |
| `LoanPipelineWorkflow` | `application/LoanPipelineWorkflow.java` |
| `LoanUnderwritingAgent` | `application/LoanUnderwritingAgent.java` (tasks in `application/LoanTasks.java`) |
| `IntakeTools` | `application/IntakeTools.java` |
| `UnderwriteTools` | `application/UnderwriteTools.java` |
| `DecisionTools` | `application/DecisionTools.java` |
| `ReportTools` | `application/ReportTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ProtectedAttributeSanitizer` | `application/ProtectedAttributeSanitizer.java` |
| `PhaseGuardrail` | `application/PhaseGuardrail.java` |
| `FairLendingScorer` | `application/FairLendingScorer.java` |
| `LoanApplicationView` | `application/LoanApplicationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `intakeStep` 60 s, `underwriteStep` 60 s, `decisionStep` 60 s, `hitlStep` no timeout (HITL gates are indefinite), `evalStep` 5 s, `reportStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(LoanPipelineWorkflow::error)`.
- **Idempotency**: each workflow uses `"pipeline-" + applicationId` as the workflow id. The agent instance id is `"agent-" + applicationId` so each application has its own per-task conversation memory.
- **One agent per application**: `LoanUnderwritingAgent` runs four tasks per application — INTAKE, UNDERWRITE, DECISION, REPORT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **HITL pause is durable**: the workflow is suspended at `hitlStep` with no time limit. An Akka Workflow survives restarts in the paused state; the `PENDING_REVIEW` status on the entity is the durable signal that a human action is required.
- **Sanitizers run before context assembly**: both `PiiSanitizer` and `ProtectedAttributeSanitizer` are registered on the agent as `before-call` hooks. They fire synchronously before the task's instruction context is sent to the model; they do not run on tool-call results.
- **Eval is synchronous and deterministic**: `FairLendingScorer` runs in-process inside `evalStep`. No LLM call; the same decision always scores the same.
- **No saga / no compensation**: every step is either a pure append-only event write, a single-task agent call, or the durable HITL pause. A failed application stays at the last successful event; the UI shows partial state.
