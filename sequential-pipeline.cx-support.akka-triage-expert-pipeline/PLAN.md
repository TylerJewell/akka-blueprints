# PLAN — triage-expert-multi-agent-workflow

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

  API[SupportCaseEndpoint]:::ep
  Entity[SupportCaseEntity]:::ese
  WF[SupportCaseWorkflow]:::wf
  TA[TriageAgent]:::agent
  EA[ExpertAgent]:::agent
  Intake[IntakeTools]:::tool
  KB[KnowledgeBaseTools]:::tool
  Sanitizer[PiiSanitizer]:::sanitizer
  Guard[RecommendationGuardrail]:::guard
  Scorer[RecommendationScorer]:::guard
  View[SupportCaseView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|gatherStep runSingleTask| TA
  TA -->|invokes| Intake
  TA -->|CustomerInfo| WF
  WF -->|summarizeStep runSingleTask| TA
  TA -->|IssueSummary| WF
  WF -->|sanitizeStep| Sanitizer
  Sanitizer -->|SanitizedSummary| WF
  WF -->|recommendStep runSingleTask| EA
  EA -.->|before-agent-response| Guard
  Guard -->|recordGuardrailBlock| Entity
  EA -->|invokes| KB
  EA -->|Recommendation| WF
  WF -->|recordCustomerInfo/Summary/Sanitized/Recommendation| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as Customer (UI)
  participant API as SupportCaseEndpoint
  participant E as SupportCaseEntity
  participant W as SupportCaseWorkflow
  participant TA as TriageAgent
  participant IT as IntakeTools
  participant PS as PiiSanitizer
  participant EA as ExpertAgent
  participant GD as RecommendationGuardrail
  participant KB as KnowledgeBaseTools
  participant Sc as RecommendationScorer

  U->>API: POST /api/cases { issueDescription }
  API->>E: create(issueDescription)
  E-->>API: { caseId }
  API->>W: start(caseId, issueDescription)
  W->>E: startGather
  W->>TA: runSingleTask(GATHER_CUSTOMER_INFO, issueDescription)
  TA->>IT: lookupCustomerAccount + classifyIssue
  IT-->>TA: AccountRecord / IssueCategory
  TA-->>W: CustomerInfo
  W->>E: recordCustomerInfo
  W->>TA: runSingleTask(SUMMARIZE_ISSUE, customerInfo)
  TA-->>W: IssueSummary
  W->>E: recordIssueSummary
  W->>PS: sanitize(issueSummary, customerInfo)
  PS-->>W: SanitizedSummary
  W->>E: recordSanitizedSummary
  W->>EA: runSingleTask(COMPOSE_RECOMMENDATION, sanitizedSummary)
  EA->>KB: searchArticles + fetchArticle
  KB-->>EA: List<ArticleRef> / ArticleContent
  EA->>GD: before-agent-response(Recommendation)
  GD-->>EA: accept
  EA-->>W: Recommendation
  W->>E: recordRecommendation
  W->>Sc: score(recommendation, sanitizedSummary)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `SupportCaseEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> GATHERING: GatherStarted
  GATHERING --> GATHERED: CustomerInfoGathered
  GATHERED --> SUMMARIZING: SummarizeStarted
  SUMMARIZING --> SUMMARIZED: IssueSummarized
  SUMMARIZED --> SANITIZING: SanitizeStarted
  SANITIZING --> SANITIZED: SummarySanitized
  SANITIZED --> RECOMMENDING: RecommendStarted
  RECOMMENDING --> RECOMMENDED: RecommendationComposed
  RECOMMENDED --> EVALUATED: EvaluationScored
  GATHERING --> FAILED: CaseFailed
  SUMMARIZING --> FAILED: CaseFailed
  RECOMMENDING --> FAILED: CaseFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`GuardrailBlocked` is a side-event recorded on the entity for audit; it does not change the status — the expert agent's retry stays inside the same task, and the workflow's `recommendStep` continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  SupportCaseEntity ||--o{ CaseCreated : emits
  SupportCaseEntity ||--o{ GatherStarted : emits
  SupportCaseEntity ||--o{ CustomerInfoGathered : emits
  SupportCaseEntity ||--o{ SummarizeStarted : emits
  SupportCaseEntity ||--o{ IssueSummarized : emits
  SupportCaseEntity ||--o{ SanitizeStarted : emits
  SupportCaseEntity ||--o{ SummarySanitized : emits
  SupportCaseEntity ||--o{ RecommendStarted : emits
  SupportCaseEntity ||--o{ RecommendationComposed : emits
  SupportCaseEntity ||--o{ EvaluationScored : emits
  SupportCaseEntity ||--o{ GuardrailBlocked : emits
  SupportCaseEntity ||--o{ CaseFailed : emits
  SupportCaseView }o--|| SupportCaseEntity : projects
  SupportCaseWorkflow }o--|| SupportCaseEntity : reads-and-writes
  TriageAgent ||--o{ CustomerInfo : returns
  TriageAgent ||--o{ IssueSummary : returns
  ExpertAgent ||--o{ Recommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SupportCaseEndpoint` | `api/SupportCaseEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SupportCaseEntity` | `application/SupportCaseEntity.java` (state in `domain/SupportCaseRecord.java`, events in `domain/SupportCaseEvent.java`) |
| `SupportCaseWorkflow` | `application/SupportCaseWorkflow.java` |
| `TriageAgent` | `application/TriageAgent.java` (tasks in `application/TriageTasks.java`) |
| `ExpertAgent` | `application/ExpertAgent.java` (tasks in `application/ExpertTasks.java`) |
| `IntakeTools` | `application/IntakeTools.java` |
| `KnowledgeBaseTools` | `application/KnowledgeBaseTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `RecommendationScorer` | `application/RecommendationScorer.java` |
| `SupportCaseView` | `application/SupportCaseView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `gatherStep` 60 s, `summarizeStep` 60 s, `sanitizeStep` 5 s, `recommendStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(SupportCaseWorkflow::error)`. The 90 s on `recommendStep` accommodates three guardrail retry iterations plus LLM latency (Lesson 4).
- **Idempotency**: each workflow uses `"workflow-" + caseId` as the workflow id; resubmitting the same caseId is rejected by the workflow runtime. The agent instance ids are `"triage-" + caseId` and `"expert-" + caseId` so each case has its own per-task conversation memory on each agent.
- **Two agents, one case**: `TriageAgent` runs two tasks per case (GATHER, SUMMARIZE); `ExpertAgent` runs one task (COMPOSE_RECOMMENDATION). Each with `maxIterationsPerTask(3)`. The 3-iteration budget on ExpertAgent gives the guardrail room to block a non-compliant response and let the agent self-correct.
- **Sanitizer is synchronous**: `PiiSanitizer` runs in-process inside `sanitizeStep`. No LLM call, no external service — fast and deterministic. The sanitized summary is the ONLY text forwarded to ExpertAgent; the original `IssueSummary` stays on the entity for authorized audit queries.
- **Guardrail-driven retry**: when `RecommendationGuardrail` blocks a response, the block is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask(3)`; if all 3 iterations are blocked, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `RecommendationScorer` runs in-process inside `evalStep`. No LLM call — the same recommendation always scores the same. This is deliberate: an evaluator that itself calls an LLM can drift; a rule-based scorer cannot.
- **No saga / no compensation**: every step is either a pure in-process call, an append-only event write, or a single-task agent call. A failed case stays at the last successful event; the UI shows the partial state.
