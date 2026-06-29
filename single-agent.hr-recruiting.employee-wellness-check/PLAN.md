# PLAN — wellness-check-agent

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

  API[CheckInEndpoint]:::ep
  CampaignE[CampaignEntity]:::ese
  CheckInE[CheckInEntity]:::ese
  Sanitizer[ResponseSanitizer]:::cons
  WF[CheckInWorkflow]:::wf
  Agent[WellnessCheckAgent]:::agent
  Guard[ResponseGuardrail]:::guard
  Surveillance[MoraleSurveillance]:::guard
  CheckInV[CheckInView]:::view
  CampaignV[CampaignView]:::view
  App[AppEndpoint]:::ep

  API -->|schedule| CampaignE
  API -->|receive| CheckInE
  CheckInE -.->|ResponseReceived| Sanitizer
  Sanitizer -->|attachSanitized| CheckInE
  Sanitizer -->|start workflow| WF
  WF -->|awaitSanitizedStep poll| CheckInE
  WF -->|analyseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|CheckInAnalysis| WF
  WF -->|recordAnalysis| CheckInE
  WF -->|surveillanceStep score| Surveillance
  Surveillance -->|SurveillanceResult| WF
  WF -->|recordSurveillance| CheckInE
  CheckInE -.->|projects| CheckInV
  CampaignE -.->|projects| CampaignV
  API -->|list/SSE| CheckInV
  API -->|list| CampaignV
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Coordinator (UI)
  participant API as CheckInEndpoint
  participant CE as CampaignEntity
  participant CkE as CheckInEntity
  participant S as ResponseSanitizer
  participant W as CheckInWorkflow
  participant A as WellnessCheckAgent
  participant G as ResponseGuardrail
  participant Sv as MoraleSurveillance

  U->>API: POST /api/campaigns
  API->>CE: schedule(request)
  CE-->>API: { campaignId }
  U->>API: POST /api/check-ins
  API->>CkE: receive(response)
  CkE-->>API: { checkInId }
  CkE-.->>S: ResponseReceived
  S->>S: redact special-category markers
  S->>CkE: attachSanitized
  S->>W: start(checkInId)
  W->>CkE: poll getCheckIn
  CkE-->>W: sanitized.isPresent()
  W->>CkE: markAnalysing
  W->>A: runSingleTask(questions + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: CheckInAnalysis
  W->>CkE: recordAnalysis(analysis)
  W->>Sv: evaluate(campaignRow)
  Sv-->>W: SurveillanceResult
  W->>CkE: recordSurveillance(result)
  CkE-.->>U: SSE event(EVALUATED)
```

## State machine — `CheckInEntity`

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: ResponseSanitized
  SANITIZED --> ANALYSING: AnalysisStarted
  ANALYSING --> ANALYSIS_RECORDED: AnalysisRecorded
  ANALYSIS_RECORDED --> ESCALATED: CrisisEscalated (crisisFlag=true)
  ANALYSIS_RECORDED --> EVALUATED: SurveillanceEvaluated
  ESCALATED --> [*]
  EVALUATED --> [*]
  RECEIVED --> FAILED: CheckInFailed (sanitizer error)
  ANALYSING --> FAILED: CheckInFailed (agent error)
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  CheckInEntity ||--o{ ResponseReceived : emits
  CheckInEntity ||--o{ ResponseSanitized : emits
  CheckInEntity ||--o{ AnalysisStarted : emits
  CheckInEntity ||--o{ AnalysisRecorded : emits
  CheckInEntity ||--o{ CrisisEscalated : emits
  CheckInEntity ||--o{ SurveillanceEvaluated : emits
  CheckInEntity ||--o{ CheckInFailed : emits
  CampaignEntity ||--o{ CampaignScheduled : emits
  CampaignEntity ||--o{ CampaignActivated : emits
  CampaignEntity ||--o{ CampaignCompleted : emits
  CheckInView }o--|| CheckInEntity : projects
  CampaignView }o--|| CampaignEntity : projects
  CampaignView }o--|| CheckInEntity : projects
  ResponseSanitizer }o--|| CheckInEntity : subscribes
  CheckInWorkflow }o--|| CheckInEntity : reads-and-writes
  WellnessCheckAgent ||--o{ CheckInAnalysis : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `CheckInEndpoint` | `api/CheckInEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `CampaignEntity` | `application/CampaignEntity.java` (state in `domain/Campaign.java`, events in `domain/CampaignEvent.java`) |
| `CheckInEntity` | `application/CheckInEntity.java` (state in `domain/CheckIn.java`, events in `domain/CheckInEvent.java`) |
| `ResponseSanitizer` | `application/ResponseSanitizer.java` |
| `CheckInWorkflow` | `application/CheckInWorkflow.java` |
| `WellnessCheckAgent` | `application/WellnessCheckAgent.java` (tasks in `application/CheckInTasks.java`) |
| `ResponseGuardrail` | `application/ResponseGuardrail.java` |
| `MoraleSurveillance` | `application/MoraleSurveillance.java` |
| `CheckInView` | `application/CheckInView.java` |
| `CampaignView` | `application/CampaignView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitSanitizedStep` 15 s, `analyseStep` 60 s, `surveillanceStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(CheckInWorkflow::error)`. The 60 s on `analyseStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"checkin-" + checkInId` as the workflow id; `ResponseSanitizer` Consumer is allowed to redeliver `ResponseReceived` events because `CheckInEntity.attachSanitized` is event-version-guarded — a second sanitize attempt against an already-sanitized check-in is a no-op.
- **One agent per check-in**: the AutonomousAgent instance id is `"wellness-" + checkInId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Crisis at the guardrail boundary**: when `ResponseGuardrail` detects `crisisFlag == true`, it emits `CrisisEscalated` on `CheckInEntity` before returning `Guardrail.accept()`. The check-in transitions to `ESCALATED` and is removed from normal morale aggregation. The workflow still proceeds to `surveillanceStep` so the campaign's crisis count is incremented.
- **Surveillance is synchronous and deterministic**: `MoraleSurveillance` runs in-process inside `surveillanceStep`. No LLM call. The same campaign ratio always produces the same flag.
- **No saga / no compensation**: each step is either pure read, append-only event write, or a single-task agent call. Nothing external to roll back.
