# PLAN — social-amplifier

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

  API[AmplificationEndpoint]:::ep
  Entity[AmplificationEntity]:::ese
  WF[AmplificationWorkflow]:::wf
  Agent[AmplifierAgent]:::agent
  Parse[ParseTools]:::tool
  Draft[DraftTools]:::tool
  Publish[PublishTools]:::tool
  Guard[BrandPolicyGuardrail]:::guard
  Scorer[BrandPolicyScorer]:::guard
  View[AmplificationView]:::view
  App[AppEndpoint]:::ep

  API -->|create| Entity
  API -->|start| WF
  WF -->|parseStep runSingleTask| Agent
  Agent -.->|before-agent-response| Guard
  Agent -.->|before-tool-call| Guard
  Agent -->|invokes| Parse
  Agent -->|invokes| Draft
  Agent -->|invokes| Publish
  Guard -->|recordBrandCheckFailed| Entity
  Guard -->|recordPublishToolBlocked| Entity
  Agent -->|ParsedArticle / DraftSet / PublishedSet| WF
  WF -->|recordParsedArticle/Drafts/Published| Entity
  WF -->|evalStep score| Scorer
  Scorer -->|BrandAuditResult| WF
  WF -->|recordBrandAudit| Entity
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
  participant API as AmplificationEndpoint
  participant E as AmplificationEntity
  participant W as AmplificationWorkflow
  participant A as AmplifierAgent
  participant G as BrandPolicyGuardrail
  participant T as Tools (Parse/Draft/Publish)
  participant Sc as BrandPolicyScorer

  U->>API: POST /api/amplifications { sourceUrl, targetPlatforms }
  API->>E: create(sourceUrl, targetPlatforms)
  E-->>API: { amplificationId }
  API->>W: start(amplificationId, sourceUrl, targetPlatforms)
  W->>E: startParse
  W->>A: runSingleTask(PARSE_ARTICLE, sourceUrl)
  A->>T: fetchArticle + extractKeyMessages
  T-->>A: ParsedArticle
  A-->>W: ParsedArticle
  W->>E: recordParsedArticle
  W->>A: runSingleTask(DRAFT_POSTS, parsedArticle)
  A->>T: draftPost + applyHashtags (per platform)
  T-->>A: DraftSet
  A->>G: before-agent-response(DraftSet)
  G-->>A: accept (all brand rules pass)
  A-->>W: DraftSet
  W->>E: recordDrafts
  W->>A: runSingleTask(PUBLISH_POSTS, draftSet)
  A->>G: before-tool-call(publishToLinkedIn, PUBLISH)
  G-->>A: accept (status PUBLISHING and draftSet present)
  A->>T: publishToLinkedIn + publishToX + publishToBluesky
  T-->>A: PublishedSet
  A-->>W: PublishedSet
  W->>E: recordPublished
  W->>Sc: score(publishedSet, draftSet, platformConfigs)
  Sc-->>W: BrandAuditResult
  W->>E: recordBrandAudit
  E-.->>U: SSE event(PUBLISHED)
```

## State machine — `AmplificationEntity`

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> CREATED
  CREATED --> PARSING: ParseStarted
  PARSING --> PARSED: ArticleParsed
  PARSED --> DRAFTING: DraftStarted
  DRAFTING --> DRAFTED: DraftsProduced
  DRAFTED --> PUBLISHING: PublishStarted
  PUBLISHING --> PUBLISHED: PostsPublished
  PUBLISHED --> [*]: BrandAuditCompleted
  PARSING --> FAILED: AmplificationFailed
  DRAFTING --> FAILED: AmplificationFailed
  PUBLISHING --> FAILED: AmplificationFailed
  FAILED --> [*]
```

