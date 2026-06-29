# PLAN — incident-management

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'fontFamily':'Instrument Sans, system-ui, sans-serif'
}}}%%
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef autonomous fill:#0e2a1e,stroke:#3fb950,color:#3fb950;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  Feeder[IncidentFeeder]:::ta
  Queue[IncidentQueue]:::ese
  Enricher[ContextEnricher]:::cons
  Classifier[ClassifierAgent]:::agent
  Infra[InfraSpecialist]:::autonomous
  App[AppSpecialist]:::autonomous
  Judge[RoutingJudge]:::agent
  Guard[ToolCallGuardrail]:::agent
  WF[IncidentWorkflow]:::wf
  Entity[IncidentEntity]:::ese
  View[IncidentView]:::view
  Scorer[RoutingEvalScorer]:::cons
  API[IncidentEndpoint]:::ep
  AppEP[AppEndpoint]:::ep

  Feeder -.->|every 30s| Queue
  Queue -.->|subscribes| Enricher
  Enricher -->|register + enrich| Entity
  Enricher -->|start workflow| WF
  WF -->|classify| Classifier
  WF -->|INVESTIGATE task| Infra
  WF -->|INVESTIGATE task| App
  WF -->|check plan| Guard
  WF -->|emit events| Entity
  Entity -.->|projects| View
  Entity -.->|subscribes| Scorer
  Scorer -->|score| Judge
  Scorer -->|recordRoutingScore| Entity
  API -->|unblock / submit| Entity
  API -->|query / SSE| View
  AppEP -->|static UI + metadata| API
```

Solid arrows = synchronous component calls. Dashed arrows = event subscriptions and scheduler ticks.

## Interaction sequence — J1 (infrastructure happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888','secondaryColor':'#141414','tertiaryColor':'#1C1C1C',
  'nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc',
  'actorTextColor':'#ffffff','noteTextColor':'#ffffff','sequenceNumberColor':'#000'
}}}%%
sequenceDiagram
  autonumber
  participant Fd as IncidentFeeder
  participant Q as IncidentQueue
  participant En as ContextEnricher
  participant E as IncidentEntity
  participant W as IncidentWorkflow
  participant Cl as ClassifierAgent
  participant In as InfraSpecialist
  participant Gu as ToolCallGuardrail
  participant Sc as RoutingEvalScorer
  participant Ju as RoutingJudge

  Fd->>Q: receive(InboundReport)
  Q->>En: InboundReportReceived
  En->>E: registerReport + attachEnriched
  En->>W: start(incidentId, enriched)
  W->>Cl: classify(enriched)
  Cl-->>W: ClassificationDecision{INFRASTRUCTURE}
  W->>E: recordClassification(decision) [emits IncidentClassified]
  E->>Sc: IncidentClassified event
  Sc->>Ju: score(enriched, decision)
  Ju-->>Sc: RoutingScore
  Sc->>E: recordRoutingScore [emits RoutingScored]
  W->>E: recordRouting(INFRASTRUCTURE) [emits IncidentRouted]
  W->>In: runSingleTask(INVESTIGATE, prompt)
  In-->>W: RemediationPlan
  W->>E: recordPlan(plan) [emits PlanDrafted]
  W->>Gu: check(enriched, plan)
  Gu-->>W: GuardrailVerdict{allowed=true}
  W->>E: publish(plan) [emits RemediationPublished, status RESOLVED]
```

The eval-event sequence (steps 7–10) runs concurrently with the workflow's continuation — `RoutingEvalScorer` is a Consumer reading the entity's event stream, independent of `IncidentWorkflow`. Both writes target the same `IncidentEntity`; the entity's commands are idempotent on `incidentId`.

## State machine — `IncidentEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518',
  'lineColor':'#cccccc','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff',
  'transitionLabelColor':'#cccccc'
}}}%%
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> ENRICHED: IncidentEnriched
  ENRICHED --> CLASSIFIED: IncidentClassified
  CLASSIFIED --> ROUTED_INFRA: category = INFRASTRUCTURE
  CLASSIFIED --> ROUTED_APP: category = APPLICATION
  CLASSIFIED --> ESCALATED: category = AMBIGUOUS
  ROUTED_INFRA --> PLAN_DRAFTED: PlanDrafted
  ROUTED_APP --> PLAN_DRAFTED: PlanDrafted
  PLAN_DRAFTED --> RESOLVED: guardrail.allowed
  PLAN_DRAFTED --> BLOCKED: guardrail.rejected
  BLOCKED --> RESOLVED: operator unblock
  RESOLVED --> [*]
  ESCALATED --> [*]
