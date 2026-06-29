# PLAN — sequential-workflow

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

  API[JobEndpoint]:::ep
  Entity[JobEntity]:::ese
  WF[JobPipelineWorkflow]:::wf
  Agent[WorkflowAgent]:::agent
  Validate[ValidateTools]:::tool
  Enrich[EnrichTools]:::tool
  Execute[ExecuteTools]:::tool
  Summarize[SummarizeTools]:::tool
  Guard[StepGuardrail]:::guard
  Scorer[QualityScorer]:::guard
  View[JobView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|validateStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Validate
  Agent -->|invokes| Enrich
  Agent -->|invokes| Execute
  Agent -->|invokes| Summarize
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|ValidationResult / EnrichedJob / JobOutput / JobSummary| WF
  WF -->|recordValidation/Enrichment/Output/Summary| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|QualityResult| WF
  WF -->|recordQuality| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as JobEndpoint
  participant E as JobEntity
  participant W as JobPipelineWorkflow
  participant A as WorkflowAgent
  participant G as StepGuardrail
  participant T as Tools (Validate/Enrich/Execute/Summarize)
  participant Sc as QualityScorer

  U->>API: POST /api/jobs { jobName, jobType, parameters }
  API->>E: create(jobSpec)
  E-->>API: { jobId }
  API->>W: start(jobId, jobSpec)
  W->>E: startValidate
  W->>A: runSingleTask(VALIDATE_JOB, jobSpec)
  A->>G: before-tool-call(checkFields, VALIDATE)
  G-->>A: accept (status VALIDATING)
  A->>T: checkFields + verifyConstraints
  T-->>A: ValidationResult
  A-->>W: ValidationResult
  W->>E: recordValidation
  W->>A: runSingleTask(ENRICH_JOB, validationResult)
  A->>G: before-tool-call(resolveParameters, ENRICH)
  G-->>A: accept (status ENRICHING and validation present)
  A->>T: resolveParameters + attachContext
  T-->>A: EnrichedJob
  A-->>W: EnrichedJob
  W->>E: recordEnrichment
  W->>A: runSingleTask(EXECUTE_JOB, enrichedJob)
  A->>G: before-tool-call(runStep, EXECUTE)
  G-->>A: accept (status EXECUTING and enrichedJob present)
  A->>T: runStep + collectArtifacts
  T-->>A: JobOutput
  A-->>W: JobOutput
  W->>E: recordOutput
  W->>A: runSingleTask(SUMMARIZE_JOB, jobOutput)
  A->>G: before-tool-call(buildSummaryEntry, SUMMARIZE)
  G-->>A: accept (status SUMMARIZING and output present)
  A->>T: buildSummaryEntry + writeOutcomeStatement
  T-->>A: JobSummary
  A-->>W: JobSummary
  W->>E: recordSummary
  W->>Sc: score(jobSummary, jobOutput, enrichedJob)
  Sc-->>W: QualityResult
  W->>E: recordQuality
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `JobEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> VALIDATING: ValidateStarted
  VALIDATING --> VALIDATED: JobValidated
  VALIDATED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: JobEnriched
  ENRICHED --> EXECUTING: ExecuteStarted
  EXECUTING --> EXECUTED: JobExecuted
  EXECUTED --> SUMMARIZING: SummarizeStarted
  SUMMARIZING --> SUMMARIZED: JobSummarized
  SUMMARIZED --> EVALUATED: QualityScored
  VALIDATING --> FAILED: JobFailed
  ENRICHING --> FAILED: JobFailed
  EXECUTING --> FAILED: JobFailed
  SUMMARIZING --> FAILED: JobFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  JobEntity ||--o{ JobCreated : emits
  JobEntity ||--o{ ValidateStarted : emits
  JobEntity ||--o{ JobValidated : emits
  JobEntity ||--o{ EnrichStarted : emits
  JobEntity ||--o{ JobEnriched : emits
  JobEntity ||--o{ ExecuteStarted : emits
  JobEntity ||--o{ JobExecuted : emits
  JobEntity ||--o{ SummarizeStarted : emits
  JobEntity ||--o{ JobSummarized : emits
  JobEntity ||--o{ QualityScored : emits
  JobEntity ||--o{ GuardrailRejected : emits
  JobEntity ||--o{ JobFailed : emits
  JobView }o--|| JobEntity : projects
  JobPipelineWorkflow }o--|| JobEntity : reads-and-writes
  WorkflowAgent ||--o{ ValidationResult : returns
  WorkflowAgent ||--o{ EnrichedJob : returns
  WorkflowAgent ||--o{ JobOutput : returns
  WorkflowAgent ||--o{ JobSummary : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `JobEndpoint` | `api/JobEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `JobEntity` | `application/JobEntity.java` (state in `domain/JobRecord.java`, events in `domain/JobEvent.java`) |
| `JobPipelineWorkflow` | `application/JobPipelineWorkflow.java` |
| `WorkflowAgent` | `application/WorkflowAgent.java` (tasks in `application/WorkflowTasks.java`) |
| `ValidateTools` | `application/ValidateTools.java` |
| `EnrichTools` | `application/EnrichTools.java` |
| `ExecuteTools` | `application/ExecuteTools.java` |
| `SummarizeTools` | `application/SummarizeTools.java` |
| `StepGuardrail` | `application/StepGuardrail.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `JobView` | `application/JobView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `validateStep` 60 s, `enrichStep` 60 s, `executeStep` 60 s, `summarizeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(JobPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + jobId` as the workflow id; restart of the same jobId is rejected by the workflow runtime. The agent instance id is `"agent-" + jobId` so each job has its own per-task conversation memory.
- **One agent per job**: `WorkflowAgent` runs four tasks per job — VALIDATE, ENRICH, EXECUTE, SUMMARIZE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `StepGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same job always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `validateStep` writes `JobValidated` BEFORE returning; `enrichStep` reads the recorded `ValidationResult` from the entity to build its task's instruction context; `executeStep` reads both `ValidationResult` and `EnrichedJob`; `summarizeStep` reads `JobOutput`. The agent itself is stateless across phases — it never holds validate + enrich + execute + summarize context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed job stays at the last successful event; the UI shows the partial state for the user.
