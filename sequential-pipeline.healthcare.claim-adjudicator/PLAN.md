# PLAN — claim-adjudication-agent

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

  API[ClaimEndpoint]:::ep
  Entity[ClaimEntity]:::ese
  WF[ClaimAdjudicationWorkflow]:::wf
  Agent[AdjudicationAgent]:::agent
  Validate[ValidateTools]:::tool
  Evaluate[EvaluateTools]:::tool
  Adjudicate[AdjudicateTools]:::tool
  Guard[PhiSanitizerGuardrail]:::guard
  Scorer[AdjudicationScorer]:::guard
  View[ClaimView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|validateStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Validate
  Agent -->|invokes| Evaluate
  Agent -->|invokes| Adjudicate
  Guard -->|recordGuardrailRejection| Entity
  Guard -->|recordPhiRedaction| Entity
  Agent -->|ValidationResult / CoverageEvaluation / AdjudicationDecision| WF
  WF -->|recordValidation/Coverage/Decision| Entity
  WF -->|humanReviewStep if DENIED| Entity
  API -->|approveDenial / overrideToApprove| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (approval path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant P as Processor (UI)
  participant API as ClaimEndpoint
  participant E as ClaimEntity
  participant W as ClaimAdjudicationWorkflow
  participant A as AdjudicationAgent
  participant G as PhiSanitizerGuardrail
  participant T as Tools (Validate/Evaluate/Adjudicate)
  participant Sc as AdjudicationScorer

  P->>API: POST /api/claims { memberInfo, codes, ... }
  API->>E: create(claim)
  E-->>API: { claimId }
  API->>W: start(claimId)
  W->>E: startValidation
  W->>A: runSingleTask(VALIDATE_CLAIM, claimId+plan+codes)
  A->>G: before-tool-call(checkEligibility, VALIDATE)
  G->>G: sanitise PHI fields in args
  G->>E: recordPhiRedaction(memberId, npi)
  G-->>A: pass sanitised call (status VALIDATING)
  A->>T: checkEligibility + verifyProcedureCodes
  T-->>A: EligibilityResult / List<CodeVerification>
  A-->>W: ValidationResult
  W->>E: recordValidation
  W->>A: runSingleTask(EVALUATE_COVERAGE, validationResult)
  A->>G: before-tool-call(lookupPolicyRules, EVALUATE)
  G-->>A: pass (status EVALUATING, validation present)
  A->>T: lookupPolicyRules + computeCoveredAmount
  T-->>A: List<PolicyRule> / CoverageAmount
  A-->>W: CoverageEvaluation
  W->>E: recordCoverage
  W->>A: runSingleTask(ADJUDICATE_CLAIM, coverage+validation)
  A->>G: before-tool-call(buildDecisionRationale, ADJUDICATE)
  G-->>A: pass (status DECIDING, coverage present)
  A->>T: buildDecisionRationale + classifyOutcome
  T-->>A: DecisionRationale / ClaimOutcome
  A-->>W: AdjudicationDecision (APPROVED)
  W->>E: recordDecision
  W->>Sc: score(decision, coverage, validation)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>P: SSE event(ADJUDICATION_EVALUATED)
```

## State machine — `ClaimEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> VALIDATING: ValidationStarted
  VALIDATING --> VALIDATED: ClaimValidated
  VALIDATED --> EVALUATING: EvaluationStarted
  EVALUATING --> EVALUATED_COVERAGE: CoverageEvaluated
  EVALUATED_COVERAGE --> DECIDING: AdjudicationStarted
  DECIDING --> DECIDED: DecisionReached
  DECIDED --> ADJUDICATION_EVALUATED: EvaluationScored (outcome=APPROVED)
  DECIDED --> PENDING_REVIEW: HumanReviewRequested (outcome=DENIED)
  PENDING_REVIEW --> DENIED: DenialApproved
  PENDING_REVIEW --> APPROVED: DenialOverridden
  PENDING_REVIEW --> ESCALATED: ClaimEscalated (timeout)
  DENIED --> ADJUDICATION_EVALUATED: EvaluationScored
  APPROVED --> ADJUDICATION_EVALUATED: EvaluationScored
  VALIDATING --> FAILED: ClaimFailed
  EVALUATING --> FAILED: ClaimFailed
  DECIDING --> FAILED: ClaimFailed
  ADJUDICATION_EVALUATED --> [*]
  ESCALATED --> [*]
  FAILED --> [*]
```

`PhiRedacted` and `GuardrailRejected` are side-events recorded on the entity for audit; they do not change the status. Only an exhausted retry budget, a step timeout, or a PHI sanitisation error transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ClaimEntity ||--o{ ClaimCreated : emits
  ClaimEntity ||--o{ ValidationStarted : emits
  ClaimEntity ||--o{ ClaimValidated : emits
  ClaimEntity ||--o{ EvaluationStarted : emits
  ClaimEntity ||--o{ CoverageEvaluated : emits
  ClaimEntity ||--o{ AdjudicationStarted : emits
  ClaimEntity ||--o{ DecisionReached : emits
  ClaimEntity ||--o{ HumanReviewRequested : emits
  ClaimEntity ||--o{ DenialApproved : emits
  ClaimEntity ||--o{ DenialOverridden : emits
  ClaimEntity ||--o{ ClaimEscalated : emits
  ClaimEntity ||--o{ EvaluationScored : emits
  ClaimEntity ||--o{ PhiRedacted : emits
  ClaimEntity ||--o{ GuardrailRejected : emits
  ClaimEntity ||--o{ ClaimFailed : emits
  ClaimView }o--|| ClaimEntity : projects
  ClaimAdjudicationWorkflow }o--|| ClaimEntity : reads-and-writes
  AdjudicationAgent ||--o{ ValidationResult : returns
  AdjudicationAgent ||--o{ CoverageEvaluation : returns
  AdjudicationAgent ||--o{ AdjudicationDecision : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ClaimEndpoint` | `api/ClaimEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ClaimEntity` | `application/ClaimEntity.java` (state in `domain/ClaimRecord.java`, events in `domain/ClaimEvent.java`) |
| `ClaimAdjudicationWorkflow` | `application/ClaimAdjudicationWorkflow.java` |
| `AdjudicationAgent` | `application/AdjudicationAgent.java` (tasks in `application/AdjudicationTasks.java`) |
| `ValidateTools` | `application/ValidateTools.java` |
| `EvaluateTools` | `application/EvaluateTools.java` |
| `AdjudicateTools` | `application/AdjudicateTools.java` |
| `PhiSanitizerGuardrail` | `application/PhiSanitizerGuardrail.java` |
| `AdjudicationScorer` | `application/AdjudicationScorer.java` |
| `ClaimView` | `application/ClaimView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `validateStep` 90 s, `evaluateStep` 90 s, `adjudicateStep` 90 s, `humanReviewStep` 72 h (dev 10 s), `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ClaimAdjudicationWorkflow::error)`. The 90 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"adj-" + claimId` as the workflow id; restart of the same claimId is rejected by the workflow runtime. The agent instance id is `"agent-" + claimId` so each claim has its own per-task conversation memory.
- **One agent per claim**: `AdjudicationAgent` runs three tasks per claim — VALIDATE, EVALUATE, ADJUDICATE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Sanitiser always fires**: `PhiSanitizerGuardrail` runs on every tool call, including retries after a phase-violation rejection. PHI that arrives in a retry is sanitised the same way as the first attempt.
- **Human-review hold**: `humanReviewStep` blocks indefinitely (within its timeout) waiting for an external command. No agent call or LLM call happens in this step — it is a pure workflow pause.
- **Eval is synchronous and deterministic**: `AdjudicationScorer` runs in-process inside `evalStep`. No LLM call, no external service.
- **Task-boundary handoff is the dependency contract**: `validateStep` writes `ClaimValidated` BEFORE returning; `evaluateStep` reads the recorded `ValidationResult` from the entity to build its task's instruction context; `adjudicateStep` reads both. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed claim stays at the last successful event; the UI shows the partial state for the processor.
