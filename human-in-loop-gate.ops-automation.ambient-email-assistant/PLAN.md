# PLAN — ambient-email-assistant

Architectural sketch for Ambient Email Assistant. All four mermaid diagrams plus the component table.

---

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  EP[EmailEndpoint]:::ep
  APP[AppEndpoint]:::ep
  WF[EmailWorkflow]:::wf
  TA[TriageAgent]:::agent
  RA[ReplyDraftAgent]:::agent
  MA[MeetingSchedulerAgent]:::agent
  TE[EmailThreadEntity]:::ese
  TV[ThreadsView]:::view

  EP -->|email-threads| WF
  WF -->|triage task| TA
  WF -->|reply-draft task| RA
  WF -->|meeting task| MA
  TA -->|recordTriage| TE
  RA -->|recordDraft| TE
  MA -->|recordDraft| TE
  EP -->|approve / dismiss| TE
  WF -->|poll status| TE
  WF -->|recordActionComplete| TE
  TE -.->|events| TV
  EP -->|getAllThreads / SSE| TV
  APP -->|static UI| EP
```

## Interaction sequence

```mermaid
sequenceDiagram
  autonumber
  actor Operator
  participant EP as EmailEndpoint
  participant WF as EmailWorkflow
  participant TA as TriageAgent
  participant RA as ReplyDraftAgent
  participant TE as EmailThreadEntity

  Operator->>EP: POST /api/email-threads {sender, subject, body}
  EP->>WF: start(threadId, emailPayload)
  WF->>TA: runSingleTask(TRIAGE)
  TA-->>WF: EmailClassification{category, urgency, suggestedAction}
  WF->>TE: recordTriage -> TRIAGED
  WF->>RA: runSingleTask(REPLY_DRAFT) [if suggestedAction=reply]
  RA-->>WF: ReplyDraft{subject, body}
  WF->>TE: recordDraft -> ACTION_DRAFTED
  Note over WF,TE: await-approval task paused; workflow polls status every 5s
  Operator->>EP: POST /api/threads/{id}/approve
  EP->>TE: approve -> APPROVED
  WF->>TE: getThread -> APPROVED
  WF->>TE: recordActionComplete [guard: status == APPROVED] -> REPLY_SENT
```

## State machine

```mermaid
stateDiagram-v2
  [*] --> RECEIVED: ThreadReceived
  RECEIVED --> TRIAGED: ThreadTriaged
  TRIAGED --> ACTION_DRAFTED: ActionDrafted
  ACTION_DRAFTED --> APPROVED: ThreadApproved
  ACTION_DRAFTED --> DISMISSED: ThreadDismissed
  APPROVED --> REPLY_SENT: ReplySent
  APPROVED --> MEETING_SCHEDULED: MeetingScheduled
  REPLY_SENT --> [*]
  MEETING_SCHEDULED --> [*]
  DISMISSED --> [*]
```

## Entity model

```mermaid
erDiagram
  EMAIL_THREAD ||--o{ THREAD_EVENT : emits
  EMAIL_THREAD {
    string id
    string sender
    string subject
    string status
    string category
    string urgency
    string draftSubject
    string actionReference
  }
  THREAD_EVENT {
    string type
    string occurredAt
  }
  THREADS_VIEW {
    string id
    string status
    string category
  }
  EMAIL_THREAD ||--|| THREADS_VIEW : projects
```

## Component table

| Component | Path (generated) |
|---|---|
| TriageAgent | `application/TriageAgent.java` |
| ReplyDraftAgent | `application/ReplyDraftAgent.java` |
| MeetingSchedulerAgent | `application/MeetingSchedulerAgent.java` |
| EmailWorkflow | `application/EmailWorkflow.java` |
| EmailTasks | `application/EmailTasks.java` |
| EmailThreadEntity | `application/EmailThreadEntity.java` |
| ThreadsView | `application/ThreadsView.java` |
| EmailEndpoint | `api/EmailEndpoint.java` |
| AppEndpoint | `api/AppEndpoint.java` |
| EmailThread / events / records | `domain/*.java` |

## Concurrency notes

- **Step timeouts.** `triageStep`, `draftActionStep`, and `executeActionStep` call agents; all three set `stepTimeout(60s)` to absorb LLM and integration latency. The default 5 s step timeout would retry before the agent returns (Lesson 4).
- **Await-approval task.** The workflow does not block a thread; `awaitApprovalStep` reads `EmailThreadEntity.getThread`, and while status is `ACTION_DRAFTED` it self-schedules a 5-second resume timer until the operator transitions the status.
- **Branch on suggestedAction.** `draftActionStep` reads the `EmailClassification` returned by `triageStep` and routes to `ReplyDraftAgent` or `MeetingSchedulerAgent`. Unrecognised action types default to `ReplyDraftAgent`.
- **Action guard.** Before any Gmail send or Calendar create tool runs, the before-tool-call guardrail re-reads `EmailThreadEntity.status`; if it is not `APPROVED`, the call is blocked and the workflow moves the thread to an error-acknowledged state.
- **Idempotency.** `threadId` is the workflow id and the entity id; re-delivery of `recordTriage` or `recordDraft` is absorbed by event-applier checks on current status.
