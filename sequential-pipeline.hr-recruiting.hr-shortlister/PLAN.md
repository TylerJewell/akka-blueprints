# PLAN — hr-shortlister

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
  classDef adapter fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[ShortlistEndpoint]:::ep
  Entity[ApplicationEntity]:::ese
  DriftEntity[DriftReportEntity]:::ese
  WF[ShortlistingWorkflow]:::wf
  DriftWF[FairnessDriftWorkflow]:::wf
  Agent[ShortlistAgent]:::agent
  Parse[ParseTools]:::tool
  Score[ScoreTools]:::tool
  Decide[ShortlistTools]:::tool
  Guard[SpecialCategoryGuardrail]:::guard
  Scorer[FairnessScorer]:::guard
  View[ApplicationView]:::view
  ERP[ErpNextAdapter]:::adapter
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  WF -->|scoreStep runSingleTask| Agent
  WF -->|shortlistStep runSingleTask| Agent
  Agent -.->|before-tool-call SCORE| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Score
  Agent -->|invokes| Decide
  Guard -->|recordRedaction| Entity
  Agent -->|Profile / CandidateScore / ShortlistDecision| WF
  WF -->|recordProfile/Score/Decision| Entity
  WF -->|approvalStep suspend| Entity
  API -->|POST decision resume| WF
  WF -->|erpNextStep| ERP
  ERP -->|WrittenToErpNext| WF
  WF -->|recordErpNextWrite| Entity
  Entity -.->|projects| View
  View -->|getApplicationsInWindow| DriftWF
  DriftWF -->|FairnessScorer.compute| Scorer
  Scorer -->|DriftReport| DriftWF
  DriftWF -->|update| DriftEntity
  API -->|list/SSE| View
  API -->|read| DriftEntity
  App -->|static| API
```

## Interaction sequence — J1 (happy path with approval)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant R as Recruiter (UI)
  participant API as ShortlistEndpoint
  participant E as ApplicationEntity
  participant W as ShortlistingWorkflow
  participant A as ShortlistAgent
  participant G as SpecialCategoryGuardrail
  participant T as Tools (Parse/Score/Decide)
  participant ERP as ErpNextAdapter

  R->>API: POST /api/applications { jobId, resumeText }
  API->>E: create(jobId, resumeText)
  E-->>API: { applicationId }
  API->>W: start(applicationId, jobId, resumeText)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_RESUME, resumeText)
  A->>T: extractProfile + normalizeEducation
  T-->>A: Profile
  A-->>W: Profile
  W->>E: recordProfile
  W->>E: startScore
  W->>A: runSingleTask(SCORE_PROFILE, profile + jobCriteria)
  A->>G: before-tool-call(evaluateCriteria, SCORE)
  G->>E: recordRedaction(age/gender/... if present)
  G-->>A: accept (sanitized profile)
  A->>T: evaluateCriteria + computeOverallScore
  T-->>A: CandidateScore
  A-->>W: CandidateScore
  W->>E: recordScore
  W->>E: startDecision
  W->>A: runSingleTask(DECIDE_SHORTLIST, score + jobId)
  A->>T: classifyDecision + generateRationale
  T-->>A: ShortlistDecision
  A-->>W: ShortlistDecision
  W->>E: recordDecision
  W->>E: requestApproval (suspend)
  E-.->>R: SSE event(AWAITING_APPROVAL)
  R->>API: POST /api/applications/{id}/decision { approve }
  API->>W: resume(recruiterDecision)
  W->>E: recordRecruiterDecision (RecruiterApproved)
  W->>ERP: writeApplicant(applicationId, decision, profile, score)
  ERP-->>W: { erpNextId }
  W->>E: recordErpNextWrite
  E-.->>R: SSE event(WRITTEN)
```

## State machine — `ApplicationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> PARSING: ParseStarted
  PARSING --> PARSED: ProfileParsed
  PARSED --> SCORING: ScoreStarted
  SCORING --> SCORED: ScoreAssigned
  SCORED --> DECIDING: DecisionStarted
  DECIDING --> DECIDED: ShortlistDecided
  DECIDED --> AWAITING_APPROVAL: ApprovalRequested
  AWAITING_APPROVAL --> APPROVED: RecruiterApproved
  AWAITING_APPROVAL --> APPROVED: RecruiterOverridden
  APPROVED --> WRITTEN: WrittenToErpNext
  PARSING --> FAILED: ApplicationFailed
  SCORING --> FAILED: ApplicationFailed
  DECIDING --> FAILED: ApplicationFailed
  WRITTEN --> [*]
  FAILED --> [*]
