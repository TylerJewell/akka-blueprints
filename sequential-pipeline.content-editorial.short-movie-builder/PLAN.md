# PLAN — short-movie-agents

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

  API[MovieEndpoint]:::ep
  Entity[MovieEntity]:::ese
  WF[MovieProductionWorkflow]:::wf
  Agent[MovieAgent]:::agent
  Script[ScriptTools]:::tool
  Storyboard[StoryboardTools]:::tool
  Assemble[AssemblyTools]:::tool
  Review[ReviewTools]:::tool
  Guard[ContentSafetyGuard]:::guard
  Scorer[PackageScorer]:::guard
  View[MovieView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|scriptStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Script
  Agent -->|invokes| Storyboard
  Agent -->|invokes| Assemble
  Agent -->|invokes| Review
  Guard -->|recordSafetyBlock| Entity
  Agent -->|MovieScript / Storyboard / AssembledPackage / ReviewResult| WF
  WF -->|recordScript/Storyboard/Package/Review| Entity
  WF -->|reviewStep score| Scorer
  Scorer -->|CoherenceResult| WF
  WF -->|recordCoherence| Entity
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
  participant API as MovieEndpoint
  participant E as MovieEntity
  participant W as MovieProductionWorkflow
  participant A as MovieAgent
  participant G as ContentSafetyGuard
  participant T as Tools (Script/Storyboard/Assembly/Review)
  participant Sc as PackageScorer

  U->>API: POST /api/productions { brief }
  API->>E: create(brief)
  E-->>API: { productionId }
  API->>W: start(productionId, brief)
  W->>E: startScripting
  W->>A: runSingleTask(WRITE_SCRIPT, brief)
  A->>T: generateScenes + writeDialogueLine
  T-->>A: List<Scene>
  A-->>G: after-llm-response(MovieScript)
  G-->>A: accept
  A-->>W: MovieScript
  W->>E: recordScript
  W->>A: runSingleTask(DESIGN_STORYBOARD, script)
  A->>T: planShot + selectFraming
  T-->>A: List<Shot>
  A-->>G: after-llm-response(Storyboard)
  G-->>A: accept
  A-->>W: Storyboard
  W->>E: recordStoryboard
  W->>A: runSingleTask(ASSEMBLE_PACKAGE, storyboard)
  A->>T: buildPackageScene + computeRuntime
  T-->>A: List<PackageScene>
  A-->>G: after-llm-response(AssembledPackage)
  G-->>A: accept
  A-->>W: AssembledPackage
  W->>E: recordPackage
  W->>A: runSingleTask(REVIEW_PACKAGE, assembledPackage)
  A->>T: checkSceneCoherence + generateReviewSummary
  T-->>A: ReviewResult
  A-->>G: after-llm-response(ReviewResult)
  G-->>A: accept
  A-->>W: ReviewResult
  W->>Sc: score(assembledPackage, storyboard, script)
  Sc-->>W: CoherenceResult
  W->>E: recordReview + recordCoherence
  E-.->>U: SSE event(REVIEWED)
```

## State machine — `MovieEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SCRIPTING: ScriptingStarted
  SCRIPTING --> SCRIPTED: ScriptWritten
  SCRIPTED --> STORYBOARDING: StoryboardingStarted
  STORYBOARDING --> STORYBOARDED: StoryboardDesigned
  STORYBOARDED --> ASSEMBLING: AssemblingStarted
  ASSEMBLING --> ASSEMBLED: PackageAssembled
  ASSEMBLED --> REVIEWING: ReviewingStarted
  REVIEWING --> REVIEWED: ReviewCompleted
  REVIEWED --> [*]
  SCRIPTING --> FAILED: ProductionFailed
  STORYBOARDING --> FAILED: ProductionFailed
  ASSEMBLING --> FAILED: ProductionFailed
  REVIEWING --> FAILED: ProductionFailed
  FAILED --> [*]
```

`SafetyBlockRecorded` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  MovieEntity ||--o{ ProductionCreated : emits
  MovieEntity ||--o{ ScriptingStarted : emits
  MovieEntity ||--o{ ScriptWritten : emits
  MovieEntity ||--o{ StoryboardingStarted : emits
  MovieEntity ||--o{ StoryboardDesigned : emits
  MovieEntity ||--o{ AssemblingStarted : emits
  MovieEntity ||--o{ PackageAssembled : emits
  MovieEntity ||--o{ ReviewingStarted : emits
  MovieEntity ||--o{ ReviewCompleted : emits
  MovieEntity ||--o{ CoherenceScoredEvent : emits
  MovieEntity ||--o{ SafetyBlockRecorded : emits
  MovieEntity ||--o{ ProductionFailed : emits
  MovieView }o--|| MovieEntity : projects
  MovieProductionWorkflow }o--|| MovieEntity : reads-and-writes
  MovieAgent ||--o{ MovieScript : returns
  MovieAgent ||--o{ Storyboard : returns
  MovieAgent ||--o{ AssembledPackage : returns
  MovieAgent ||--o{ ReviewResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MovieEndpoint` | `api/MovieEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MovieEntity` | `application/MovieEntity.java` (state in `domain/MovieRecord.java`, events in `domain/MovieEvent.java`) |
| `MovieProductionWorkflow` | `application/MovieProductionWorkflow.java` |
| `MovieAgent` | `application/MovieAgent.java` (tasks in `application/MovieTasks.java`) |
| `ScriptTools` | `application/ScriptTools.java` |
| `StoryboardTools` | `application/StoryboardTools.java` |
| `AssemblyTools` | `application/AssemblyTools.java` |
| `ReviewTools` | `application/ReviewTools.java` |
| `ContentSafetyGuard` | `application/ContentSafetyGuard.java` |
| `PackageScorer` | `application/PackageScorer.java` |
| `MovieView` | `application/MovieView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `scriptStep` 60 s, `storyboardStep` 60 s, `assembleStep` 60 s, `reviewStep` 60 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MovieProductionWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + productionId` as the workflow id; restart of the same productionId is rejected by the workflow runtime. The agent instance id is `"agent-" + productionId` so each production has its own per-task conversation memory.
- **One agent per production**: `MovieAgent` runs four tasks per production — SCRIPT, STORYBOARD, ASSEMBLE, REVIEW — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives `ContentSafetyGuard` room to reject a policy-violating response and still let the agent self-correct.
- **Guardrail-driven retry**: when `ContentSafetyGuard` rejects a task result, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail the safety check, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `PackageScorer` runs in-process inside `reviewStep`. No LLM call, no external service — the same package always scores the same.
- **Task-boundary handoff is the dependency contract**: `scriptStep` writes `ScriptWritten` BEFORE returning; `storyboardStep` reads the recorded `MovieScript` from the entity to build its task's instruction context; `assembleStep` reads both `MovieScript` and `Storyboard`; `reviewStep` reads all three. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed production stays at the last successful event; the UI shows the partial state for the user.
