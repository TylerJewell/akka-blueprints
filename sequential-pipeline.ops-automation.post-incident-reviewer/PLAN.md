# PLAN — post-incident-reviewer

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
  classDef notifier fill:#0e1a2a,stroke:#60a5fa,color:#60a5fa;

  API[PIREndpoint]:::ep
  Entity[PIREntity]:::ese
  WF[PIRWorkflow]:::wf
  Agent[PIRAgent]:::agent
  Gather[GatherTools]:::tool
  Assess[AssessTools]:::tool
  Draft[DraftTools]:::tool
  Guard[PIRGuardrail]:::guard
  Notifier[PIRSignoffNotifier]:::notifier
  View[PIRView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|gatherStep runSingleTask| Agent
  Agent -->|invokes| Gather
  Agent -->|invokes| Assess
  Agent -->|invokes| Draft
  Agent -.->|before-agent-response| Guard
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|EvidenceLog / ImpactAssessment / PostIncidentReview| WF
  WF -->|recordEvidence/Assessment/Review| Entity
  WF -->|signoffStep| Notifier
  Notifier -->|notify| WF
  API -->|POST signoff| WF
  WF -->|complete/reject| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
sequenceDiagram
  autonumber
  participant Op as Operator (UI)
  participant API as PIREndpoint
  participant E as PIREntity
  participant W as PIRWorkflow
  participant A as PIRAgent
  participant G as PIRGuardrail
  participant T as Tools (Gather/Assess/Draft)
  participant N as PIRSignoffNotifier

  Op->>API: POST /api/pir { incidentId }
  API->>E: create(incidentId)
  E-->>API: { pirId }
  API->>W: start(pirId, incidentId)
  W->>E: startGather
  W->>A: runSingleTask(GATHER_EVIDENCE, incidentId)
  A->>T: fetchIncidentRecord + fetchTimelineEvents
  T-->>A: IncidentRecord + List<TimelineEvent>
  A-->>W: EvidenceLog
  W->>E: recordEvidence
  W->>A: runSingleTask(ASSESS_IMPACT, evidenceLog)
  A->>T: classifyImpact + identifyRootCause
  T-->>A: ImpactClassification + RootCause
  A-->>W: ImpactAssessment
  W->>E: recordAssessment
  W->>A: runSingleTask(DRAFT_REVIEW, impactAssessment)
  A->>T: composeExecutiveSummary + buildActionItems
  T-->>A: String + List<ActionItem>
  A->>G: before-agent-response(PostIncidentReview)
  G-->>A: accept (all 4 checks pass)
  A-->>W: PostIncidentReview
  W->>E: recordReview
  W->>N: notify(pirId, assigneeEmail)
  W->>E: requestSignoff(assigneeEmail)
  Note over W: signoffStep — workflow pauses
  Op->>API: POST /api/pir/{pirId}/signoff { approved: true }
  API->>E: recordSignoff
  API->>W: resumeSignoff
  W->>E: complete
  E-.->>Op: SSE event(COMPLETE)
```

## State machine — `PIREntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> GATHERING: GatherStarted
  GATHERING --> GATHERED: EvidenceGathered
  GATHERED --> ASSESSING: AssessStarted
  ASSESSING --> ASSESSED: ImpactAssessed
  ASSESSED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: ReviewDrafted
  DRAFTED --> AWAITING_SIGNOFF: SignoffRequested
  AWAITING_SIGNOFF --> COMPLETE: ReviewComplete
  AWAITING_SIGNOFF --> REJECTED: ReviewRejected
  GATHERING --> FAILED: PIRFailed
  ASSESSING --> FAILED: PIRFailed
  DRAFTING --> FAILED: PIRFailed
  COMPLETE --> [*]
  REJECTED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's draftStep continues. Only an exhausted retry budget or a step timeout transitions to FAILED. The AWAITING_SIGNOFF state waits indefinitely for a human decision; it never times out.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  PIREntity ||--o{ PIRCreated : emits
  PIREntity ||--o{ GatherStarted : emits
  PIREntity ||--o{ EvidenceGathered : emits
  PIREntity ||--o{ AssessStarted : emits
  PIREntity ||--o{ ImpactAssessed : emits
  PIREntity ||--o{ DraftStarted : emits
  PIREntity ||--o{ ReviewDrafted : emits
  PIREntity ||--o{ GuardrailRejected : emits
  PIREntity ||--o{ SignoffRequested : emits
  PIREntity ||--o{ SignoffRecorded : emits
  PIREntity ||--o{ ReviewComplete : emits
  PIREntity ||--o{ ReviewRejected : emits
  PIREntity ||--o{ PIRFailed : emits
  PIRView }o--|| PIREntity : projects
  PIRWorkflow }o--|| PIREntity : reads-and-writes
  PIRAgent ||--o{ EvidenceLog : returns
  PIRAgent ||--o{ ImpactAssessment : returns
  PIRAgent ||--o{ PostIncidentReview : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PIREndpoint` | `api/PIREndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PIREntity` | `application/PIREntity.java` (state in `domain/PIRRecord.java`, events in `domain/PIREvent.java`) |
| `PIRWorkflow` | `application/PIRWorkflow.java` |
| `PIRAgent` | `application/PIRAgent.java` (tasks in `application/PIRTasks.java`) |
| `GatherTools` | `application/GatherTools.java` |
| `AssessTools` | `application/AssessTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `PIRGuardrail` | `application/PIRGuardrail.java` |
| `PIRSignoffNotifier` | `application/PIRSignoffNotifier.java` |
| `PIRView` | `application/PIRView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `gatherStep` 60 s, `assessStep` 60 s, `draftStep` 90 s (extra budget for guardrail retries), `error` 5 s. `signoffStep` has no timeout — human accountability requires a human decision. Default step recovery `maxRetries(2).failoverTo(PIRWorkflow::error)`.
- **Idempotency**: each workflow uses `"pir-" + pirId` as the workflow id; restart of the same pirId is rejected by the workflow runtime. The agent instance id is `"agent-" + pirId`.
- **One agent per review**: `PIRAgent` runs three tasks per review — GATHER, ASSESS, DRAFT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a non-compliant draft and still let the agent self-correct.
- **Guardrail-driven retry**: when `PIRGuardrail` rejects a draft response, the rejection is returned as a structured error to the agent loop. The agent retries within its iteration budget; if all 4 iterations fail, `draftStep` fails over to the `error` step.
- **HITL sign-off is indefinite**: `signoffStep` pauses the workflow thread. It does not consume resources while waiting — the workflow state is durable on the entity. The sign-off endpoint resumes the workflow when the incident owner acts.
- **Task-boundary handoff is the dependency contract**: `gatherStep` writes `EvidenceGathered` BEFORE returning; `assessStep` reads the recorded `EvidenceLog` from the entity to build its task's instruction context; `draftStep` reads both `EvidenceLog` and `ImpactAssessment`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed review stays at the last successful event; the UI shows the partial state.