```

The `RoutingScored` event does not change `status`; it attaches the eval result. The state machine therefore treats it as a no-op transition (omitted from the diagram for clarity).

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{
  'primaryColor':'#0A0A0A','primaryTextColor':'#ffffff','primaryBorderColor':'#222',
  'lineColor':'#888'
}}}%%
erDiagram
  IncidentEntity ||--o{ IncidentRegistered : emits
  IncidentEntity ||--o{ IncidentEnriched : emits
  IncidentEntity ||--o{ IncidentClassified : emits
  IncidentEntity ||--o{ IncidentRouted : emits
  IncidentEntity ||--o{ PlanDrafted : emits
  IncidentEntity ||--o{ GuardrailVerdictAttached : emits
  IncidentEntity ||--o{ RemediationPublished : emits
  IncidentEntity ||--o{ RemediationBlocked : emits
  IncidentEntity ||--o{ IncidentEscalated : emits
  IncidentEntity ||--o{ RoutingScored : emits
  IncidentView }o--|| IncidentEntity : projects
  IncidentQueue ||--o{ InboundReportReceived : emits
  ContextEnricher }o--|| IncidentQueue : subscribes
  RoutingEvalScorer }o--|| IncidentEntity : subscribes
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `IncidentFeeder` | `application/IncidentFeeder.java` |
| `IncidentQueue` | `application/IncidentQueue.java` |
| `ContextEnricher` | `application/ContextEnricher.java` |
| `ClassifierAgent` | `application/ClassifierAgent.java` |
| `InfraSpecialist` | `application/InfraSpecialist.java` |
| `AppSpecialist` | `application/AppSpecialist.java` |
| `RoutingJudge` | `application/RoutingJudge.java` |
| `ToolCallGuardrail` | `application/ToolCallGuardrail.java` |
| `IncidentWorkflow` | `application/IncidentWorkflow.java` |
| `IncidentEntity` | `application/IncidentEntity.java` (state in `domain/Incident.java`, events in `domain/IncidentEvent.java`) |
| `IncidentView` | `application/IncidentView.java` |
| `RoutingEvalScorer` | `application/RoutingEvalScorer.java` |
| `IncidentEndpoint` | `api/IncidentEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| Task definitions | `application/IncidentTasks.java` |
| Mock provider (option a) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout.** `classifyStep` 20 s, `guardrailStep` 20 s, `infraStep` / `appStep` / `publishStep` 60 s each. On timeout, default recovery is `maxRetries(2).failoverTo(error)` which transitions the incident to `ESCALATED` with the failure reason captured.
- **Idempotency.** Every per-incident primitive is keyed by `incidentId`: `IncidentEntity` id is `incidentId`; `IncidentWorkflow` id is `incidentId`; agent sessions for `ClassifierAgent`, `RoutingJudge`, and `ToolCallGuardrail` use `incidentId`. Duplicate enrich events fold into a single workflow start (workflow start is idempotent per id).
- **Race between eval and workflow.** `RoutingEvalScorer` (Consumer) and `IncidentWorkflow` both append events to the same `IncidentEntity`. Order is not guaranteed but does not matter: `RoutingScored` only mutates `routingScore`, never `status`. The view materialises both events independently.
- **No saga compensation.** The handoff is a single-direction transfer of ownership; once the specialist returns its `RemediationPlan`, the workflow either publishes or blocks based on the guardrail verdict. There is no rollback path — a blocked plan sits in `BLOCKED` until an operator unblocks via `POST /api/incidents/{id}/unblock`.
- **No HITL on the happy path.** This is the distinction from `human-in-loop-gate`. The system only waits for a human when the guardrail blocks; everything else flows through to `RESOLVED` autonomously.
- **Feeder throughput.** `IncidentFeeder` drips one report every 30 s; the system can process each incident end-to-end inside that window with mock or real LLMs.
