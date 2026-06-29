# PLAN — pitch-builder

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
  classDef sanitizer fill:#0e2a1e,stroke:#34d399,color:#34d399;

  API[PitchbookEndpoint]:::ep
  Entity[PitchbookEntity]:::ese
  WF[PitchbookPipelineWorkflow]:::wf
  Agent[PitchAgent]:::agent
  Research[ResearchTools]:::tool
  Comps[ComparablesTools]:::tool
  Draft[DraftTools]:::tool
  Sanitizer[PiiSanitizer]:::sanitizer
  Validator[CitationValidator]:::guard
  View[PitchbookView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|researchStep runSingleTask| Agent
  Agent -->|invokes| Research
  Agent -->|invokes| Comps
  Agent -->|invokes| Draft
  WF -->|sanitize ResearchPack| Sanitizer
  Sanitizer -->|PiiRedactionLogged| Entity
  Agent -.->|before-agent-response| Validator
  Validator -->|logCitationRejection| Entity
  Agent -->|ResearchPack / CompsTable / Pitchbook| WF
  WF -->|recordResearch/Comps/Draft| Entity
  WF -->|validationStep score| Validator
  Validator -->|ValidationResult| WF
  WF -->|recordValidation| Entity
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
  participant API as PitchbookEndpoint
  participant E as PitchbookEntity
  participant W as PitchbookPipelineWorkflow
  participant A as PitchAgent
  participant T as Tools (Research/Comps/Draft)
  participant S as PiiSanitizer
  participant V as CitationValidator

  U->>API: POST /api/pitchbooks { target }
  API->>E: create(target)
  E-->>API: { pitchbookId }
  API->>W: start(pitchbookId, target)
  W->>E: startResearch
  W->>A: runSingleTask(RESEARCH_TARGET, target)
  A->>T: searchFilings + fetchHeadline
  T-->>A: List<RawResearchItem>
  A-->>W: ResearchPack (raw)
  W->>S: sanitize(rawPack)
  S->>E: logPiiRedaction(field, patternId, ...)
  S-->>W: ResearchPack (redacted)
  W->>E: recordResearch(redactedPack)
  W->>A: runSingleTask(RUN_COMPARABLES, researchPack)
  A->>T: selectPeers + buildMultiplesTable
  T-->>A: CompsTable
  A-->>W: CompsTable
  W->>E: recordComps
  W->>A: runSingleTask(DRAFT_PITCHBOOK, comps)
  A->>T: formatSection + buildCoverPage
  T-->>A: Pitchbook (draft)
  A->>V: before-agent-response(Pitchbook)
  V-->>A: accept (all citations valid)
  A-->>W: Pitchbook
  W->>E: recordDraft
  W->>V: validationStep score(pitchbook, comps)
  V-->>W: ValidationResult
  W->>E: recordValidation
  E-.->>U: SSE event(VALIDATED)
```

## State machine — `PitchbookEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RESEARCHING: ResearchStarted
  RESEARCHING --> RESEARCHED: TargetResearched
  RESEARCHED --> COMPS_RUNNING: CompsStarted
  COMPS_RUNNING --> COMPS_READY: ComparablesBuilt
  COMPS_READY --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftWritten
  DRAFTED --> VALIDATED: ValidationScored
  RESEARCHING --> FAILED: PitchbookFailed
  COMPS_RUNNING --> FAILED: PitchbookFailed
  DRAFTING --> FAILED: PitchbookFailed
  VALIDATED --> [*]
  FAILED --> [*]
```

`PiiRedactionLogged` and `CitationRejected` are side-events recorded on the entity for audit; they do not change the status — the PII sanitizer runs once per `researchStep`, and the agent's retry stays inside the same `draftStep` task. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  PitchbookEntity ||--o{ PitchbookCreated : emits
  PitchbookEntity ||--o{ ResearchStarted : emits
  PitchbookEntity ||--o{ TargetResearched : emits
  PitchbookEntity ||--o{ CompsStarted : emits
  PitchbookEntity ||--o{ ComparablesBuilt : emits
  PitchbookEntity ||--o{ DraftStarted : emits
  PitchbookEntity ||--o{ DraftWritten : emits
  PitchbookEntity ||--o{ ValidationScored : emits
  PitchbookEntity ||--o{ PiiRedactionLogged : emits
  PitchbookEntity ||--o{ CitationRejected : emits
  PitchbookEntity ||--o{ PitchbookFailed : emits
  PitchbookView }o--|| PitchbookEntity : projects
  PitchbookPipelineWorkflow }o--|| PitchbookEntity : reads-and-writes
  PitchAgent ||--o{ ResearchPack : returns
  PitchAgent ||--o{ CompsTable : returns
  PitchAgent ||--o{ Pitchbook : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PitchbookEndpoint` | `api/PitchbookEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PitchbookEntity` | `application/PitchbookEntity.java` (state in `domain/PitchbookRecord.java`, events in `domain/PitchbookEvent.java`) |
| `PitchbookPipelineWorkflow` | `application/PitchbookPipelineWorkflow.java` |
| `PitchAgent` | `application/PitchAgent.java` (tasks in `application/PitchTasks.java`) |
| `ResearchTools` | `application/ResearchTools.java` |
| `ComparablesTools` | `application/ComparablesTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `PiiPatternRegistry` | `application/PiiPatternRegistry.java` |
| `CitationValidator` | `application/CitationValidator.java` |
| `PitchbookView` | `application/PitchbookView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `researchStep` 60 s, `comparablesStep` 60 s, `draftStep` 60 s, `validationStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(PitchbookPipelineWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + pitchbookId` as the workflow id; restart of the same pitchbookId is rejected by the workflow runtime. The agent instance id is `"agent-" + pitchbookId` so each pitchbook has its own per-task conversation memory.
- **One agent per pitchbook**: `PitchAgent` runs three tasks per pitchbook — RESEARCH, COMPARABLES, DRAFT — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the citation guardrail room to reject a misordered draft and still let the agent self-correct.
- **PII sanitizer is synchronous**: `PiiSanitizer.sanitize()` runs in-process inside `researchStep` between the agent returning and the entity write. It is not an LLM call; latency is bounded by the number of `RawResearchItem` entries (typically < 10 ms).
- **Validation is synchronous and deterministic**: `CitationValidator` runs in-process inside `validationStep`. No LLM call — the same pitchbook always scores the same. This is a deliberate single-agent invariant.
- **Task-boundary handoff is the dependency contract**: `researchStep` writes `TargetResearched` (with the sanitized pack) BEFORE returning; `comparablesStep` reads the recorded `ResearchPack` from the entity to build its task's instruction context; `draftStep` reads both. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed pitchbook stays at the last successful event; the UI shows the partial state for the user.
