# PLAN — financial-advisor-pipeline

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

  API[AdvisoryEndpoint]:::ep
  Entity[AdvisoryEntity]:::ese
  WF[AdvisorPipelineWorkflow]:::wf
  Agent[FinancialAdvisorAgent]:::agent
  Research[ResearchTools]:::tool
  Strategy[StrategyTools]:::tool
  Execution[ExecutionTools]:::tool
  Risk[RiskTools]:::tool
  Guard[DisclaimerGuardrail]:::guard
  Sanitizer[SectorSanitizer]:::sanitizer
  Scorer[ComplianceScorer]:::guard
  View[AdvisoryView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|researchStep runSingleTask| Agent
  WF -->|strategyStep runSingleTask| Agent
  WF -->|executionStep runSingleTask| Agent
  WF -->|riskStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard -.->|after disclaimer| Sanitizer
  Guard -->|DisclaimerInjected| Entity
  Sanitizer -->|SanitizerFired| Entity
  Agent -->|invokes| Research
  Agent -->|invokes| Strategy
  Agent -->|invokes| Execution
  Agent -->|invokes| Risk
  Agent -->|MarketSnapshot / Strategy / ExecutionPlan / RiskProfile| WF
  WF -->|recordSnapshot/Strategy/Plan/Risk| Entity
  WF -->|complianceScore| Scorer
  Scorer -->|ComplianceResult| WF
  WF -->|recordCompliance| Entity
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
  participant API as AdvisoryEndpoint
  participant E as AdvisoryEntity
  participant W as AdvisorPipelineWorkflow
  participant A as FinancialAdvisorAgent
  participant G as DisclaimerGuardrail
  participant S as SectorSanitizer
  participant T as Tools (Research/Strategy/Execution/Risk)
  participant Sc as ComplianceScorer

  U->>API: POST /api/advisories { query }
  API->>E: create(query)
  E-->>API: { advisoryId }
  API->>W: start(advisoryId, query)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_MARKET, query)
  A->>T: fetchMarketData + lookupBenchmark
  T-->>A: List<MarketDataPoint> / benchmarkReturn
  A-->>G: outbound response (MarketSnapshot)
  G->>E: DisclaimerInjected
  G-->>S: response + disclaimer
  S-->>W: MarketSnapshot (sanitized)
  W->>E: recordSnapshot
  W->>A: runSingleTask(DEFINE_STRATEGY, snapshot)
  A->>T: buildAllocationTargets + scoreRationale
  T-->>A: List<AllocationTarget> / List<RationaleItem>
  A-->>G: outbound response (Strategy)
  G->>E: DisclaimerInjected
  G-->>S: response + disclaimer
  S-->>W: Strategy (sanitized)
  W->>E: recordStrategy
  W->>A: runSingleTask(PLAN_EXECUTION, strategy)
  A->>T: sequenceActions + resolveInstruments
  T-->>A: List<ActionItem> / List<Instrument>
  A-->>G: outbound response (ExecutionPlan)
  G->>E: DisclaimerInjected
  G-->>S: response + disclaimer
  S-->>W: ExecutionPlan (sanitized)
  W->>E: recordExecutionPlan
  W->>A: runSingleTask(ASSESS_RISK, executionPlan)
  A->>T: measureVolatility + classifyRiskBand
  T-->>A: VolatilityMetrics / RiskBand
  A-->>G: outbound response (RiskProfile)
  G->>E: DisclaimerInjected
  G-->>S: response + disclaimer
  S-->>W: RiskProfile (sanitized)
  W->>E: recordRiskProfile + recordReport
  W->>Sc: score(report, riskProfile, strategy)
  Sc-->>W: ComplianceResult
  W->>E: recordCompliance
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `AdvisoryEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RESEARCHING: ResearchStarted
  RESEARCHING --> RESEARCHED: MarketResearched
  RESEARCHED --> STRATEGIZING: StrategyStarted
  STRATEGIZING --> STRATEGIZED: StrategyDefined
  STRATEGIZED --> PLANNING: ExecutionStarted
  PLANNING --> PLANNED: ExecutionPlanned
  PLANNED --> ASSESSING: RiskStarted
  ASSESSING --> EVALUATED: ComplianceScored
  RESEARCHING --> FAILED: AdvisoryFailed
  STRATEGIZING --> FAILED: AdvisoryFailed
  PLANNING --> FAILED: AdvisoryFailed
  ASSESSING --> FAILED: AdvisoryFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

`DisclaimerInjected` and `SanitizerFired` are side-events recorded on the entity for audit; they do not change the `AdvisoryStatus`. A failed advisory preserves the partial state from all completed phases so the UI can show what was collected before the failure.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  AdvisoryEntity ||--o{ AdvisoryCreated : emits
  AdvisoryEntity ||--o{ ResearchStarted : emits
  AdvisoryEntity ||--o{ MarketResearched : emits
  AdvisoryEntity ||--o{ StrategyStarted : emits
  AdvisoryEntity ||--o{ StrategyDefined : emits
  AdvisoryEntity ||--o{ ExecutionStarted : emits
  AdvisoryEntity ||--o{ ExecutionPlanned : emits
  AdvisoryEntity ||--o{ RiskStarted : emits
  AdvisoryEntity ||--o{ RiskAssessed : emits
  AdvisoryEntity ||--o{ ReportAssembled : emits
  AdvisoryEntity ||--o{ ComplianceScored : emits
  AdvisoryEntity ||--o{ DisclaimerInjected : emits
  AdvisoryEntity ||--o{ SanitizerFired : emits
  AdvisoryEntity ||--o{ AdvisoryFailed : emits
  AdvisoryView }o--|| AdvisoryEntity : projects
  AdvisorPipelineWorkflow }o--|| AdvisoryEntity : reads-and-writes
  FinancialAdvisorAgent ||--o{ MarketSnapshot : returns
  FinancialAdvisorAgent ||--o{ Strategy : returns
  FinancialAdvisorAgent ||--o{ ExecutionPlan : returns
  FinancialAdvisorAgent ||--o{ RiskProfile : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AdvisoryEndpoint` | `api/AdvisoryEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AdvisoryEntity` | `application/AdvisoryEntity.java` (state in `domain/AdvisoryRecord.java`, events in `domain/AdvisoryEvent.java`) |
