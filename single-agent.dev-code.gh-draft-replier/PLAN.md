# PLAN — gitty

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

  API[DraftEndpoint]:::ep
  Entity[DraftEntity]:::ese
  Fetcher[ThreadFetcher]:::cons
  WF[DraftWorkflow]:::wf
  Agent[ReplyDrafterAgent]:::agent
  Guard[ToneGuardrail]:::guard
  View[DraftView]:::view
  App[AppEndpoint]:::ep

  API -->|queue| Entity
  Entity -.->|ThreadQueued| Fetcher
  Fetcher -->|GitHub REST| Fetcher
  Fetcher -->|attachThread| Entity
  Fetcher -->|start workflow| WF
  WF -->|awaitThreadStep poll| Entity
  WF -->|draftStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Guard ---->|accept/reject| Agent
  Agent -->|DraftReply| WF
  WF -->|recordDraft| Entity
  WF -->|pendingReviewStep wait| Entity
  API -->|approve/discard| WF
  WF -->|approveDraft/discardDraft| Entity
  Entity -.->|projects| View
  API -->|list/SSE| View
  App -->|static| API
```

## Interaction sequence — J1 (happy path)

```mermaid
sequenceDiagram
  autonumber
  participant U as Maintainer (UI)
  participant API as DraftEndpoint
  participant E as DraftEntity
  participant F as ThreadFetcher
  participant W as DraftWorkflow
  participant A as ReplyDrafterAgent
  participant G as ToneGuardrail

  U->>API: POST /api/drafts
  API->>E: queue(request)
  E-->>API: { draftId }
  E-.->>F: ThreadQueued
  F->>F: GitHub REST GET /issues/{n} + /comments
  F->>E: attachThread(thread)
  F->>W: start(draftId)
  W->>E: poll getDraft
  E-->>W: thread.isPresent()
  W->>E: markDrafting
  W->>A: runSingleTask(contextNote + attachment)
  A->>G: before-agent-response(candidate)
  G-->>A: accept
  A-->>W: DraftReply
  W->>E: recordDraft(draft)
  W-->>U: SSE event(PENDING_REVIEW)
  U->>API: POST /api/drafts/{id}/approve
  API->>W: approveDraft(action)
  W->>E: approveDraft(action)
  E-.->>U: SSE event(APPROVED)
```

## State machine — `DraftEntity`

```mermaid
stateDiagram-v2
  [*] --> QUEUED
  QUEUED --> THREAD_FETCHED: ThreadFetched
  THREAD_FETCHED --> DRAFTING: DraftingStarted
  DRAFTING --> PENDING_REVIEW: DraftProduced
  PENDING_REVIEW --> APPROVED: DraftApproved (maintainer)
  PENDING_REVIEW --> DISCARDED: DraftDiscarded (maintainer)
  QUEUED --> FAILED: DraftFailed (fetch error)
  DRAFTING --> FAILED: DraftFailed (agent error)
  APPROVED --> [*]
  DISCARDED --> [*]
  FAILED --> [*]
```

## Entity model

```mermaid
erDiagram
  DraftEntity ||--o{ ThreadQueued : emits
  DraftEntity ||--o{ ThreadFetched : emits
  DraftEntity ||--o{ DraftingStarted : emits
  DraftEntity ||--o{ DraftProduced : emits
  DraftEntity ||--o{ DraftApproved : emits
  DraftEntity ||--o{ DraftDiscarded : emits
  DraftEntity ||--o{ DraftFailed : emits
  DraftView }o--|| DraftEntity : projects
  ThreadFetcher }o--|| DraftEntity : subscribes
  DraftWorkflow }o--|| DraftEntity : reads-and-writes
  ReplyDrafterAgent ||--o{ DraftReply : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `DraftEndpoint` | `api/DraftEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `DraftEntity` | `application/DraftEntity.java` (state in `domain/Draft.java`, events in `domain/DraftEvent.java`) |
| `ThreadFetcher` | `application/ThreadFetcher.java` |
| `DraftWorkflow` | `application/DraftWorkflow.java` |
| `ReplyDrafterAgent` | `application/ReplyDrafterAgent.java` (tasks in `application/DraftTasks.java`) |
| `ToneGuardrail` | `application/ToneGuardrail.java` |
| `DraftView` | `application/DraftView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| `MockThreadFetcher` (option-a only) | `application/MockThreadFetcher.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `awaitThreadStep` 20 s, `draftStep` 60 s, `pendingReviewStep` no timeout (human gate), `error` 5 s. Default step recovery `maxRetries(2).failoverTo(DraftWorkflow::error)`. The 60 s on `draftStep` accommodates LLM latency (Lesson 4).
- **Idempotency**: every workflow uses `"draft-" + draftId` as the workflow id; the `ThreadFetcher` Consumer is allowed to redeliver `ThreadQueued` events because `DraftEntity.attachThread` is event-version-guarded — a second fetch attempt against an already-fetched draft is a no-op.
- **One agent per draft**: the AutonomousAgent instance id is `"drafter-" + draftId`, giving each task its own conversation context. The agent's `capability(...).maxIterationsPerTask(3)` caps guardrail-triggered retries at 3.
- **Guardrail-driven retry**: when `ToneGuardrail` rejects a candidate response, the structured error is returned to the agent loop. The loop counts toward `maxIterationsPerTask`; if all 3 iterations fail validation, `draftStep` fails over to `error` and the entity transitions to `FAILED`.
- **HITL is a workflow pause**: `pendingReviewStep` has no step timeout and waits indefinitely for a maintainer action. The workflow only advances when `DraftEndpoint` calls `DraftWorkflow.approveDraft(...)` or `DraftWorkflow.discardDraft(...)`.
- **No GitHub write in the sample**: `ThreadFetcher` reads the GitHub API; no component writes back. The `DraftApproved` event records the approved text for the deployer to post separately.
- **No saga / no compensation**: every step is either a pure read, an append-only event write, a single-task agent call, or a human wait. There is nothing external to roll back.
