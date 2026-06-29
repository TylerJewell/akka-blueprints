# PLAN — java-bug-assistant

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
  classDef store fill:#0d1e2a,stroke:#64b5f6,color:#64b5f6;

  API[BugEndpoint]:::ep
  Entity[BugEntity]:::ese
  Normalizer[BugNormalizer]:::cons
  Searcher[TicketSearcher]:::cons
  Store[TicketStore]:::store
  WF[ResolutionWorkflow]:::wf
  Agent[BugResolutionAgent]:::agent
  Guard[TicketWriteGuardrail]:::guard
  Scorer[RecommendationScorer]:::guard
  View[BugView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  Entity -.->|BugSubmitted| Normalizer
  Normalizer -->|attachNormalized| Entity
  Entity -.->|BugNormalized| Searcher
  Searcher -->|query| Store
  Store -->|SearchResult list| Searcher
  Searcher -->|attachSearchResults| Entity
  Searcher -->|start workflow| WF
  WF -->|awaitNormalizedStep poll| Entity
  WF -->|searchStep poll| Entity
  WF -->|resolveStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|ResolutionRecommendation| WF
  WF -->|recordRecommendation| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalResult| WF
  WF -->|recordEvaluation| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as BugEndpoint
  participant E as BugEntity
  participant N as BugNormalizer
  participant S as TicketSearcher
  participant TS as TicketStore
  participant W as ResolutionWorkflow
  participant A as BugResolutionAgent
  participant G as TicketWriteGuardrail
  participant Sc as RecommendationScorer

  U->>API: POST /api/bugs
  API->>E: submit(report)
  E-->>API: { bugId }
  E-.->>N: BugSubmitted
  N->>N: normalize report
  N->>E: attachNormalized
  E-.->>S: BugNormalized
  S->>TS: search(cleanedReport)
  TS-->>S: List<SearchResult>
  S->>E: attachSearchResults
  S->>W: start(bugId)
  W->>E: poll awaitNormalized
  E-->>W: normalized.isPresent()
  W->>E: poll searchStep
  E-->>W: searchResults.isPresent()
  W->>E: startResolving
  W->>A: runSingleTask(searchResults + attachment)
  A->>G: before-tool-call(createTicket args)
  G-->>A: allow
  A-->>W: ResolutionRecommendation
  W->>E: recordRecommendation(recommendation)
  W->>Sc: score(recommendation, searchResults)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation(eval)
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `BugEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> NORMALIZED: BugNormalized
  NORMALIZED --> SEARCHING: SearchStarted
  SEARCHING --> SEARCH_COMPLETE: SearchResultsAttached
  SEARCH_COMPLETE --> RESOLVING: ResolutionStarted
  RESOLVING --> RECOMMENDATION_RECORDED: RecommendationRecorded
  RECOMMENDATION_RECORDED --> EVALUATED: EvaluationScored
  RESOLVING --> FAILED: BugFailed (agent error)
  NORMALIZED --> FAILED: BugFailed (search error)
  SUBMITTED --> FAILED: BugFailed (normalizer error)
  EVALUATED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  BugEntity ||--o{ BugSubmitted : emits
  BugEntity ||--o{ BugNormalized : emits
  BugEntity ||--o{ SearchStarted : emits
  BugEntity ||--o{ SearchResultsAttached : emits
  BugEntity ||--o{ ResolutionStarted : emits
  BugEntity ||--o{ RecommendationRecorded : emits
  BugEntity ||--o{ EvaluationScored : emits
  BugEntity ||--o{ BugFailed : emits
  BugView }o--|| BugEntity : projects
  BugNormalizer }o--|| BugEntity : subscribes
  TicketSearcher }o--|| BugEntity : subscribes
  ResolutionWorkflow }o--|| BugEntity : reads-and-writes
  BugResolutionAgent ||--o{ ResolutionRecommendation : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `BugEndpoint` | `api/BugEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `BugEntity` | `application/BugEntity.java` (state in `domain/Bug.java`, events in `domain/BugEvent.java`) |
| `BugNormalizer` | `application/BugNormalizer.java` |
| `TicketSearcher` | `application/TicketSearcher.java` |
| `TicketStore` | `application/TicketStore.java` |
| `ResolutionWorkflow` | `application/ResolutionWorkflow.java` |
| `BugResolutionAgent` | `application/BugResolutionAgent.java` (tasks in `application/BugTasks.java`) |
| `TicketWriteGuardrail` | `application/TicketWriteGuardrail.java` |
| `RecommendationScorer` | `application/RecommendationScorer.java` |
| `BugView` | `application/BugView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitNormalizedStep` 15 s, `searchStep` 30 s, `resolveStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(ResolutionWorkflow::error)`. The 60 s on `resolveStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"resolution-" + bugId` as the workflow id; the `TicketSearcher` Consumer is allowed to redeliver `BugNormalized` events because `BugEntity.attachSearchResults` is event-version-guarded — a second search attempt against an already-searched bug is a no-op.
- **One agent per bug**: the AutonomousAgent instance id is `"resolver-" + bugId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(4)` caps guardrail-triggered retries at 4.
- **Guardrail-driven retry**: when `TicketWriteGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow's `resolveStep` fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `RecommendationScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same recommendation always scores the same.
- **TicketStore is read-only at runtime**: the seeded tickets are loaded once at startup and never mutated by the running workflow. The guardrail prevents unvalidated writes; validated writes in a real deployment would go through `BugEndpoint` → `BugEntity`, not directly through the store.