| `AdvisorPipelineWorkflow` | `application/AdvisorPipelineWorkflow.java` |
| `FinancialAdvisorAgent` | `application/FinancialAdvisorAgent.java` (tasks in `application/AdvisorTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `StrategyTools` | `application/StrategyTools.java` |
| `ExecutionTools` | `application/ExecutionTools.java` |
| `RiskTools` | `application/RiskTools.java` |
| `DisclaimerGuardrail` | `application/DisclaimerGuardrail.java` |
| `SectorSanitizer` | `application/SectorSanitizer.java` |
| `ComplianceScorer` | `application/ComplianceScorer.java` |
| `AdvisoryView` | `application/AdvisoryView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 70 s, `strategyStep` 70 s, `executionStep` 70 s, `riskStep` 70 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AdvisorPipelineWorkflow::error)`. The 70 s on each agent-calling step accommodates LLM latency plus the disclaimer guardrail and sanitizer overhead (Lesson 4).
- **Idempotency**: each workflow uses `"advisory-" + advisoryId` as the workflow id; restart of the same advisoryId is rejected by the workflow runtime. The agent instance id is `"agent-" + advisoryId` so each advisory has its own per-task conversation memory.
- **One agent per advisory**: `FinancialAdvisorAgent` runs four tasks per advisory — RESEARCH, STRATEGY, EXECUTE, ASSESS — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget accommodates transient tool failures without letting the agent loop indefinitely.
- **Governance is output-side, not input-side**: unlike a phase-gate guardrail that checks tool-call order, `DisclaimerGuardrail` and `SectorSanitizer` operate on every outbound agent response, regardless of which task produced it. They fire on all four phases.
- **Eval is synchronous and deterministic**: `ComplianceScorer` runs in-process inside `riskStep` after the risk profile is recorded. No LLM call — the same report always scores the same.
- **Task-boundary handoff**: `researchStep` writes `MarketResearched` BEFORE advancing; `strategyStep` reads the recorded `MarketSnapshot` from the entity to build its instruction context; `executionStep` reads `Strategy`; `riskStep` reads all three. The agent is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed advisory stays at the last successful event; the UI shows the partial state.
- **Disclaimer audit completeness**: because `DisclaimerGuardrail` records `DisclaimerInjected` on the entity for every outbound response, the advisory's `disclaimerLog` always has exactly as many entries as there were successful phase responses. A missing entry is a gap the audit trail surfaces.
