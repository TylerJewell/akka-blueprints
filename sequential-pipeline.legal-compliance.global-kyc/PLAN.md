# PLAN — global-kyc-agent

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
  classDef hitl fill:#0e2020,stroke:#22d3ee,color:#22d3ee;

  API[KycEndpoint]:::ep
  Entity[KycCaseEntity]:::ese
  WF[KycPipelineWorkflow]:::wf
  Agent[KycAgent]:::agent
  Collect[CollectTools]:::tool
  Verify[VerifyTools]:::tool
  Decide[DecideTools]:::tool
  Sanitizer[PiiSanitizer]:::guard
  Hitl[HitlReviewGate]:::hitl
  Scorer[DecisionScorer]:::guard
  View[KycView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|collectStep runSingleTask| Agent
  Agent -->|invokes| Collect
  Agent -->|invokes| Verify
  Agent -->|invokes| Decide
  Agent -->|DocumentSet / VerificationResult / KycDecision| WF
  WF -->|recordDocuments/Verification/Decision| Entity
  WF -->|decideStep DECLINE| Hitl
  Hitl -->|ReviewRequested pause| Entity
  API -->|submitReview resume| WF
  WF -->|evalStep score| Scorer
  Scorer -->|ScorerResult| WF
  WF -->|recordEval| Entity
  Entity -.->|projects| View
  View -.->|before-write| Sanitizer
  Sanitizer -.->|SanitizerApplied| Entity
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path, PASS outcome)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as KycEndpoint
  participant E as KycCaseEntity
  participant W as KycPipelineWorkflow
  participant A as KycAgent
  participant T as Tools (Collect/Verify/Decide)
  participant Sc as DecisionScorer
  participant San as PiiSanitizer
  participant V as KycView

  U->>API: POST /api/cases { applicantId, jurisdiction }
  API->>E: create(applicantId, jurisdiction)
  E-->>API: { caseId }
  API->>W: start(caseId, applicantId, jurisdiction)
  W->>E: startCollect
  W->>A: runSingleTask(COLLECT_DOCUMENTS, context)
  A->>T: fetchDocument + lookupJurisdictionRules
  T-->>A: List<Document>, List<JurisdictionRule>
  A-->>W: DocumentSet
  W->>E: recordDocuments
  E->>San: project DocumentsCollected event
  San->>V: write tokenised KycViewRow
  San->>E: SanitizerApplied
  W->>A: runSingleTask(VERIFY_IDENTITY, documentSet)
  A->>T: checkDocumentAuthenticity + evaluateRule
  T-->>A: DocumentVerification / RuleCheckResult
  A-->>W: VerificationResult
  W->>E: recordVerification
  W->>A: runSingleTask(RENDER_DECISION, verification)
  A->>T: compileDecision + buildRationale
  T-->>A: KycOutcome + rationale
  A-->>W: KycDecision (outcome=PASS)
  W->>E: recordDecision
  W->>Sc: score(decision, verification, documents)
  Sc-->>W: ScorerResult
  W->>E: recordEval
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `KycCaseEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> COLLECTING: CollectStarted
  COLLECTING --> COLLECTED: DocumentsCollected
  COLLECTED --> VERIFYING: VerifyStarted
  VERIFYING --> VERIFIED: IdentityVerified
  VERIFIED --> DECIDING: DecideStarted
  DECIDING --> DECIDED: DecisionRendered (PASS)
  DECIDING --> PENDING_REVIEW: ReviewRequested (DECLINE/REFER/PENDING_DOCS)
  PENDING_REVIEW --> REVIEWED: ReviewResolved
  REVIEWED --> DECIDED: (advance to evalStep)
  DECIDED --> EVALUATED: DecisionEvaluated
  COLLECTING --> FAILED: CaseFailed
  VERIFYING --> FAILED: CaseFailed
  DECIDING --> FAILED: CaseFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`SanitizerApplied` is a side-event recorded on the entity for audit; it does not change the case status. `ReviewRequested` pauses the workflow but does not transition the status until the compliance officer posts a resolution.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  KycCaseEntity ||--o{ CaseCreated : emits
  KycCaseEntity ||--o{ CollectStarted : emits
  KycCaseEntity ||--o{ DocumentsCollected : emits
  KycCaseEntity ||--o{ VerifyStarted : emits
  KycCaseEntity ||--o{ IdentityVerified : emits
  KycCaseEntity ||--o{ DecideStarted : emits
  KycCaseEntity ||--o{ DecisionRendered : emits
  KycCaseEntity ||--o{ ReviewRequested : emits
  KycCaseEntity ||--o{ ReviewResolved : emits
  KycCaseEntity ||--o{ DecisionEvaluated : emits
  KycCaseEntity ||--o{ SanitizerApplied : emits
  KycCaseEntity ||--o{ CaseFailed : emits
  KycView }o--|| KycCaseEntity : projects
  KycPipelineWorkflow }o--|| KycCaseEntity : reads-and-writes
  KycAgent ||--o{ DocumentSet : returns
  KycAgent ||--o{ VerificationResult : returns
  KycAgent ||--o{ KycDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `KycEndpoint` | `api/KycEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `KycCaseEntity` | `application/KycCaseEntity.java` (state in `domain/KycCaseRecord.java`, events in `domain/KycCaseEvent.java`) |
| `KycPipelineWorkflow` | `application/KycPipelineWorkflow.java` |
| `KycAgent` | `application/KycAgent.java` (tasks in `application/KycTasks.java`) |
| `CollectTools` | `application/CollectTools.java` |
| `VerifyTools` | `application/VerifyTools.java` |
| `DecideTools` | `application/DecideTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `HitlReviewGate` | `application/HitlReviewGate.java` |
| `DecisionScorer` | `application/DecisionScorer.java` |
| `KycView` | `application/KycView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `collectStep` 60 s, `verifyStep` 60 s, `decideStep` 60 s, `evalStep` 5 s, `reviewStep` unbounded (human-paced), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(KycPipelineWorkflow::error)`. The reviewStep uses `maxRetries(0)` — no automatic retry of a human review gate.
- **Idempotency**: each workflow uses `"kyc-" + caseId` as the workflow id; restart of the same caseId is rejected by the workflow runtime. The agent instance id is `"agent-" + caseId` so each case has its own per-task conversation memory.
- **One agent per case**: `KycAgent` runs three tasks per case — COLLECT, VERIFY, DECIDE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the agent room to self-correct within a phase.
- **HITL is a workflow pause, not an agent retry**: when `decideStep` returns a DECLINE outcome, `HitlReviewGate` transitions the workflow to `reviewStep`, which waits for external input via the workflow engine's `waitForInput()` mechanism. No additional LLM call is made during review. The compliance officer's POST to `/api/cases/{id}/review` is the resume signal.
- **PII sanitization is synchronous and deterministic**: `PiiSanitizer` runs inside the view table-updater. No LLM call, no external service. The same `DocumentSet` always produces the same tokens. The raw values are never written to the view.
- **Eval is synchronous and deterministic**: `DecisionScorer` runs in-process inside `evalStep`. No LLM call. The same decision always scores the same.
- **Task-boundary handoff is the dependency contract**: `collectStep` writes `DocumentsCollected` BEFORE returning; `verifyStep` reads the recorded `DocumentSet` from the entity to build its task's instruction context; `decideStep` reads both `DocumentSet` and `VerificationResult`. The agent is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed case stays at the last successful event; the UI shows the partial state.
