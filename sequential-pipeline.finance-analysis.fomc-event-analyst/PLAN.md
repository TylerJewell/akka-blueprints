# PLAN — fomc-event-analyst

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

  API[FomcEndpoint]:::ep
  Entity[FomcEventEntity]:::ese
  WF[FomcPipelineWorkflow]:::wf
  Agent[FomcAnalystAgent]:::agent
  Gather[GatherTools]:::tool
  Interpret[InterpretTools]:::tool
  Synthesize[SynthesizeTools]:::tool
  Guard[FinancialOutputGuardrail]:::guard
  View[PolicyAnalysisView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|gatherStep runSingleTask| Agent
  WF -->|interpretStep runSingleTask| Agent
  WF -->|synthesizeStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -->|invokes| Gather
  Agent -->|invokes| Interpret
  Agent -->|invokes| Synthesize
  Guard -->|recordGuardrailRejection| Entity
  Agent -->|MarketSnapshot / PolicySignalSet / PolicyAnalysis| WF
  WF -->|recordSnapshot/SignalSet/Analysis| Entity
  WF -->|recordReview| Entity
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
  participant API as FomcEndpoint
  participant E as FomcEventEntity
  participant W as FomcPipelineWorkflow
  participant A as FomcAnalystAgent
  participant G as FinancialOutputGuardrail
  participant T as Tools (Gather/Interpret/Synthesize)

  U->>API: POST /api/analyses { eventId }
  API->>E: create(eventId)
  E-->>API: { analysisId }
  API->>W: start(analysisId, eventId)
  W->>E: startGather
  W->>A: runSingleTask(GATHER_MARKET_DATA, eventId)
  A->>T: fetchIndicators + fetchYieldCurve
  T-->>A: List<MarketIndicator> + YieldCurveSnapshot
  A-->>W: MarketSnapshot
  W->>E: recordSnapshot
  W->>A: runSingleTask(INTERPRET_POLICY_SIGNALS, snapshot)
  A->>T: extractPolicySignals + classifyRateMove
  T-->>A: List<PolicySignal> + RateMoveForecast
  A-->>W: PolicySignalSet
  W->>E: recordSignalSet
  W->>A: runSingleTask(SYNTHESIZE_ANALYSIS, signalSet)
  A->>T: draftPolicySection + compileGrounding
  T-->>A: PolicySection + List<GroundingRef>
  A-->>G: before-agent-response(PolicyAnalysis candidate)
  G-->>A: accept (all quality checks pass)
  A-->>W: PolicyAnalysis
  W->>E: recordAnalysis + recordReview(ACCEPTED)
  W->>E: OutputReviewed
  E-.->>U: SSE event(REVIEWED)
```

## State machine — `FomcEventEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> GATHERING: GatherStarted
  GATHERING --> GATHERED: MarketDataGathered
  GATHERED --> INTERPRETING: InterpretStarted
  INTERPRETING --> INTERPRETED: PolicySignalsInterpreted
  INTERPRETED --> SYNTHESIZING: SynthesizeStarted
  SYNTHESIZING --> SYNTHESIZED: AnalysisSynthesized
  SYNTHESIZED --> REVIEWED: OutputReviewed
  GATHERING --> FAILED: AnalysisFailed
  INTERPRETING --> FAILED: AnalysisFailed
  SYNTHESIZING --> FAILED: AnalysisFailed
  REVIEWED --> [*]
  FAILED --> [*]
```

`GuardrailRejected` is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same SYNTHESIZE task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  FomcEventEntity ||--o{ AnalysisCreated : emits
  FomcEventEntity ||--o{ GatherStarted : emits
  FomcEventEntity ||--o{ MarketDataGathered : emits
  FomcEventEntity ||--o{ InterpretStarted : emits
  FomcEventEntity ||--o{ PolicySignalsInterpreted : emits
  FomcEventEntity ||--o{ SynthesizeStarted : emits
  FomcEventEntity ||--o{ AnalysisSynthesized : emits
  FomcEventEntity ||--o{ OutputReviewed : emits
  FomcEventEntity ||--o{ GuardrailRejected : emits
  FomcEventEntity ||--o{ AnalysisFailed : emits
  PolicyAnalysisView }o--|| FomcEventEntity : projects
  FomcPipelineWorkflow }o--|| FomcEventEntity : reads-and-writes
  FomcAnalystAgent ||--o{ MarketSnapshot : returns
  FomcAnalystAgent ||--o{ PolicySignalSet : returns
  FomcAnalystAgent ||--o{ PolicyAnalysis : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `FomcEndpoint` | `api/FomcEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `FomcEventEntity` | `application/FomcEventEntity.java` (state in `domain/AnalysisRecord.java`, events in `domain/AnalysisEvent.java`) |
| `FomcPipelineWorkflow` | `application/FomcPipelineWorkflow.java` |
| `FomcAnalystAgent` | `application/FomcAnalystAgent.java` (tasks in `application/FomcTasks.java`) |
| `GatherTools` | `application/GatherTools.java` |
| `InterpretTools` | `application/InterpretTools.java` |
| `SynthesizeTools` | `application/SynthesizeTools.java` |
| `FinancialOutputGuardrail` | `application/FinancialOutputGuardrail.java` |
| `PolicyAnalysisView` | `application/PolicyAnalysisView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `gatherStep` 60 s, `interpretStep` 60 s, `synthesizeStep` 60 s, `reviewStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(FomcPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + analysisId` as the workflow id; restart of the same analysisId is rejected by the workflow runtime. The agent instance id is `"agent-" + analysisId` so each analysis has its own per-task conversation memory.
- **One agent per analysis**: `FomcAnalystAgent` runs three tasks per analysis — GATHER, INTERPRET, SYNTHESIZE — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject an ungrounded output and still let the agent self-correct.
- **Guardrail-driven retry**: when `FinancialOutputGuardrail` rejects the SYNTHESIZE task's response, the rejection is returned as a structured error to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed analysis stays at the last successful event; the UI shows the partial state for the user.
- **Task-boundary handoff is the dependency contract**: `gatherStep` writes `MarketDataGathered` BEFORE returning; `interpretStep` reads the recorded `MarketSnapshot` to build the INTERPRET task's instruction context; `synthesizeStep` reads both `MarketSnapshot` and `PolicySignalSet`. The agent itself is stateless across phases.