BrandCheckFailed and PublishToolBlocked are side-events recorded on the entity for audit; they do not change the status — the agent's retry stays inside the same task, and the workflow's step continues. Only an exhausted retry budget or a step timeout transitions to FAILED.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1f1900','primaryTextColor':'#ffffff','primaryBorderColor':'#F5C518','lineColor':'#888','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff'}}}%%
erDiagram
  AmplificationEntity ||--o{ AmplificationCreated : emits
  AmplificationEntity ||--o{ ParseStarted : emits
  AmplificationEntity ||--o{ ArticleParsed : emits
  AmplificationEntity ||--o{ DraftStarted : emits
  AmplificationEntity ||--o{ DraftsProduced : emits
  AmplificationEntity ||--o{ PublishStarted : emits
  AmplificationEntity ||--o{ PostsPublished : emits
  AmplificationEntity ||--o{ BrandAuditCompleted : emits
  AmplificationEntity ||--o{ BrandCheckFailed : emits
  AmplificationEntity ||--o{ PublishToolBlocked : emits
  AmplificationEntity ||--o{ AmplificationFailed : emits
  AmplificationView }o--|| AmplificationEntity : projects
  AmplificationWorkflow }o--|| AmplificationEntity : reads-and-writes
  AmplifierAgent ||--o{ ParsedArticle : returns
  AmplifierAgent ||--o{ DraftSet : returns
  AmplifierAgent ||--o{ PublishedSet : returns
```

## Component table — Java file targets

| Component | Path (generated) |
|---|---|
| `AmplificationEndpoint` | `api/AmplificationEndpoint.java` |
| `AppEndpoint` | `api/AppEndpoint.java` |
| `AmplificationEntity` | `application/AmplificationEntity.java` (state in `domain/AmplificationRecord.java`, events in `domain/AmplificationEvent.java`) |
| `AmplificationWorkflow` | `application/AmplificationWorkflow.java` |
| `AmplifierAgent` | `application/AmplifierAgent.java` (tasks in `application/AmplifierTasks.java`) |
| `ParseTools` | `application/ParseTools.java` |
| `DraftTools` | `application/DraftTools.java` |
| `PublishTools` | `application/PublishTools.java` |
| `BrandPolicyGuardrail` | `application/BrandPolicyGuardrail.java` |
| `BrandPolicyScorer` | `application/BrandPolicyScorer.java` |
| `AmplificationView` | `application/AmplificationView.java` |
| `MockModelProvider` (option-a only) | `application/MockModelProvider.java` |
| Bootstrap | `Bootstrap.java` |

## Concurrency notes

- **Per-step timeout**: `parseStep` 60 s, `draftStep` 60 s, `publishStep` 60 s, `evalStep` 5 s, `error` 5 s. Default step recovery `maxRetries(2).failoverTo(AmplificationWorkflow::error)`. The 60 s on each agent-calling step accommodates LLM latency including tool round-trips and potential brand-policy retry iterations (Lesson 4).
- **Idempotency**: each workflow uses `"amp-" + amplificationId` as the workflow id; restart of the same amplificationId is rejected by the workflow runtime. The agent instance id is `"agent-" + amplificationId` so each run has its own per-task conversation memory.
- **One agent per run**: `AmplifierAgent` runs three tasks per amplification — PARSE, DRAFT, PUBLISH — each with `capability(...).maxIterationsPerTask(4)`. The 4-iteration budget accommodates brand-policy rejections on DRAFT and publish-gate rejections on PUBLISH.
- **Dual-hook guardrail**: `BrandPolicyGuardrail` implements both hooks. The response hook fires on every DRAFT task response; the tool-call hook fires on every PUBLISH task tool call. The two hooks are registered independently on the agent's guardrail-configuration block.
- **Guardrail-driven retry**: when `BrandPolicyGuardrail` rejects a response or tool call, the rejection is returned as a structured error to the agent loop. Each rejection counts toward `maxIterationsPerTask`; if all 4 iterations fail validation, the workflow step fails over to `error` and the entity transitions to `FAILED`.
- **Eval is synchronous and deterministic**: `BrandPolicyScorer` runs in-process inside `evalStep`. No LLM call — the same run always scores the same.
- **Task-boundary handoff is the dependency contract**: `parseStep` writes `ArticleParsed` BEFORE returning; `draftStep` reads the recorded `ParsedArticle` from the entity to build its task's instruction context; `publishStep` reads both `ParsedArticle` and `DraftSet`. The agent itself is stateless across phases.
- **No saga / no compensation**: every step is either pure read, append-only event write, or a single-task agent call. A failed run stays at the last successful event; the UI shows the partial state.
