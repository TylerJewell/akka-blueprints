# PLAN — traced-agent-otel

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
  classDef support fill:#1a1a2e,stroke:#818cf8,color:#818cf8;

  API[ConversationEndpoint]:::ep
  Entity[ConversationEntity]:::ese
  Collector[SpanCollector]:::cons
  WF[TraceExportWorkflow]:::wf
  Agent[TracedConversationAgent]:::agent
  Instr[SpanInstrumentor]:::support
  Monitor[PerformanceMonitor]:::support
  Exporter[OtlpExporter]:::support
  View[ConversationView]:::view
  App[AppEndpoint]:::ep

  API -->|submit + start workflow| Entity
  API -->|start| WF
  Entity -.->|AgentRunCompleted| Collector
  Collector -->|attachSpanSummary| Entity
  WF -->|agentStep runSingleTask| Agent
  Agent -->|spans| Instr
  Instr -->|SpanRecord| Agent
  Agent -->|AgentResponse| WF
  WF -->|recordAgentResponse| Entity
  WF -->|spanFlushStep| Exporter
  Exporter -->|FlushResult| WF
  WF -->|recordFlush| Entity
  WF -->|monitorStep| Monitor
  Monitor -->|PerformanceMonitorResult| WF
  WF -->|recordMonitorResult| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as User (UI)
  participant API as ConversationEndpoint
  participant E as ConversationEntity
  participant W as TraceExportWorkflow
  participant A as TracedConversationAgent
  participant I as SpanInstrumentor
  participant C as SpanCollector
  participant Ex as OtlpExporter
  participant M as PerformanceMonitor

  U->>API: POST /api/conversations
  API->>E: submit(request)
  API->>W: start(conversationId)
  E-->>API: { conversationId }
  W->>E: markRunning
  W->>A: runSingleTask(prompt)
  A->>I: before-LLM hook
  I-->>A: SpanRecord (LLM_CALL start)
  A->>A: LLM call
  A->>I: after-LLM hook
  I-->>A: SpanRecord (LLM_CALL end)
  A-->>W: AgentResponse{answerText, spans}
  W->>E: recordAgentResponse(response)
  E-.->>C: AgentRunCompleted
  C->>C: aggregate SpanSummary
  C->>E: attachSpanSummary(summary)
  W->>Ex: flush(spans)
  Ex-->>W: FlushResult{exported, healthy, url}
  W->>E: recordFlush(flushResult)
  W->>M: score(spanSummary)
  M-->>W: PerformanceMonitorResult
  W->>E: recordMonitorResult(result)
  E-.->>U: SSE event(MONITORED)
```

## State machine — `ConversationEntity`

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED
  SUBMITTED --> RUNNING: AgentRunStarted
  RUNNING --> COMPLETED: AgentRunCompleted
  COMPLETED --> FLUSHED: TraceFlushed
  FLUSHED --> MONITORED: PerformanceMonitorScored
  RUNNING --> FAILED: ConversationFailed (agent error)
  COMPLETED --> FAILED: ConversationFailed (flush step max retries)
  MONITORED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  ConversationEntity ||--o{ PromptSubmitted : emits
  ConversationEntity ||--o{ AgentRunStarted : emits
  ConversationEntity ||--o{ AgentRunCompleted : emits
  ConversationEntity ||--o{ SpanSummaryAttached : emits
  ConversationEntity ||--o{ TraceFlushed : emits
  ConversationEntity ||--o{ PerformanceMonitorScored : emits
  ConversationEntity ||--o{ ConversationFailed : emits
  ConversationView }o--|| ConversationEntity : projects
  SpanCollector }o--|| ConversationEntity : subscribes
  TraceExportWorkflow }o--|| ConversationEntity : reads-and-writes
  TracedConversationAgent ||--o{ AgentResponse : returns
  AgentResponse ||--|{ SpanRecord : contains
  SpanSummary ||--|{ SpanRecord : aggregates
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `ConversationEndpoint` | `api/ConversationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `ConversationEntity` | `application/ConversationEntity.java` (state in `domain/Conversation.java`, events in `domain/ConversationEvent.java`) |
| `SpanCollector` | `application/SpanCollector.java` |
| `TraceExportWorkflow` | `application/TraceExportWorkflow.java` |
| `TracedConversationAgent` | `application/TracedConversationAgent.java` (tasks in `application/ConversationTasks.java`) |
| `SpanInstrumentor` | `application/SpanInstrumentor.java` |
| `PerformanceMonitor` | `application/PerformanceMonitor.java` |
| `OtlpExporter` / `ExporterPort` | `application/OtlpExporter.java`, `application/ExporterPort.java` |
| `ConversationView` | `application/ConversationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `agentStep` 90 s, `spanFlushStep` 10 s, `monitorStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(TraceExportWorkflow::error)`. The 90 s on `agentStep` accommodates LLM latency for multi-turn conversations (Lesson 4).
- **Non-blocking flush**: `spanFlushStep` advances to `monitorStep` even when `exporterHealthy = false`. The FlushResult carries the failure detail; the deployer monitor reads it without stalling the conversation lifecycle.
- **Idempotency**: `"conv-" + conversationId` is the workflow id. `SpanCollector.attachSpanSummary` is event-version-guarded — a redelivery of `AgentRunCompleted` is a no-op if a `SpanSummaryAttached` event already exists.
- **One agent per conversation**: the AutonomousAgent instance id is `"agent-" + conversationId`. Each conversation gets its own context; `maxIterationsPerTask(5)` gives the agent room for multi-turn tool use within a single task.
- **SpanInstrumentor is not an Akka component**: it is a plain Java helper. The agent holds a reference and calls it synchronously. No component client, no entity state — just a builder that produces `SpanRecord` POJOs.
- **PerformanceMonitor is synchronous and deterministic**: runs in-process inside `monitorStep`. No LLM call. Same `SpanSummary` always produces the same score — the single-agent invariant holds.
