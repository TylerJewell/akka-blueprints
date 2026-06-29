# PLAN — meeting-to-tasks

Architecture this blueprint resolves to once `SPEC.md` runs through `/akka:specify` → `/akka:plan`. All four mermaid diagrams and the component table are required. The generated UI renders these diagrams on the Architecture tab with the Lesson 24 CSS overrides.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef helper fill:#241016,stroke:#f472b6,color:#f472b6;

  SIM[NotesSimulator]:::ta
  EP[MeetingEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[MeetingPipelineWorkflow]:::wf
  AG[TaskExtractionAgent]:::agent
  ENT[MeetingEntity]:::ese
  VIEW[MeetingsView]:::view
  TRELLO[TrelloClient]:::helper
  SLACK[SlackClient]:::helper
  CSV[CsvWriter]:::helper
  PII[PiiSanitizer]:::helper

  SIM -->|starts workflow| WF
  EP -->|POST meetings starts| WF
  EP -->|approve / reject| ENT
  WF -->|sanitize| PII
  WF -->|extract| AG
  WF -->|push cards| TRELLO
  WF -->|export| CSV
  WF -->|notify| SLACK
  WF -->|record events| ENT
  ENT -.->|events| VIEW
  EP -->|list + SSE| VIEW
  APP -->|static UI| EP
```

Solid arrows are synchronous command calls; the dashed arrow is the event subscription that feeds the view.

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  participant U as User / UI
  participant EP as MeetingEndpoint
  participant WF as MeetingPipelineWorkflow
  participant PII as PiiSanitizer
  participant AG as TaskExtractionAgent
  participant ENT as MeetingEntity
  participant TR as TrelloClient
  participant SL as SlackClient

  U->>EP: POST /api/meetings {title, rawNotes}
  EP->>ENT: receive
  EP->>WF: start(meetingId)
  WF->>PII: redact(rawNotes)
  WF->>ENT: recordSanitized(notes, redactionCount)
  WF->>AG: extract(sanitizedNotes)
  AG-->>WF: ExtractedTasks
  WF->>ENT: recordExtraction(tasks)
  Note over WF,ENT: status AWAITING_APPROVAL — pipeline pauses, polls every 5s
  U->>EP: POST /api/meetings/{id}/approve
  EP->>ENT: approve
  WF->>ENT: getMeeting (poll picks up APPROVED)
  WF->>TR: createCards (after before-tool-call guardrail)
  WF->>SL: postMessage (after before-tool-call guardrail)
  WF->>ENT: recordCompletion
  ENT-->>U: SSE meeting COMPLETED
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED
  RECEIVED --> SANITIZED: NotesSanitized
  SANITIZED --> EXTRACTED: TasksExtracted
  EXTRACTED --> AWAITING_APPROVAL
  AWAITING_APPROVAL --> APPROVED: MeetingApproved
  AWAITING_APPROVAL --> REJECTED: MeetingRejected
  APPROVED --> COMPLETED: board + csv + slack done
  RECEIVED --> FAILED: MeetingFailed
  SANITIZED --> FAILED: MeetingFailed
  EXTRACTED --> FAILED: MeetingFailed
  APPROVED --> FAILED: MeetingFailed
  REJECTED --> [*]
  COMPLETED --> [*]
  FAILED --> [*]
```

CSS overrides (Lesson 24) the generated `index.html` must carry so the state names and transition labels stay visible:

```css
.diagram-card .mermaid .nodeLabel,
.diagram-card .mermaid .stateLabel,
.diagram-card .mermaid g.statediagram-state .label,
.diagram-card .mermaid g.statediagram-state .label *,
.diagram-card .mermaid g.statediagram-state text,
.diagram-card .mermaid .label foreignObject p { color:#fff !important; fill:#fff !important; }
.diagram-card .mermaid .edgeLabel foreignObject { overflow:visible !important; }
```

Plus `nodeTextColor`, `stateLabelColor`, and `transitionLabelColor: #cccccc` in `mermaid.initialize({themeVariables})`.

## Entity model

```mermaid
erDiagram
  MEETING ||--o{ MEETING_EVENT : emits
  MEETING ||--|| MEETINGS_VIEW_ROW : projects
  MEETING {
    string id
    string title
    string rawNotes
    enum status
    string receivedAt
    string sanitizedNotes
    int redactionCount
    list extractedTasks
    string trelloBoardUrl
    string csvPath
    string slackMessageTs
  }
  MEETING_EVENT {
    string type
    string meetingId
    string timestamp
  }
  TASK_ITEM {
    string title
    string assignee
    string dueDate
    string priority
  }
  MEETING ||--o{ TASK_ITEM : contains
```

## Component table

| Component | Akka primitive | Path (generated) |
|---|---|---|
| TaskExtractionAgent | Agent | `application/TaskExtractionAgent.java` |
| MeetingPipelineWorkflow | Workflow | `application/MeetingPipelineWorkflow.java` |
| MeetingEntity | EventSourcedEntity | `application/MeetingEntity.java` |
| MeetingsView | View | `application/MeetingsView.java` |
| NotesSimulator | TimedAction | `application/NotesSimulator.java` |
| MeetingEndpoint | HttpEndpoint | `api/MeetingEndpoint.java` |
| AppEndpoint | HttpEndpoint | `api/AppEndpoint.java` |
| PiiSanitizer | helper | `application/PiiSanitizer.java` |
| TrelloClient | helper | `application/TrelloClient.java` |
| SlackClient | helper | `application/SlackClient.java` |
| CsvWriter | helper | `application/CsvWriter.java` |
| Meeting / MeetingStatus / MeetingEvent | domain | `domain/*.java` |
| Bootstrap | service-setup | `Bootstrap.java` |

Component count from the validator: **2 http-endpoint · 1 timed-action · 1 view · 1 workflow · 1 service-setup · 1 agent · 1 event-sourced-entity**.

## Concurrency notes

- **Step timeouts (Lesson 4).** `extractStep` calls the LLM; override `settings()` with `stepTimeout(MeetingPipelineWorkflow::extractStep, ofSeconds(60))`. `awaitApprovalStep` uses a short 10s step timeout and self-schedules a 5s resume poll while status is `AWAITING_APPROVAL`. `WorkflowSettings` is nested in `Workflow` — no import (Lesson 5).
- **Idempotency.** The workflow is keyed by `meetingId` (a fresh UUID per run). Each entity command is naturally idempotent on its event; re-delivery of `recordBoardPush` / `recordSlack` overwrites the same Optional fields with equal values.
- **Compensation.** The pipeline is forward-only. If `pushToBoardStep` or `notifyStep` fails after the guardrail passes, `defaultStepRecovery(maxRetries(2).failoverTo(error))` runs; the `error` step calls `MeetingEntity.markFailed(reason)` and the run ends in `FAILED`. No partial rollback of already-created Trello cards is attempted in the baseline — the failure reason records how far the run got.
- **Guardrail placement.** The before-tool-call guardrail runs inside `pushToBoardStep` and `notifyStep` immediately before the external client call, so a malformed or un-redacted payload blocks the write rather than failing after it.
