# PLAN — lead-qualifier

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
  classDef sanitizer fill:#1a0a1a,stroke:#d946ef,color:#d946ef;

  API[LeadEndpoint]:::ep
  Entity[LeadEntity]:::ese
  WF[LeadPipelineWorkflow]:::wf
  Agent[InquiryAgent]:::agent
  Capture[CaptureTools]:::tool
  Qualify[QualifyTools]:::tool
  Enrich[EnrichTools]:::tool
  Guard[CrmWriteGuardrail]:::guard
  Sanitizer[PiiSanitizer]:::sanitizer
  Scorer[QualityScorer]:::guard
  View[LeadView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|captureStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Guard -->|sanitize PII| Sanitizer
  Agent -->|invokes| Capture
  Agent -->|invokes| Qualify
  Agent -->|invokes| Enrich
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|InquiryForm / LeadScore / CrmEntry| WF
  WF -->|recordForm/Score/CrmEntry| Entity
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
  participant U as User (UI)
  participant API as LeadEndpoint
  participant E as LeadEntity
  participant W as LeadPipelineWorkflow
  participant A as InquiryAgent
  participant G as CrmWriteGuardrail
  participant PS as PiiSanitizer
  participant T as Tools (Capture/Qualify/Enrich)
  participant Sc as QualityScorer

  U->>API: POST /api/leads { rawInquiry }
  API->>E: create(rawInquiry)
  E-->>API: { leadId }
  API->>W: start(leadId, rawInquiry)
  W->>E: startCapture
  W->>A: runSingleTask(CAPTURE_INQUIRY, rawInquiry)
  A->>G: before-tool-call(parseContact, CAPTURE)
  G-->>A: accept (status CAPTURING)
  A->>T: parseContact + detectIntent
  T-->>A: ContactFields + IntentSignals
  A-->>W: InquiryForm
  W->>E: recordForm
  W->>A: runSingleTask(QUALIFY_LEAD, form)
  A->>G: before-tool-call(scoreFit, QUALIFY)
  G-->>A: accept (status QUALIFYING and form present)
  A->>T: scoreFit + classifyUrgency
  T-->>A: FitScore + UrgencyTier
  A-->>W: LeadScore
  W->>E: recordScore
  W->>A: runSingleTask(ENRICH_CRM, score+form)
  A->>G: before-tool-call(writeCrmStage, ENRICH)
  G->>PS: sanitize PII in payload
  PS-->>G: masked payload
  G-->>A: accept (status ENRICHING, schema valid)
  A->>T: writeCrmStage + assignOwner
  T-->>A: CrmStageResult + OwnerAssignment
  A-->>W: CrmEntry
  W->>E: recordCrmEntry
  W->>Sc: score(crmEntry, leadScore, form)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `LeadEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> CAPTURING: CaptureStarted
  CAPTURING --> CAPTURED: InquiryCaptured
  CAPTURED --> QUALIFYING: QualifyStarted
  QUALIFYING --> QUALIFIED: QualificationScored
  QUALIFIED --> ENRICHING: EnrichStarted
  ENRICHING --> ENRICHED: CrmEntryWritten
  ENRICHED --> EVALUATED: EvaluationScored
  CAPTURING --> FAILED: LeadFailed
  QUALIFYING --> FAILED: LeadFailed
  ENRICHING --> FAILED: LeadFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

GuardrailRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  LeadEntity ||--o{ LeadCreated : emits
  LeadEntity ||--o{ CaptureStarted : emits
  LeadEntity ||--o{ InquiryCaptured : emits
  LeadEntity ||--o{ QualifyStarted : emits
  LeadEntity ||--o{ QualificationScored : emits
  LeadEntity ||--o{ EnrichStarted : emits
  LeadEntity ||--o{ CrmEntryWritten : emits
  LeadEntity ||--o{ EvaluationScored : emits
  LeadEntity ||--o{ GuardrailRejected : emits
  LeadEntity ||--o{ LeadFailed : emits
  LeadView }o--|| LeadEntity : projects
  LeadPipelineWorkflow }o--|| LeadEntity : reads-and-writes
  InquiryAgent ||--o{ InquiryForm : returns
  InquiryAgent ||--o{ LeadScore : returns
  InquiryAgent ||--o{ CrmEntry : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `LeadEndpoint` | `api/LeadEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `LeadEntity` | `application/LeadEntity.java` (state in `domain/LeadRecord.java`, events in `domain/LeadEvent.java`) |
| `LeadPipelineWorkflow` | `application/LeadPipelineWorkflow.java` |
| `InquiryAgent` | `application/InquiryAgent.java` (tasks in `application/LeadTasks.java`) |
| `CaptureTools` | `application/CaptureTools.java` |
| `QualifyTools` | `application/QualifyTools.java` |
| `EnrichTools` | `application/EnrichTools.java` |
| `CrmWriteGuardrail` | `application/CrmWriteGuardrail.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `QualityScorer` | `application/QualityScorer.java` |
| `LeadView` | `application/LeadView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `captureStep` 60 s, `qualifyStep` 60 s, `enrichStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(LeadPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + leadId` as the workflow id; restart of the same leadId is rejected by the workflow runtime. The agent instance id is `"agent-" + leadId` so each lead has its own per-task conversation memory.
- **One agent per lead**: `InquiryAgent` runs three tasks per lead — CAPTURE, QUALIFY, ENRICH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered or schema-invalid tool call and still let the agent self-correct.
- **Guardrail-driven retry**: when `CrmWriteGuardrail` rejects a tool call (phase-violation or schema-violation), the rejection is returned as a structured error to the agent loop. If all 4 iterations fail, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **PII masking is synchronous**: `PiiSanitizer` runs in-process inside `CrmWriteGuardrail` before the tool body executes. No async path exists between guardrail accept and tool invocation, so no unmasked payload can escape even on race conditions.
- **Eval is synchronous and deterministic**: `QualityScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same CRM entry always scores the same. Single-agent invariant is preserved.
- **Task-boundary handoff is the dependency contract**: `captureStep` writes `InquiryCaptured` BEFORE returning; `qualifyStep` reads the recorded `InquiryForm` from the entity to build its task's instruction context; `enrichStep` reads both `InquiryForm` and `LeadScore`. The agent is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed lead stays at the last successful event; the UI shows partial state.
