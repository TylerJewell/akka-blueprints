# PLAN — siem-enrichment

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

  API[AlertEndpoint]:::ep
  Entity[AlertEntity]:::ese
  WF[AlertEnrichmentWorkflow]:::wf
  Agent[AlertEnrichmentAgent]:::agent
  Fetch[FetchTools]:::tool
  Enrich[EnrichTools]:::tool
  Ticket[TicketTools]:::tool
  Guard[TicketScopeGuardrail]:::guard
  Scorer[TriageQualityScorer]:::guard
  View[AlertView]:::view
  App[AppEndpoint]:::ep

  API -->|receive| Entity
  API -->|start| WF
  WF -->|fetchStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Fetch
  Agent -->|invokes| Enrich
  Agent -->|invokes| Ticket
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|AlertDetail / EnrichedAlert / TriageTicket| WF
  WF -->|recordAlertDetail/Enrichment/TriageTicket| Entity
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
  participant U as Operator (UI)
  participant API as AlertEndpoint
  participant E as AlertEntity
  participant W as AlertEnrichmentWorkflow
  participant A as AlertEnrichmentAgent
  participant G as TicketScopeGuardrail
  participant T as Tools (Fetch/Enrich/Ticket)
  participant Sc as TriageQualityScorer

  U->>API: POST /api/alerts { rawAlertJson }
  API->>E: receive(rawAlertJson)
  E-->>API: { alertId }
  API->>W: start(alertId)
  W->>E: startFetch
  W->>A: runSingleTask(FETCH_ALERT, alertId)
  A->>G: before-tool-call(fetchAlertDetail, FETCH)
  G-->>A: accept (status FETCHING)
  A->>T: fetchAlertDetail + fetchHostContext
  T-->>A: AlertDetail
  A-->>W: AlertDetail
  W->>E: recordAlertDetail
  W->>A: runSingleTask(ENRICH_ALERT, alertDetail)
  A->>G: before-tool-call(lookupMitreTechnique, ENRICH)
  G-->>A: accept (status ENRICHING and alertDetail present)
  A->>T: lookupMitreTechnique + mapAttackPattern
  T-->>A: List<AttackPattern>
  A-->>W: EnrichedAlert
  W->>E: recordEnrichment
  W->>A: runSingleTask(CREATE_TICKET, enrichedAlert)
  A->>G: before-tool-call(openZendeskTicket, TRIAGE)
  G-->>A: accept (status TRIAGING and enrichedAlert present)
  A->>T: openZendeskTicket + assignTicketTeam
  T-->>A: TriageTicket
  A-->>W: TriageTicket
  W->>E: recordTriageTicket
  W->>Sc: score(triageTicket, enrichedAlert, alertDetail)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `AlertEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> FETCHING: FetchStarted
  FETCHING --> FETCHED: FetchCompleted
  FETCHED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: EnrichmentProduced
  ENRICHED --> TRIAGING: TriageStarted
  TRIAGING --> TRIAGED: TicketCreated
  TRIAGED --> EVALUATED: EvaluationScored
  FETCHING --> FAILED: AlertFailed
  ENRICHING --> FAILED: AlertFailed
  TRIAGING --> FAILED: AlertFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  AlertEntity ||--o{ AlertReceived : emits
  AlertEntity ||--o{ FetchStarted : emits
  AlertEntity ||--o{ FetchCompleted : emits
  AlertEntity ||--o{ EnrichStarted : emits
  AlertEntity ||--o{ EnrichmentProduced : emits
  AlertEntity ||--o{ TriageStarted : emits
  AlertEntity ||--o{ TicketCreated : emits
  AlertEntity ||--o{ EvaluationScored : emits
  AlertEntity ||--o{ GuardrailRejected : emits
  AlertEntity ||--o{ AlertFailed : emits
  AlertView }o--|| AlertEntity : projects
  AlertEnrichmentWorkflow }o--|| AlertEntity : reads-and-writes
  AlertEnrichmentAgent ||--o{ AlertDetail : returns
  AlertEnrichmentAgent ||--o{ EnrichedAlert : returns
  AlertEnrichmentAgent ||--o{ TriageTicket : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AlertEndpoint` | `api/AlertEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AlertEntity` | `application/AlertEntity.java` (state in `domain/AlertRecord.java`, events in `domain/AlertEvent.java`) |
| `AlertEnrichmentWorkflow` | `application/AlertEnrichmentWorkflow.java` |
| `AlertEnrichmentAgent` | `application/AlertEnrichmentAgent.java` (tasks in `application/AlertTasks.java`) |
| `FetchTools` | `application/FetchTools.java` |
| `EnrichTools` | `application/EnrichTools.java` |
| `TicketTools` | `application/TicketTools.java` |
| `TicketScopeGuardrail` | `application/TicketScopeGuardrail.java` |
| `TriageQualityScorer` | `application/TriageQualityScorer.java` |
| `AlertView` | `application/AlertView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `fetchStep` 60 s, `enrichStep` 60 s, `ticketStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AlertEnrichmentWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + alertId` as the workflow id; restart of the same alertId is rejected by the workflow runtime. The agent instance id is `"agent-" + alertId` so each alert has its own per-task conversation memory.
- **One agent per alert**: `AlertEnrichmentAgent` runs three tasks per alert — FETCH, ENRICH, TRIAGE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `TicketScopeGuardrail` rejects a tool call, the rejection is returned as a structured error to the agent loop. If all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to FAILED.
- **Eval is synchronous and deterministic**: `TriageQualityScorer` runs in-process inside `evalStep`. No LLM call — the same enriched alert always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `fetchStep` writes `FetchCompleted` BEFORE returning; `enrichStep` reads the recorded `AlertDetail` from the entity to build its task's instruction context; `ticketStep` reads both `AlertDetail` and `EnrichedAlert`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed alert stays at the last successful event; the UI shows the partial state.
- **External write boundary**: `TicketTools.openZendeskTicket` is the only tool that would make a write to an external system in a production deployment. `TicketScopeGuardrail` is the enforcement point that ensures this call never fires on an unenriched alert. In the blueprint's in-process stub, the Zendesk call returns a deterministic `ZendeskTicketRef`; a deployer swaps the stub for a real Zendesk client without changing the guardrail or workflow.
