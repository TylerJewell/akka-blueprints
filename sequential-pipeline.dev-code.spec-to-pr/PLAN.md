# PLAN — spec-to-pr

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
  classDef hitl fill:#1a1a2a,stroke:#818cf8,color:#818cf8;

  API[SpecEndpoint]:::ep
  Entity[SpecRunEntity]:::ese
  WF[SpecPipelineWorkflow]:::wf
  Agent[ImplementationAgent]:::agent
  Parse[ParseTools]:::tool
  Plan[PlanTools]:::tool
  Draft[DraftTools]:::tool
  Guard[WriteGuardrail]:::guard
  Hold[ReviewHold]:::hitl
  CI[CiScorer]:::guard
  View[SpecRunView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Plan
  Agent -->|invokes| Draft
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ParsedSpec / ChangePlan / DraftPr| WF
  WF -->|recordParsedSpec/ChangePlan/DraftPr| Entity
  WF -->|reviewHoldStep pause| Hold
  API -->|approve/reject| Entity
  Entity -->|signal resume| WF
  WF -->|ciCheckStep| CI
  CI -->|CiResult| WF
  WF -->|recordCiPassed/CiFailed| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 + J3 (happy path with review)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant R as Reviewer (UI)
  participant API as SpecEndpoint
  participant E as SpecRunEntity
  participant W as SpecPipelineWorkflow
  participant A as ImplementationAgent
  participant G as WriteGuardrail
  participant T as Tools (Parse/Plan/Draft)
  participant CI as CiScorer

  U->>API: POST /api/runs { specText }
  API->>E: create(specText)
  E-->>API: { runId }
  API->>W: start(runId, specText)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_SPEC, specText)
  A->>G: before-tool-call(extractRequirements, PARSE)
  G-->>A: accept (status PARSING)
  A->>T: extractRequirements + identifyAffectedFiles
  T-->>A: ParsedSpec
  A-->>W: ParsedSpec
  W->>E: recordParsedSpec
  W->>A: runSingleTask(PLAN_CHANGES, parsedSpec)
  A->>G: before-tool-call(lookupFileContent, PLAN)
  G-->>A: accept (status PLANNING and parsedSpec present)
  A->>T: lookupFileContent + proposeEdit
  T-->>A: ChangePlan
  A-->>W: ChangePlan
  W->>E: recordChangePlan
  W->>A: runSingleTask(DRAFT_PR, changePlan)
  A->>G: before-tool-call(composePrTitle, DRAFT)
  G-->>A: accept (status DRAFTING and changePlan present)
  A->>T: composePrTitle + composePrDescription
  T-->>A: DraftPr
  A-->>W: DraftPr
  W->>E: recordDraftPr
  W->>W: reviewHoldStep (pause — awaiting human signal)
  E-.->>U: SSE event(AWAITING_REVIEW)
  R->>API: POST /api/runs/{id}/approve { reviewerId, comment }
  API->>E: approve(reviewDecision)
  E-->>W: signal("review-decision", approved)
  W->>E: startCi
  W->>CI: evaluate(draftPr, changePlan)
  CI-->>W: CiResult { passed: true }
  W->>E: recordCiPassed
  E-.->>U: SSE event(MERGE_READY)
```

## State machine — `SpecRunEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: SpecParsed
  PARSED --> PLANNING: PlanStarted
  PLANNING --> PLANNED: PlanProduced
  PLANNED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: PrDrafted
  DRAFTED --> AWAITING_REVIEW: (workflow parks)
  AWAITING_REVIEW --> CI_RUNNING: ReviewApproved
  AWAITING_REVIEW --> REJECTED: ReviewRejected
  CI_RUNNING --> CI_PASSED: CiPassed
  CI_PASSED --> MERGE_READY: (auto-advance)
  CI_RUNNING --> CI_FAILED: CiFailed
  PARSING --> FAILED: RunFailed
  PLANNING --> FAILED: RunFailed
  DRAFTING --> FAILED: RunFailed
  MERGE_READY --> [*]
  REJECTED --> [*]
  CI_FAILED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status. ReviewHold is a workflow pause with no timer — only an explicit human signal advances the run. CI_FAILED is terminal; a new run must be submitted if the deployer wants to retry.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  SpecRunEntity ||--o{ RunCreated : emits
  SpecRunEntity ||--o{ ParseStarted : emits
  SpecRunEntity ||--o{ SpecParsed : emits
  SpecRunEntity ||--o{ PlanStarted : emits
  SpecRunEntity ||--o{ PlanProduced : emits
  SpecRunEntity ||--o{ DraftStarted : emits
  SpecRunEntity ||--o{ PrDrafted : emits
  SpecRunEntity ||--o{ ReviewApproved : emits
  SpecRunEntity ||--o{ ReviewRejected : emits
  SpecRunEntity ||--o{ CiStarted : emits
  SpecRunEntity ||--o{ CiPassed : emits
  SpecRunEntity ||--o{ CiFailed : emits
  SpecRunEntity ||--o{ GuardrailRejected : emits
  SpecRunEntity ||--o{ RunFailed : emits
  SpecRunView }o--|| SpecRunEntity : projects
  SpecPipelineWorkflow }o--|| SpecRunEntity : reads-and-writes
  ImplementationAgent ||--o{ ParsedSpec : returns
  ImplementationAgent ||--o{ ChangePlan : returns
  ImplementationAgent ||--o{ DraftPr : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `SpecEndpoint` | `api/SpecEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `SpecRunEntity` | `application/SpecRunEntity.java` (state in `domain/SpecRunRecord.java`, events in `domain/SpecRunEvent.java`) |
| `SpecPipelineWorkflow` | `application/SpecPipelineWorkflow.java` |
| `ImplementationAgent` | `application/ImplementationAgent.java` (tasks in `application/SpecTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `PlanTools` | `application/PlanTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `WriteGuardrail` | `application/WriteGuardrail.java` |
| `CiScorer` | `application/CiScorer.java` |
| `SpecRunView` | `application/SpecRunView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `planStep` 60 s, `draftStep` 60 s, `ciCheckStep` 10 s, `error` 5 s. `reviewHoldStep` has no timeout — the hold is unbounded by design; only an explicit `ReviewApproved` or `ReviewRejected` signal unblocks it.
- **HITL hold is not a timeout**: the workflow uses `Workflow.pause()` and resumes only on a human-triggered entity command that signals the workflow. Auto-advance after a timer is explicitly NOT wired.
- **Idempotency**: each workflow uses `"pipeline-" + runId` as the workflow id. The agent instance id is `"agent-" + runId` so each run has its own per-task conversation memory.
- **One agent per run**: `ImplementationAgent` runs three tasks per run — PARSE, PLAN, DRAFT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget accommodates a guardrail rejection and a self-correction.
- **CI gate is synchronous and deterministic**: `CiScorer` runs in-process inside `ciCheckStep`. No LLM call — the same DraftPr always scores the same.
- **Task-boundary handoff is the dependency contract**: `parseStep` writes `SpecParsed` BEFORE returning; `planStep` reads the recorded `ParsedSpec` from the entity to build its task's instruction context; `draftStep` reads both `ParsedSpec` and `ChangePlan`. The agent is stateless across phases.
- **Guardrail-driven retry**: when `WriteGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error`.