```

`SpecialCategoryRedacted` is a side-event recorded on the entity for audit; it does not change the status. `RecruiterOverridden` transitions to `APPROVED` just as `RecruiterApproved` does — the difference is in the payload, not the target state.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ApplicationEntity ||--o{ ResumeReceived : emits
  ApplicationEntity ||--o{ ParseStarted : emits
  ApplicationEntity ||--o{ ProfileParsed : emits
  ApplicationEntity ||--o{ ScoreStarted : emits
  ApplicationEntity ||--o{ ScoreAssigned : emits
  ApplicationEntity ||--o{ DecisionStarted : emits
  ApplicationEntity ||--o{ ShortlistDecided : emits
  ApplicationEntity ||--o{ SpecialCategoryRedacted : emits
  ApplicationEntity ||--o{ ApprovalRequested : emits
  ApplicationEntity ||--o{ RecruiterApproved : emits
  ApplicationEntity ||--o{ RecruiterOverridden : emits
  ApplicationEntity ||--o{ WrittenToErpNext : emits
  ApplicationEntity ||--o{ ApplicationFailed : emits
  ApplicationView }o--|| ApplicationEntity : projects
  ShortlistingWorkflow }o--|| ApplicationEntity : reads-and-writes
  ShortlistAgent ||--o{ Profile : returns
  ShortlistAgent ||--o{ CandidateScore : returns
  ShortlistAgent ||--o{ ShortlistDecision : returns
  DriftReportEntity ||--o{ DriftReportUpdated : emits
  FairnessDriftWorkflow }o--|| ApplicationView : reads
  FairnessDriftWorkflow }o--|| DriftReportEntity : writes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ShortlistEndpoint` | `api/ShortlistEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/ApplicationRecord.java`, events in `domain/ApplicationEvent.java`) |
| `DriftReportEntity` | `application/DriftReportEntity.java` |
| `ShortlistingWorkflow` | `application/ShortlistingWorkflow.java` |
| `FairnessDriftWorkflow` | `application/FairnessDriftWorkflow.java` |
| `ShortlistAgent` | `application/ShortlistAgent.java` (tasks in `application/ShortlistTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `ScoreTools` | `application/ScoreTools.java` |
| `ShortlistTools` | `application/ShortlistTools.java` |
| `SpecialCategoryGuardrail` | `application/SpecialCategoryGuardrail.java` |
| `FairnessScorer` | `application/FairnessScorer.java` |
| `ErpNextAdapter` | `application/ErpNextAdapter.java` |
| `ApplicationView` | `application/ApplicationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `scoreStep` 60 s, `shortlistStep` 60 s, `erpNextStep` 30 s, `error` 5 s. `approvalStep` has no hard timeout — recruiter approval is human-paced. Default step recovery `maxRetries(2).failoverTo(ShortlistingWorkflow::error)`.
- **Idempotency**: each workflow uses `"shortlist-" + applicationId` as the workflow id. The agent instance id is `"agent-" + applicationId` so each application has its own per-task conversation memory.
- **One agent per application**: `ShortlistAgent` runs three tasks per application — PARSE, SCORE, DECIDE — each with `capability(...).maxIterationsPerTask(4)`.
- **Sanitizer is unconditional**: `SpecialCategoryGuardrail` always fires for SCORE-phase tool calls. It never rejects — it sanitizes and passes. The resulting audit trail (one `SpecialCategoryRedacted` event per field per application) is the compliance record.
- **Approval gate is the authority boundary**: the workflow can write to ERPNext only after `RecruiterApproved` or `RecruiterOverridden` has been emitted. The ERPNext write uses the final (possibly overridden) decision, not the agent's raw output.
- **Fairness drift is population-level**: `FairnessScorer` runs outside the per-application pipeline, on the full 30-day window. A single-application guardrail cannot detect population-level skew; the periodic scanner is the right cut for that.
- **No saga / no compensation**: every step is append-only. A failed application stays at the last successful event; the UI shows the partial state.
