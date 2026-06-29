# PLAN — pokemon-advisor

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[AdvisoryEndpoint]:::ep
  Entity[RosterEntity]:::ese
  Validator[RosterValidator]:::cons
  WF[AdvisoryWorkflow]:::wf
  Agent[TeamAdvisorAgent]:::agent
  Guard[RecommendationGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[AdvisoryView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|RosterSubmitted| Validator
  Validator -->|attachValidated| Entity
  Validator -->|start workflow| WF
  WF -->|awaitValidatedStep poll| Entity
  WF -->|adviseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|TeamRecommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|scoreStep score| Scorer
  Scorer -->|CoverageScore| WF
  WF -->|recordCoverageScore| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Trainer (UI)
  participant API as AdvisoryEndpoint
  participant E as RosterEntity
  participant V as RosterValidator
  participant W as AdvisoryWorkflow
  participant A as TeamAdvisorAgent
  participant G as RecommendationGuardrail
  participant Sc as CoverageScorer

  U->>API: POST /api/advisories
  API->>E: submit(submission)
  E-->>API: { advisoryId }
  E-.->>V: RosterSubmitted
  V->>V: check legality
  V->>E: attachValidated
  V->>W: start(advisoryId)
  W->>E: poll getAdvisory
  E-->>W: validated.isPresent()
  W->>E: markAdvising
  W->>A: runSingleTask(format constraints + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: TeamRecommendation
  W->>E: recordRecommendation(recommendation)
  W->>Sc: score(recommendation, roster)
  Sc-->>W: CoverageScore
  W->>E: recordCoverageScore(score)
  E-.->>U: SSE event(SCORED)
```

## State machine — `RosterEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> VALIDATED: RosterValidated
  VALIDATED --> ADVISING: AdvisingStarted
  ADVISING --> RECOMMENDATION_RECORDED: RecommendationRecorded
  RECOMMENDATION_RECORDED --> SCORED: CoverageScored
  ADVISING --> FAILED: AdvisoryFailed (agent error)
  SUBMITTED --> FAILED: AdvisoryFailed (validation error)
  SCORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  RosterEntity ||--o{ RosterSubmitted : emits
  RosterEntity ||--o{ RosterValidated : emits
  RosterEntity ||--o{ AdvisingStarted : emits
  RosterEntity ||--o{ RecommendationRecorded : emits
  RosterEntity ||--o{ CoverageScored : emits
  RosterEntity ||--o{ AdvisoryFailed : emits
  AdvisoryView }o--|| RosterEntity : projects
  RosterValidator }o--|| RosterEntity : subscribes
  AdvisoryWorkflow }o--|| RosterEntity : reads-and-writes
  TeamAdvisorAgent ||--o{ TeamRecommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AdvisoryEndpoint` | `api/AdvisoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `RosterEntity` | `application/RosterEntity.java` (state in `domain/Advisory.java`, events in `domain/AdvisoryEvent.java`) |
| `RosterValidator` | `application/RosterValidator.java` |
| `AdvisoryWorkflow` | `application/AdvisoryWorkflow.java` |
| `TeamAdvisorAgent` | `application/TeamAdvisorAgent.java` (tasks in `application/AdvisoryTasks.java`) |
| `RecommendationGuardrail` | `application/RecommendationGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `AdvisoryView` | `application/AdvisoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidatedStep` 15 s, `adviseStep` 60 s, `scoreStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AdvisoryWorkflow::error)`. The 60 s on `adviseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"advisory-" + advisoryId` as the workflow id; the `RosterValidator` Consumer is allowed to redeliver `RosterSubmitted` events because `RosterEntity.attachValidated` is event-version-guarded — a second validate attempt against an already-validated advisory is a no-op.
- **One agent per advisory**: the AutonomousAgent instance id is `"advisor-" + advisoryId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `RecommendationGuardrail` rejects a candidate response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, the workflow's `adviseStep` fails over to `error` and the entity transitions to `FAILED`.
- **Scoring is synchronous and deterministic**: `CoverageScorer` runs in-process inside `scoreStep`. No LLM call, no external service — the same recommendation always scores the same. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. There is nothing external to roll back.
