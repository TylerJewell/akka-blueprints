# PLAN — inbound-lead-qualification

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
  classDef sanitizer fill:#1a0e2a,stroke:#c084fc,color:#c084fc;

  API[LeadEndpoint]:::ep
  Entity[LeadEntity]:::ese
  WF[LeadPipelineWorkflow]:::wf
  Agent[LeadAgent]:::agent
  Enrich[EnrichTools]:::tool
  Qualify[QualifyTools]:::tool
  Notify[NotifyTools]:::tool
  Guard[SlackGuardrail]:::guard
  Sanitizer[PiiSanitizer]:::sanitizer
  Scorer[QualificationEvaluator]:::guard
  View[LeadView]:::view
  App[AppEndpoint]:::ep

  API -->|submit| Entity
  API -->|start| WF
  WF -->|enrichStep runSingleTask| Agent
  Sanitizer -.->|sanitize instruction| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Enrich
  Agent -->|invokes| Qualify
  Agent -->|invokes| Notify
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|FirmographicProfile / QualificationScore / SlackPayload| WF
  WF -->|recordProfile/Score/Notification| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|EvalRecord| WF
  WF -->|recordEval| Entity
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
  participant API as LeadEndpoint
  participant E as LeadEntity
  participant W as LeadPipelineWorkflow
  participant San as PiiSanitizer
  participant A as LeadAgent
  participant G as SlackGuardrail
  participant T as Tools (Enrich/Qualify/Notify)
  participant Sc as QualificationEvaluator

  U->>API: POST /api/leads { formData }
  API->>E: submit(formData)
  E-->>API: { leadId }
  API->>W: start(leadId, formData)
  W->>E: startEnrich
  W->>San: sanitize(formData instructions)
  San-->>W: anonymized instruction string
  W->>A: runSingleTask(ENRICH_LEAD, sanitized context)
  A->>G: before-tool-call(lookupFirmographics, ENRICH)
  G-->>A: accept (non-Slack tool)
  A->>T: lookupFirmographics + fetchTechStack
  T-->>A: FirmographicProfile
  A-->>W: FirmographicProfile
  W->>E: recordProfile
  W->>A: runSingleTask(QUALIFY_LEAD, profile + formData)
  A->>G: before-tool-call(scoreByRubric, QUALIFY)
  G-->>A: accept (non-Slack tool)
  A->>T: scoreByRubric + assignRep
  T-->>A: QualificationScore
  A-->>W: QualificationScore
  W->>E: recordScore
  W->>A: runSingleTask(NOTIFY_SALES, score + profile)
  A->>G: before-tool-call(postLeadToSlack, NOTIFY)
  G-->>A: accept (status QUALIFIED, score present)
  A->>T: buildSlackMessage + postLeadToSlack
  T-->>A: SlackPayload
  A-->>W: SlackPayload
  W->>E: recordNotification
  W->>Sc: evaluate(score, profile, formData)
  Sc-->>W: EvalRecord
  W->>E: recordEval
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `LeadEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: EnrichmentCompleted
  ENRICHED --> QUALIFYING: QualifyStarted
  QUALIFYING --> QUALIFIED: QualificationScored
  QUALIFIED --> NOTIFYING: NotifyStarted
  NOTIFYING --> NOTIFIED: NotificationSent
  NOTIFIED --> EVALUATED: EvaluationRecorded
  ENRICHING --> FAILED: LeadFailed
  QUALIFYING --> FAILED: LeadFailed
  NOTIFYING --> FAILED: LeadFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  LeadEntity ||--o{ LeadSubmitted : emits
  LeadEntity ||--o{ EnrichStarted : emits
  LeadEntity ||--o{ EnrichmentCompleted : emits
  LeadEntity ||--o{ QualifyStarted : emits
  LeadEntity ||--o{ QualificationScored : emits
  LeadEntity ||--o{ NotifyStarted : emits
  LeadEntity ||--o{ NotificationSent : emits
  LeadEntity ||--o{ EvaluationRecorded : emits
  LeadEntity ||--o{ GuardrailRejected : emits
  LeadEntity ||--o{ LeadFailed : emits
  LeadView }o--|| LeadEntity : projects
  LeadPipelineWorkflow }o--|| LeadEntity : reads-and-writes
  LeadAgent ||--o{ FirmographicProfile : returns
  LeadAgent ||--o{ QualificationScore : returns
  LeadAgent ||--o{ SlackPayload : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `LeadEndpoint` | `api/LeadEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `LeadEntity` | `application/LeadEntity.java` (state in `domain/LeadRecord.java`, events in `domain/LeadEvent.java`) |
| `LeadPipelineWorkflow` | `application/LeadPipelineWorkflow.java` |
| `LeadAgent` | `application/LeadAgent.java` (tasks in `application/LeadTasks.java`) |
| `EnrichTools` | `application/EnrichTools.java` |
| `QualifyTools` | `application/QualifyTools.java` |
| `NotifyTools` | `application/NotifyTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `SlackGuardrail` | `application/SlackGuardrail.java` |
| `QualificationEvaluator` | `application/QualificationEvaluator.java` |
| `LeadView` | `application/LeadView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| `MockSlackClient` (no token) | `application/MockSlackClient.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `enrichStep` 60 s, `qualifyStep` 60 s, `notifyStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(LeadPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + leadId` as the workflow id; restart of the same leadId is rejected by the workflow runtime. The agent instance id is `"agent-" + leadId` so each lead has its own per-task conversation memory.
- **One agent per lead**: `LeadAgent` runs three tasks per lead — ENRICH, QUALIFY, NOTIFY — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered Slack write and still let the agent self-correct.
- **Guardrail-driven retry**: when `SlackGuardrail` rejects `postLeadToSlack`, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to FAILED.
- **PII sanitizer is synchronous**: `PiiSanitizer` runs in the calling thread before the instruction string is forwarded to the agent. The pseudonymous tokens are stable per `leadId`; re-runs of the same lead produce the same tokens.
- **Eval is synchronous and deterministic**: `QualificationEvaluator` runs in-process inside `evalStep`. No LLM call — the same score and profile always produce the same confidence.
- **Task-boundary handoff is the dependency contract**: `enrichStep` writes `EnrichmentCompleted` BEFORE returning; `qualifyStep` reads the recorded `FirmographicProfile` from the entity to build its task's instruction context; `notifyStep` reads both `FirmographicProfile` and `QualificationScore`. The agent itself is stateless across phases.
- **Slack write is external**: `postLeadToSlack` makes an outbound HTTP call to the Slack API. `SlackGuardrail` ensures this call cannot fire until the qualified score is present. `MockSlackClient` substitutes when `SLACK_BOT_TOKEN` is absent.
