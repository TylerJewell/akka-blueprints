# PLAN — pipeline

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

  API[BriefingEndpoint]:::ep
  Entity[BriefingEntity]:::ese
  WF[BriefingPipelineWorkflow]:::wf
  Agent[ReportAgent]:::agent
  Collect[CollectTools]:::tool
  Analyze[AnalyzeTools]:::tool
  Report[ReportTools]:::tool
  Guard[PhaseGuardrail]:::guard
  Scorer[CompletenessScorer]:::guard
  View[BriefingView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|collectStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Collect
  Agent -->|invokes| Analyze
  Agent -->|invokes| Report
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|SignalSet / Analysis / Briefing| WF
  WF -->|recordSignals/Analysis/Report| Entity
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
  participant U as User (UI)
  participant API as BriefingEndpoint
  participant E as BriefingEntity
  participant W as BriefingPipelineWorkflow
  participant A as ReportAgent
  participant G as PhaseGuardrail
  participant T as Tools (Collect/Analyze/Report)
  participant Sc as CompletenessScorer

  U->>API: POST /api/briefings { topic }
  API->>E: create(topic)
  E-->>API: { briefingId }
  API->>W: start(briefingId, topic)
  W->>E: startCollect
  W->>A: runSingleTask(COLLECT_SIGNALS, topic)
  A->>G: before-tool-call(searchSignals, COLLECT)
  G-->>A: accept (status COLLECTING)
  A->>T: searchSignals + fetchSnippet
  T-->>A: List<Signal>
  A-->>W: SignalSet
  W->>E: recordSignals
  W->>A: runSingleTask(ANALYZE_SIGNALS, signals)
  A->>G: before-tool-call(extractClaims, ANALYZE)
  G-->>A: accept (status ANALYZING and signals present)
  A->>T: extractClaims + clusterClaims
  T-->>A: List<Claim> / List<Theme>
  A-->>W: Analysis
  W->>E: recordAnalysis
  W->>A: runSingleTask(WRITE_REPORT, analysis)
  A->>G: before-tool-call(formatSection, REPORT)
  G-->>A: accept (status REPORTING and analysis present)
  A->>T: formatSection + gatherSources
  T-->>A: Section / List<Source>
  A-->>W: Briefing
  W->>E: recordReport
  W->>Sc: score(briefing, analysis, signals)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `BriefingEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> COLLECTING: CollectStarted
  COLLECTING --> COLLECTED: SignalsCollected
  COLLECTED --> ANALYZING: AnalyzeStarted
  ANALYZING --> ANALYZED: AnalysisProduced
  ANALYZED --> REPORTING: ReportStarted
  REPORTING --> REPORTED: ReportWritten
  REPORTED --> EVALUATED: EvaluationScored
  COLLECTING --> FAILED: BriefingFailed
  ANALYZING --> FAILED: BriefingFailed
  REPORTING --> FAILED: BriefingFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  BriefingEntity ||--o{ BriefingCreated : emits
  BriefingEntity ||--o{ CollectStarted : emits
  BriefingEntity ||--o{ SignalsCollected : emits
  BriefingEntity ||--o{ AnalyzeStarted : emits
  BriefingEntity ||--o{ AnalysisProduced : emits
  BriefingEntity ||--o{ ReportStarted : emits
  BriefingEntity ||--o{ ReportWritten : emits
  BriefingEntity ||--o{ EvaluationScored : emits
  BriefingEntity ||--o{ GuardrailRejected : emits
  BriefingEntity ||--o{ BriefingFailed : emits
  BriefingView }o--|| BriefingEntity : projects
  BriefingPipelineWorkflow }o--|| BriefingEntity : reads-and-writes
  ReportAgent ||--o{ SignalSet : returns
  ReportAgent ||--o{ Analysis : returns
  ReportAgent ||--o{ Briefing : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BriefingEndpoint` | `api/BriefingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BriefingEntity` | `application/BriefingEntity.java` (state in `domain/BriefingRecord.java`, events in `domain/BriefingEvent.java`) |
| `BriefingPipelineWorkflow` | `application/BriefingPipelineWorkflow.java` |
| `ReportAgent` | `application/ReportAgent.java` (tasks in `application/ReportTasks.java`) |
| `CollectTools` | `application/CollectTools.java` |
| `AnalyzeTools` | `application/AnalyzeTools.java` |
| `ReportTools` | `application/ReportTools.java` |
| `PhaseGuardrail` | `application/PhaseGuardrail.java` |
| `CompletenessScorer` | `application/CompletenessScorer.java` |
| `BriefingView` | `application/BriefingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `collectStep` 60 s, `analyzeStep` 60 s, `reportStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(BriefingPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + briefingId` as the workflow id; restart of the same briefingId is rejected by the workflow runtime. The agent instance id is `"agent-" + briefingId` so each briefing has its own per-task conversation memory.
- **One agent per briefing**: `ReportAgent` runs three tasks per briefing — COLLECT, ANALYZE, REPORT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `PhaseGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `CompletenessScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same briefing always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `collectStep` writes `SignalsCollected` BEFORE returning; `analyzeStep` reads the recorded `SignalSet` from the entity to build its task's instruction context; `reportStep` reads both `SignalSet` and `Analysis`. The agent itself is stateless across phases — it never holds collect + analyze + report context in one conversation.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed briefing stays at the last successful event; the UI shows the partial state for the user.
