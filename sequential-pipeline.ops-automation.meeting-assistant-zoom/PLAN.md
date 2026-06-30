# PLAN — meeting-assistant-zoom

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
  classDef san fill:#0e1f2a,stroke:#38bdf8,color:#38bdf8;

  API[MeetingEndpoint]:::ep
  Entity[MeetingEntity]:::ese
  WF[MeetingPipelineWorkflow]:::wf
  Agent[MeetingAgent]:::agent
  Transcribe[TranscribeTools]:::tool
  Summarize[SummarizeTools]:::tool
  Dispatch[DispatchTools]:::tool
  Guard[ScopeGuardrail]:::guard
  Pii[PiiSanitizer]:::san
  Scorer[ActionItemScorer]:::guard
  View[MeetingView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|sanitize first| Pii
  Pii -->|SanitizedTranscript| WF
  WF -->|transcribeStep runSingleTask| Agent
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Transcribe
  Agent -->|invokes| Summarize
  Agent -->|invokes| Dispatch
  Guard -->|recordScopeRejection| Entity
  Agent -->|Transcript / MeetingSummary / MeetingPackage| WF
  WF -->|recordTranscript/Summary/Package| Entity
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
  participant API as MeetingEndpoint
  participant E as MeetingEntity
  participant W as MeetingPipelineWorkflow
  participant P as PiiSanitizer
  participant A as MeetingAgent
  participant G as ScopeGuardrail
  participant T as Tools (Transcribe/Summarize/Dispatch)
  participant Sc as ActionItemScorer

  U->>API: POST /api/meetings { meetingTitle, rawTranscript }
  API->>E: create(meetingTitle, rawTranscript)
  E-->>API: { meetingId }
  API->>W: start(meetingId, meetingTitle)
  W->>P: sanitize(rawTranscript)
  P-->>W: SanitizedTranscript { text, redactionAudit }
  W->>E: startTranscribe(redactionAudit)
  W->>A: runSingleTask(TRANSCRIBE_MEETING, sanitizedText)
  A->>G: before-tool-call(segmentTranscript, TRANSCRIBE)
  G-->>A: accept (status TRANSCRIBING)
  A->>T: segmentTranscript + labelSpeakers
  T-->>A: List<Segment>
  A-->>W: Transcript
  W->>E: recordTranscript
  W->>A: runSingleTask(SUMMARIZE_MEETING, transcript)
  A->>G: before-tool-call(extractActionItems, SUMMARIZE)
  G-->>A: accept (status SUMMARIZING and transcript present)
  A->>T: extractActionItems + identifyDecisions
  T-->>A: List<ActionItem> / List<Decision>
  A-->>W: MeetingSummary
  W->>E: recordSummary
  W->>A: runSingleTask(DISPATCH_FOLLOWUPS, summary)
  A->>G: before-tool-call(scheduleFollowUp, DISPATCH)
  G-->>A: accept (status DISPATCHING and summary present)
  A->>T: scheduleFollowUp + createTask
  T-->>A: FollowUpEvent / TaskRecord
  A-->>W: MeetingPackage
  W->>E: recordPackage
  W->>Sc: score(meetingPackage, summary, transcript)
  Sc-->>W: EvalResult
  W->>E: recordEvaluation
  E-.->>U: SSE event(EVALUATED)
```

## State machine — `MeetingEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> TRANSCRIBING: TranscribeStarted
  TRANSCRIBING --> TRANSCRIBED: TranscriptProduced
  TRANSCRIBED --> SUMMARIZING: SummarizeStarted
  SUMMARIZING --> SUMMARIZED: SummaryProduced
  SUMMARIZED --> DISPATCHING: DispatchStarted
  DISPATCHING --> DISPATCHED: FollowUpsDispatched
  DISPATCHED --> EVALUATED: EvaluationScored
  TRANSCRIBING --> FAILED: MeetingFailed
  SUMMARIZING --> FAILED: MeetingFailed
  DISPATCHING --> FAILED: MeetingFailed
  EVALUATED --> [*]
  FAILED --> [*]
```

ScopeRejected is a side-event recorded on the entity for audit; it does not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  MeetingEntity ||--o{ MeetingCreated : emits
  MeetingEntity ||--o{ TranscribeStarted : emits
  MeetingEntity ||--o{ TranscriptProduced : emits
  MeetingEntity ||--o{ SummarizeStarted : emits
  MeetingEntity ||--o{ SummaryProduced : emits
  MeetingEntity ||--o{ DispatchStarted : emits
  MeetingEntity ||--o{ FollowUpsDispatched : emits
  MeetingEntity ||--o{ EvaluationScored : emits
  MeetingEntity ||--o{ ScopeRejected : emits
  MeetingEntity ||--o{ MeetingFailed : emits
  MeetingView }o--|| MeetingEntity : projects
  MeetingPipelineWorkflow }o--|| MeetingEntity : reads-and-writes
  MeetingAgent ||--o{ Transcript : returns
  MeetingAgent ||--o{ MeetingSummary : returns
  MeetingAgent ||--o{ MeetingPackage : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `MeetingEndpoint` | `api/MeetingEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `MeetingEntity` | `application/MeetingEntity.java` (state in `domain/MeetingRecord.java`, events in `domain/MeetingEvent.java`) |
| `MeetingPipelineWorkflow` | `application/MeetingPipelineWorkflow.java` |
| `MeetingAgent` | `application/MeetingAgent.java` (tasks in `application/MeetingTasks.java`) |
| `TranscribeTools` | `application/TranscribeTools.java` |
| `SummarizeTools` | `application/SummarizeTools.java` |
| `DispatchTools` | `application/DispatchTools.java` |
| `ScopeGuardrail` | `application/ScopeGuardrail.java` |
| `PiiSanitizer` | `application/PiiSanitizer.java` |
| `ActionItemScorer` | `application/ActionItemScorer.java` |
| `MeetingView` | `application/MeetingView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `transcribeStep` 90 s, `summarizeStep` 90 s, `dispatchStep` 90 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(MeetingPipelineWorkflow::error)`. The 90 s on each agent-calling step accommodates transcript segmentation plus LLM latency (Lesson 4).
- **Idempotency**: each workflow uses `"pipeline-" + meetingId` as the workflow id; restart of the same meetingId is rejected by the workflow runtime. The agent instance id is `"agent-" + meetingId` so each meeting has its own per-task conversation memory.
- **One agent per meeting**: `MeetingAgent` runs three tasks per meeting — TRANSCRIBE, SUMMARIZE, DISPATCH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget gives the guardrail room to reject a misordered or out-of-scope call and still let the agent self-correct.
- **PiiSanitizer runs before the agent**: `transcribeStep` calls `PiiSanitizer.sanitize(rawTranscript)` synchronously before handing the sanitized text to `runSingleTask`. The LLM never sees the original. The redaction audit is recorded on the entity with `TranscribeStarted` so it is auditable independently of the transcript result.
- **Eval is synchronous and deterministic**: `ActionItemScorer` runs in-process inside `evalStep`. No LLM call, no external service — the same meeting package always scores the same.
- **Task-boundary handoff is the dependency contract**: `transcribeStep` writes `TranscriptProduced` BEFORE returning; `summarizeStep` reads the recorded `Transcript` from the entity to build its task's instruction context; `dispatchStep` reads both `Transcript` and `MeetingSummary`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed meeting stays at the last successful event; the UI shows the partial state.
