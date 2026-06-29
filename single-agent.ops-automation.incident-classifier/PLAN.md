# PLAN — incident-classifier

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
  classDef eval fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[ClassificationEndpoint]:::ep
  Entity[IncidentEntity]:::ese
  Validator[VocabularyValidator]:::cons
  WF[ClassificationWorkflow]:::wf
  Agent[IncidentClassifierAgent]:::agent
  Evaluator[AccuracyEvaluator]:::eval
  View[ClassificationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|IncidentSubmitted| Validator
  Validator -->|markValidated| Entity
  Validator -->|start workflow| WF
  WF -->|awaitValidatedStep poll| Entity
  WF -->|classifyStep runSingleTask| Agent
  Agent -->|ClassificationResult| WF
  WF -->|recordClassification| Entity
  WF -->|evalStep score| Evaluator
  Evaluator -->|AccuracyContribution| WF
  WF -->|recordAccuracy| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ClassificationEndpoint
  participant E as IncidentEntity
  participant V as VocabularyValidator
  participant W as ClassificationWorkflow
  participant A as IncidentClassifierAgent
  participant Ev as AccuracyEvaluator

  U->>API: POST /api/incidents
  API->>E: submit(submission)
  E-->>API: { incidentId }
  E-.->>V: IncidentSubmitted
  V->>V: load taxonomy, count categories/subcategories/CIs
  V->>E: markValidated(scope)
  V->>W: start(incidentId)
  W->>E: poll getIncident
  E-->>W: scope.isPresent()
  W->>E: markClassifying
  W->>A: runSingleTask(taxonomy + attachment)
  A-->>W: ClassificationResult
  W->>E: recordClassification(result)
  W->>Ev: score(result, taxonomyTable)
  Ev-->>W: AccuracyContribution
  W->>E: recordAccuracy(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `IncidentEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> TAXONOMY_VALIDATED: TaxonomyValidated
  TAXONOMY_VALIDATED --> CLASSIFYING: ClassificationStarted
  CLASSIFYING --> CLASSIFICATION_RECORDED: ClassificationRecorded
  CLASSIFICATION_RECORDED --> EVALUATED: AccuracyScored
  CLASSIFYING --> FAILED: ClassificationFailed (agent error)
  SUBMITTED --> FAILED: ClassificationFailed (validator error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  IncidentEntity ||--o{ IncidentSubmitted : emits
  IncidentEntity ||--o{ TaxonomyValidated : emits
  IncidentEntity ||--o{ ClassificationStarted : emits
  IncidentEntity ||--o{ ClassificationRecorded : emits
  IncidentEntity ||--o{ AccuracyScored : emits
  IncidentEntity ||--o{ ClassificationFailed : emits
  ClassificationView }o--|| IncidentEntity : projects
  VocabularyValidator }o--|| IncidentEntity : subscribes
  ClassificationWorkflow }o--|| IncidentEntity : reads-and-writes
  IncidentClassifierAgent ||--o{ ClassificationResult : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ClassificationEndpoint` | `api/ClassificationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `IncidentEntity` | `application/IncidentEntity.java` (state in `domain/Incident.java`, events in `domain/IncidentEvent.java`) |
| `VocabularyValidator` | `application/VocabularyValidator.java` |
| `ClassificationWorkflow` | `application/ClassificationWorkflow.java` |
| `IncidentClassifierAgent` | `application/IncidentClassifierAgent.java` (tasks in `application/IncidentTasks.java`) |
| `AccuracyEvaluator` | `application/AccuracyEvaluator.java` |
| `TaxonomyTable` | `application/TaxonomyTable.java` |
| `ClassificationView` | `application/ClassificationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitValidatedStep` 15 s, `classifyStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ClassificationWorkflow::error)`. The 60 s on `classifyStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"classification-" + incidentId` as the workflow id; `VocabularyValidator` is idempotent because `IncidentEntity.markValidated` is event-version-guarded — a second validation attempt against an already-validated incident is a no-op.
- **One agent per incident**: the AutonomousAgent instance id is `"classifier-" + incidentId`, giving each task its own conversation context. `capability(...).maxIterationsPerTask(3)` caps retries.
- **Rolling accuracy**: every `AccuracyScored` event contributes to the ClassificationView's rolling window. The view stores `eval.score` per row; the endpoint aggregates the 50 most recent contributions into a percentage.
- **Eval is synchronous and deterministic**: `AccuracyEvaluator` runs in-process inside `evalStep`. No LLM call — the same classification always scores the same against the same taxonomy. This is a deliberate single-agent guarantee.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, or a single-task agent call. There is nothing external to roll back.
