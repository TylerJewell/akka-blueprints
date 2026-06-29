# PLAN — akka-llm-pipeline-workflow

Architectural sketch consumed by `/akka:plan` and rendered on the generated system's Architecture tab. The four mermaid diagrams below carry the theme variables and CSS overrides from Lesson 24; without them, state names render black-on-black and edge labels clip.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#0e1e2a','primaryTextColor':'#ffffff','primaryBorderColor':'#7EC8E3','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
flowchart TB
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef activity fill:#251503,stroke:#F97316,color:#F97316;
  classDef ep fill:#161616,stroke:#fff,color:#fff;
  classDef guard fill:#2a0e0e,stroke:#ff5f57,color:#ff5f57;

  API[PostEndpoint]:::ep
  Entity[PostEntity]:::ese
  WF[ContentWorkflow]:::wf
  OA[OutlineActivity]:::activity
  DA[DraftActivity]:::activity
  SG[SchemaGuardrail]:::guard
  EG[EditorialGuardrail]:::guard
  View[PostView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|outlineStep| OA
  OA -.->|after-llm-response| SG
  SG -->|recordValidationFailure| Entity
  OA -->|PostOutline| WF
  WF -->|recordOutline| Entity
  WF -->|draftStep reads outline| Entity
  WF -->|draftStep| DA
  DA -.->|before-agent-response| EG
  EG -->|flag or approve| Entity
  DA -->|BlogPost| WF
  WF -->|approve or flag| Entity
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
  participant API as PostEndpoint
  participant E as PostEntity
  participant W as ContentWorkflow
  participant OA as OutlineActivity
  participant SG as SchemaGuardrail
  participant DA as DraftActivity
  participant EG as EditorialGuardrail

  U->>API: POST /api/posts { topic, wordTarget }
  API->>E: create(topic, wordTarget)
  E-->>API: { postId }
  API->>W: start(postId, topic, wordTarget)
  W->>E: startOutline
  W->>OA: call(topic, wordTarget)
  OA-->>SG: after-llm-response(PostOutline)
  SG-->>OA: accept (all fields valid)
  OA-->>W: PostOutline
  W->>E: recordOutline(outline)
  W->>E: startDraft (reads outline)
  W->>DA: call(PostOutline)
  DA-->>EG: before-agent-response(BlogPost)
  EG-->>DA: accept (all rules passed)
  DA-->>W: BlogPost
  W->>E: startReview
  W->>E: approve(review)
  E-.->>U: SSE event(READY)
```

## State machine — `PostEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> OUTLINING: OutlineStarted
  OUTLINING --> OUTLINED: OutlineProduced
  OUTLINED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftProduced
  DRAFTED --> REVIEWING: ReviewStarted
  REVIEWING --> READY: PostApproved
  REVIEWING --> FLAGGED: PostFlagged
  OUTLINING --> FAILED: ValidationFailed
  OUTLINING --> FAILED: PostFailed
  DRAFTING --> FAILED: PostFailed
  REVIEWING --> FAILED: PostFailed
  READY --> [*]
  FLAGGED --> [*]
  FAILED --> [*]
```

`ValidationFailed` is recorded on the entity when `SchemaGuardrail` rejects the outline; it transitions the post directly to `FAILED` because the outline cannot be recovered within the same workflow run. `PostFlagged` transitions to `FLAGGED` (a distinct terminal state — the draft exists but requires editorial attention). `FAILED` and `FLAGGED` are both terminal; neither has a resume path.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  PostEntity ||--o{ PostCreated : emits
  PostEntity ||--o{ OutlineStarted : emits
  PostEntity ||--o{ OutlineProduced : emits
  PostEntity ||--o{ DraftStarted : emits
  PostEntity ||--o{ DraftProduced : emits
  PostEntity ||--o{ ReviewStarted : emits
  PostEntity ||--o{ PostApproved : emits
  PostEntity ||--o{ PostFlagged : emits
  PostEntity ||--o{ ValidationFailed : emits
  PostEntity ||--o{ PostFailed : emits
  PostView }o--|| PostEntity : projects
  ContentWorkflow }o--|| PostEntity : reads-and-writes
  OutlineActivity ||--o{ PostOutline : returns
  DraftActivity ||--o{ BlogPost : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `PostEndpoint` | `api/PostEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `PostEntity` | `application/PostEntity.java` (state in `domain/PostRecord.java`, events in `domain/PostEvent.java`) |
| `ContentWorkflow` | `application/ContentWorkflow.java` |
| `OutlineActivity` | `application/OutlineActivity.java` |
| `DraftActivity` | `application/DraftActivity.java` |
| `SchemaGuardrail` | `application/SchemaGuardrail.java` |
| `EditorialGuardrail` | `application/EditorialGuardrail.java` |
| `PostView` | `application/PostView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `outlineStep` 60 s, `draftStep` 90 s, `reviewStep` 10 s, `failStep` 5 s. Default step recovery `maxRetries(2).failoverTo(ContentWorkflow::failStep)`. The 90 s on the draft step accommodates longer LLM generation for a full blog post (Lesson 4).
- **Idempotency**: each workflow uses `"workflow-" + postId` as the workflow id; restart of the same `postId` is rejected by the workflow runtime.
- **No agent**: there are zero `AutonomousAgent` instances. Both LLM calls are typed activities executed directly inside the workflow steps via the `ModelClient` API. The guardrails are registered on the activities, not on agents.
- **Typed handoff is the information contract**: `outlineStep` writes `OutlineProduced` BEFORE returning; `draftStep` reads the recorded `PostOutline` from the entity to build the draft LLM call's context. The raw topic is never passed to the draft call.
- **Guardrails are synchronous**: `SchemaGuardrail` runs in-process before `outlineStep` writes the outline. `EditorialGuardrail` runs in-process before `reviewStep` finalises the status. Neither makes an LLM call; both are rule-based and deterministic.
- **FLAGGED vs FAILED**: `FAILED` means the pipeline could not produce a valid output (e.g., schema validation error). `FLAGGED` means a complete output was produced but it did not pass editorial policy. Both are terminal; the distinction is surfaced in the UI with different card colours.
- **No saga / no compensation**: each step is either a pure read, an append-only event write, or a single typed LLM call. A failed post stays at the last successful event; the UI shows the partial state.
