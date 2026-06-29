# PLAN — score-aggregator

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
  classDef det fill:#0e1e0e,stroke:#4ade80,color:#4ade80;
  classDef gate fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ApplicationEndpoint]:::ep
  Entity[ApplicationEntity]:::ese
  WF[ApplicationPipelineWorkflow]:::wf
  Agent[ScreeningAgent]:::agent
  ScoreTools[ScoreTools]:::tool
  RecommendTools[RecommendTools]:::tool
  Aggregator[ScoreAggregator]:::det
  Notifier[StatusNotifier]:::det
  Gate[QualityGate]:::gate
  View[ApplicationView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|screenStep runSingleTask| Agent
  Agent -->|invokes| ScoreTools
  Agent -->|invokes| RecommendTools
  Agent -->|ScreenResult| WF
  WF -->|recordScreenResult| Entity
  WF -->|scoreStep direct call| Aggregator
  Aggregator -->|CandidateScore| WF
  WF -->|recordScore| Entity
  WF -->|recommendStep runSingleTask| Agent
  Agent -->|Recommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|notifyStep direct call| Notifier
  Notifier -->|StatusUpdate| WF
  WF -->|notifyStep direct call| Gate
  Gate -->|GateResult| WF
  WF -->|recordStatusUpdate / recordGateResult| Entity
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
  participant API as ApplicationEndpoint
  participant E as ApplicationEntity
  participant W as ApplicationPipelineWorkflow
  participant A as ScreeningAgent
  participant T as Tools (Score/Recommend)
  participant Agg as ScoreAggregator
  participant N as StatusNotifier
  participant G as QualityGate

  U->>API: POST /api/applications { candidateName, resumeText, targetRole }
  API->>E: create(candidateName, resumeText, targetRole)
  E-->>API: { applicationId }
  API->>W: start(applicationId)
  W->>E: startScreening
  W->>A: runSingleTask(SCREEN_RESUME, resumeText + role)
  A->>T: lookupRoleRequirements(role)
  T-->>A: RoleRequirements
  A->>T: checkSkillMatch(skills, requirements)
  T-->>A: SkillMatchResult
  A-->>W: ScreenResult
  W->>E: recordScreenResult
  W->>Agg: score(screenResult, role, experienceYears, location)
  Agg-->>W: CandidateScore
  W->>E: recordScore
  W->>A: runSingleTask(GENERATE_RECOMMENDATION, screenResult + score)
  A->>T: fetchCandidateHistory(candidateId)
  T-->>A: CandidateHistory
  A->>T: lookupMarketBenchmark(role)
  T-->>A: MarketBenchmark
  A-->>W: Recommendation
  W->>E: recordRecommendation
  W->>N: notify(applicationId, recommendation, score)
  N-->>W: StatusUpdate
  W->>G: evaluate(candidateScore)
  G-->>W: GateResult
  W->>E: recordStatusUpdate + recordGateResult
  E-.->>U: SSE event(NOTIFIED)
```

## State machine — `ApplicationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SCREENING: ScreeningStarted
  SCREENING --> SCREENED: ScreeningCompleted
  SCREENED --> SCORING: ScoringStarted
  SCORING --> SCORED: ScoringCompleted
  SCORED --> RECOMMENDING: RecommendationStarted
  RECOMMENDING --> RECOMMENDED: RecommendationCompleted
  RECOMMENDED --> NOTIFIED: NotificationRecorded
  SCREENING --> FAILED: ApplicationFailed
  SCORING --> FAILED: ApplicationFailed
  RECOMMENDING --> FAILED: ApplicationFailed
  NOTIFIED --> [*]
  FAILED --> [*]
```

GateResultRecorded is recorded on the entity after NOTIFIED; it does not change the status. The gate is non-blocking at runtime — it annotates the application with a PASS/FAIL result; a human recruiter makes the final call. A FAIL gate result highlights the card in the UI.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  ApplicationEntity ||--o{ ApplicationCreated : emits
  ApplicationEntity ||--o{ ScreeningStarted : emits
  ApplicationEntity ||--o{ ScreeningCompleted : emits
  ApplicationEntity ||--o{ ScoringStarted : emits
  ApplicationEntity ||--o{ ScoringCompleted : emits
  ApplicationEntity ||--o{ RecommendationStarted : emits
  ApplicationEntity ||--o{ RecommendationCompleted : emits
  ApplicationEntity ||--o{ NotificationRecorded : emits
  ApplicationEntity ||--o{ GateResultRecorded : emits
  ApplicationEntity ||--o{ ApplicationFailed : emits
  ApplicationView }o--|| ApplicationEntity : projects
  ApplicationPipelineWorkflow }o--|| ApplicationEntity : reads-and-writes
  ScreeningAgent ||--o{ ScreenResult : returns
  ScreeningAgent ||--o{ Recommendation : returns
  ScoreAggregator ||--o{ CandidateScore : returns
  StatusNotifier ||--o{ StatusUpdate : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ApplicationEndpoint` | `api/ApplicationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ApplicationEntity` | `application/ApplicationEntity.java` (state in `domain/ApplicationRecord.java`, events in `domain/ApplicationEvent.java`) |
| `ApplicationPipelineWorkflow` | `application/ApplicationPipelineWorkflow.java` |
| `ScreeningAgent` | `application/ScreeningAgent.java` (tasks in `application/ScreeningTasks.java`) |
| `ScoreTools` | `application/ScoreTools.java` |
| `RecommendTools` | `application/RecommendTools.java` |
| `ScoreAggregator` | `application/ScoreAggregator.java` |
| `StatusNotifier` | `application/StatusNotifier.java` |
| `QualityGate` | `application/QualityGate.java` |
| `ApplicationView` | `application/ApplicationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `screenStep` 60 s, `scoreStep` 10 s, `recommendStep` 60 s, `notifyStep` 10 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ApplicationPipelineWorkflow::error)`. The 60 s on each LLM-calling step accommodates model latency including tool round-trips (Lesson 4). The 10 s on deterministic steps is generous given they make no LLM call.
- **Idempotency**: each workflow uses `"pipeline-" + applicationId` as the workflow id; restart of the same applicationId is rejected by the workflow runtime. The agent instance id is `"agent-" + applicationId` so each application has its own per-task conversation memory.
- **One agent per application**: `ScreeningAgent` runs two tasks per application — SCREEN and RECOMMEND — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget is for the LLM phases only; deterministic steps have no iteration concept.
- **Deterministic steps are synchronous**: `ScoreAggregator.score(...)` and `StatusNotifier.notify(...)` execute in the workflow step thread. They hold no state and make no network calls; both complete in under 1 ms on typical hardware.
- **Gate is non-blocking**: `QualityGate.evaluate(...)` runs inside `notifyStep` after the `StatusUpdate` is recorded. A gate failure records `GateResult{pass: false}` on the entity and highlights the UI card; it does not abort the step or fail the workflow. The decision to block or advance still belongs to a human recruiter.
- **No saga / no compensation**: every step is either pure read, append-only event write, a single-task agent call, or a deterministic in-process function. A failed application stays at the last successful event; the UI shows the partial state for the recruiter.
- **Mixed-agent invariant**: `ScreeningAgent` is the only `AutonomousAgent`. `ScoreAggregator`, `StatusNotifier`, and `QualityGate` are plain Java objects invoked by the workflow. This is the property the blueprint is designed to show — and the CI test gate is the governance mechanism that enforces it holds across builds.
