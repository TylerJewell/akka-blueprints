# PLAN — citation-rag

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

  API[QueryEndpoint]:::ep
  Entity[QueryEntity]:::ese
  WF[CitationQueryWorkflow]:::wf
  Agent[CitationAgent]:::agent
  Retrieve[RetrieveTools]:::tool
  Attribute[AttributeTools]:::tool
  Compose[ComposeTools]:::tool
  Guard[CitationGuardrail]:::guard
  Scorer[CoverageScorer]:::guard
  View[QueryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|retrieveStep runSingleTask| Agent
  Agent -.->|after-llm-response| Guard
  Agent -->|invokes| Retrieve
  Agent -->|invokes| Attribute
  Agent -->|invokes| Compose
  Guard -->|recordCitationGuardrailRejection| Entity
  Agent -->|PassageSet / ClaimSet / Answer| WF
  WF -->|recordPassages/Claims/Answer| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|CitationScore| WF
  WF -->|recordCitationScore| Entity
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
  participant API as QueryEndpoint
  participant E as QueryEntity
  participant W as CitationQueryWorkflow
  participant A as CitationAgent
  participant G as CitationGuardrail
  participant T as Tools (Retrieve/Attribute/Compose)
  participant Sc as CoverageScorer

  U->>API: POST /api/queries { queryText }
  API->>E: create(queryText)
  E-->>API: { queryId }
  API->>W: start(queryId, queryText)
  W->>E: startRetrieve
  W->>A: runSingleTask(RETRIEVE_PASSAGES, queryText)
  A->>T: searchPassages + fetchPassage
  T-->>A: List<Passage>
  A-->>W: PassageSet
  W->>E: recordPassages
  W->>A: runSingleTask(ATTRIBUTE_CLAIMS, passages)
  A->>T: extractClaims + linkClaims
  T-->>A: List<Claim>
  A-->>W: ClaimSet
  W->>E: recordClaims
  W->>A: runSingleTask(COMPOSE_ANSWER, claims+passages)
  A->>T: draftAnswer + formatCitations
  T-->>A: AnswerDraft + List<Citation>
  A-->>W: Answer (candidate)
  W->>G: after-llm-response(Answer, PassageSet)
  G-->>W: accept (all claims cited)
  W->>E: recordAnswer
  W->>Sc: score(answer, claimSet, passages)
  Sc-->>W: CitationScore
  W->>E: recordCitationScore
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `QueryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RETRIEVING: RetrieveStarted
  RETRIEVING --> RETRIEVED: PassagesRetrieved
  RETRIEVED --> ATTRIBUTING: AttributeStarted
  ATTRIBUTING --> ATTRIBUTED: ClaimsAttributed
  ATTRIBUTED --> COMPOSING: ComposeStarted
  COMPOSING --> COMPOSED: AnswerComposed
  COMPOSED --> EVALUATED: CitationScored
  RETRIEVING --> FAILED: QueryFailed
  ATTRIBUTING --> FAILED: QueryFailed
  COMPOSING --> FAILED: QueryFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`CitationGuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same COMPOSE task, and the workflow's `composeStep` continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  QueryEntity ||--o{ QueryCreated : emits
  QueryEntity ||--o{ RetrieveStarted : emits
  QueryEntity ||--o{ PassagesRetrieved : emits
  QueryEntity ||--o{ AttributeStarted : emits
  QueryEntity ||--o{ ClaimsAttributed : emits
  QueryEntity ||--o{ ComposeStarted : emits
  QueryEntity ||--o{ AnswerComposed : emits
  QueryEntity ||--o{ CitationScored : emits
  QueryEntity ||--o{ CitationGuardrailRejected : emits
  QueryEntity ||--o{ QueryFailed : emits
  QueryView }o--|| QueryEntity : projects
  CitationQueryWorkflow }o--|| QueryEntity : reads-and-writes
  CitationAgent ||--o{ PassageSet : returns
  CitationAgent ||--o{ ClaimSet : returns
  CitationAgent ||--o{ Answer : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `QueryEndpoint` | `api/QueryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `QueryEntity` | `application/QueryEntity.java` (state in `domain/QueryRecord.java`, events in `domain/QueryEvent.java`) |
| `CitationQueryWorkflow` | `application/CitationQueryWorkflow.java` |
| `CitationAgent` | `application/CitationAgent.java` (tasks in `application/CitationTasks.java`) |
| `RetrieveTools` | `application/RetrieveTools.java` |
| `AttributeTools` | `application/AttributeTools.java` |
| `ComposeTools` | `application/ComposeTools.java` |
| `CitationGuardrail` | `application/CitationGuardrail.java` |
| `CoverageScorer` | `application/CoverageScorer.java` |
| `QueryView` | `application/QueryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `retrieveStep` 60 s, `attributeStep` 60 s, `composeStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CitationQueryWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"citation-" + queryId` as the workflow id; restart of the same queryId is rejected by the workflow runtime. The agent instance id is `"agent-" + queryId` so each query has its own per-task conversation memory.
- **One agent per query**: `CitationAgent` runs three tasks per query — RETRIEVE, ATTRIBUTE, COMPOSE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the citation guardrail room to reject an uncited answer and still let the agent self-correct.
- **Guardrail-driven retry**: when `CitationGuardrail` rejects a COMPOSE response, the structured error is returned to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to FAILED.
- **Eval is synchronous and deterministic**: `CoverageScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same answer always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `retrieveStep` writes `PassagesRetrieved` BEFORE returning; `attributeStep` reads the recorded `PassageSet` from the entity to build its task's instruction context; `composeStep` reads both `PassageSet` and `ClaimSet`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed query stays at the last successful event; the UI shows the partial state for the user.
